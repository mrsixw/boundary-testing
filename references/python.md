# Python worked examples

Both examples drive the component at its boundary and fake only the systems
beyond it. Names, hosts and data are placeholders.

## Click CLI calling an HTTP service

The command reads a token from the user's home directory, calls an HTTP API and
prints a table. The test runs the real command and fakes the API, the
filesystem and the clock.

```python
import json
from pathlib import Path

import requests_mock
from click.testing import CliRunner
from freezegun import freeze_time
from pyfakefs.fake_filesystem_unittest import Patcher

from mytool.cli import cli


def test_pool_list_sends_token_and_prints_empty_result():
    with requests_mock.Mocker() as http, freeze_time("2026-01-02 09:00:00"), Patcher() as fs:
        fs.fs.create_file(
            Path.home() / ".mytool" / "token",
            contents=json.dumps({"access_token": "placeholder", "expires_at": "2026-01-02T10:00:00"}),
        )
        http.get("https://api.example.test/pools", json=[])

        result = CliRunner().invoke(cli, ["pool", "list"], env={"MYTOOL_NO_UPDATE_CHECK": "1"})

    assert result.exit_code == 0, result.output
    assert "No pools found" in result.output
    (request,) = http.request_history
    assert request.qs == {"datacenter": ["all"]}
    assert request.headers["Authorization"] == "Bearer placeholder"
```

What it covers: argument parsing, config and token loading, the HTTP client,
URL and header building, response handling and output. Nothing inside the
package is mocked.

An error path follows the same shape:

```python
def test_pool_list_reports_api_failure():
    with requests_mock.Mocker() as http, Patcher() as fs:
        fs.fs.create_file(Path.home() / ".mytool" / "token", contents='{"access_token": "placeholder"}')
        http.get("https://api.example.test/pools", status_code=503)

        result = CliRunner().invoke(cli, ["pool", "list"])

    assert result.exit_code == 1
    assert "service unavailable" in result.output.lower()
```

## Flask API with a database and a chained HTTP service

The route validates input, writes to the database and notifies another service.
The test uses the app factory with an SQLite database, sends requests through
the test client and fakes the downstream service.

```python
import pytest
import requests_mock

from myapi import create_app
from myapi.models import Datacenter, db


@pytest.fixture
def client(tmp_path):
    app = create_app({"TESTING": True, "DATABASE_URL": f"sqlite:///{tmp_path / 'test.db'}"})
    # The app context stays open so tests can query the database directly.
    with app.app_context():
        db.create_all()
        db.session.add(Datacenter(slug="dc1", cluster_size=3))
        db.session.commit()
        with app.test_client() as client:
            yield client


@pytest.fixture
def downstream():
    with requests_mock.Mocker() as http:
        http.post("https://notify.example.test/events", status_code=202)
        yield http


def test_update_cluster_size_persists_and_notifies(client, downstream):
    response = client.put(
        "/datacenters/dc1",
        json={"cluster_size": 10},
        headers={"Authorization": "Bearer placeholder"},
    )

    assert response.status_code == 200
    assert response.get_json() == {"slug": "dc1", "cluster_size": 10}
    assert Datacenter.query.filter_by(slug="dc1").one().cluster_size == 10
    (event,) = downstream.request_history
    assert event.json() == {"type": "datacenter.updated", "slug": "dc1"}


def test_update_rejects_negative_size(client, downstream):
    response = client.put("/datacenters/dc1", json={"cluster_size": -1}, headers={"Authorization": "Bearer placeholder"})

    assert response.status_code == 400
    assert downstream.call_count == 0
```

Fake auth at its edge too. Configure the app with a test signing key and send a
token signed with it, or point the token validator at a `requests-mock` stub of
the identity provider. Do not replace the auth decorator or set the endpoint's
auth to a `Mock()`.

## A complex algorithm, tested directly

A redaction filter that masks secrets in streamed output has many cases, so it
gets direct tests in addition to a CLI test that shows it is wired in.

```python
import pytest

from mytool.output import redact


@pytest.mark.parametrize(
    ("line", "expected"),
    [
        ("token=abc123", "token=****"),
        ("Authorization: Bearer abc.def.ghi", "Authorization: Bearer ****"),
        ("password='p@ss word'", "password='****'"),
        ("no secrets here", "no secrets here"),
        ("", ""),
    ],
)
def test_redact(line, expected):
    assert redact(line) == expected
```
