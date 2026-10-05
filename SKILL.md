---
name: boundary-testing
description: "Write and review tests that drive a component through its public boundary (CLI, HTTP API, module interface) and replace only the systems beyond it with established libraries such as requests-mock, SQLite, freezegun and pyfakefs, with direct edge-case tests reserved for complex algorithms. Use whenever tests are written or changed, whether or not the work is test-first, and whenever a PR or diff that adds or changes tests is reviewed. Not for the red-green-refactor loop itself (use tdd, which composes this skill) or for diagnosing an unknown failure (use diagnose-bug)."
---

# Boundary testing

Test what a component does, through the entry point its callers use, and fake
only what lies beyond it. Such tests survive refactoring and cover the wiring
between the pieces, which is where thin per-function tests leave gaps.

## Choose the mode

- **Write mode:** tests are being written or changed. Apply the rules below and
  report which boundary each new test drives and which systems it replaces.
- **Review mode:** a PR or diff adds or changes tests. Stay read-only. Apply
  [Review prompts](#review-prompts) and return findings in the calling review's
  format. Each finding cites a changed line and the consequence. Applicability
  alone is not a finding.

Repository test conventions take precedence. When the repository already has a
boundary harness, use it rather than adding a second one.

## The rule

1. **Find the boundary.** It is the entry point a caller or operator uses: the
   CLI command, the HTTP route, the public function of a library or the inputs
   and outputs of an infrastructure module.
2. **Drive the boundary for real.** Invoke the command with real arguments and
   environment, or send a real request through the framework's test client, so
   that parsing, routing, auth, validation and serialisation all run.
3. **Replace only what lies beyond it.** HTTP services, databases, the clock,
   the filesystem, cloud APIs and message queues. Never mock the component's own
   client, models, config loader or private methods.
4. **Assert on what is observable.** Exit code, output, response status and
   body, stored state, files written and the requests sent to external systems.
5. **Use established libraries for the fakes.** They are more faithful than
   hand-written stubs and `mock.patch` chains. Prefer the ones the repository
   already uses. When none covers the boundary, add a well-known one as a test
   dependency and say so in the change.

| Boundary or external system | Python library |
|---|---|
| Click CLI | `click.testing.CliRunner` |
| Flask app | `app.test_client()` from an app factory |
| FastAPI or Starlette app | `fastapi.testclient.TestClient` |
| HTTP calls through `requests` | `requests-mock` or `responses` |
| HTTP calls through `httpx` | `respx` |
| SQL database | SQLite (in memory or a temporary file) behind the real ORM |
| The clock | `freezegun` or `time-machine` |
| The filesystem | `pyfakefs`, or `tmp_path` for real but isolated files |
| AWS | `moto` |
| Environment variables | `monkeypatch.setenv` or `CliRunner.invoke(env=...)` |

[references/python.md](references/python.md) has worked examples for a Click CLI
calling an HTTP service and for a Flask API with a database and a chained HTTP
service. Other languages follow the same rule with their own equivalents, such
as `httptest` in Go or `supertest` and `msw` in Node.

## The exception: complex algorithms

Also test a complex algorithm directly at its own interface, with its edge
cases. Examples are parsers, schedulers and placement logic, money and date
arithmetic, version comparison, diffing, redaction filters, rule rendering,
state machines and retry or backoff calculations. A boundary test cannot reach
every case of such logic cheaply. Keep a boundary test as well, so that the
wiring stays covered.

A thin wrapper around a framework or a library call is not a complex algorithm.
Cover it through the boundary.

## Anti-patterns

- Mocking the component's own client, models or config so that the code under
  test never builds a URL, a header or a query.
- Patching private methods of the class under test and asserting they were
  called.
- Calling a handler or resource method inside a request context instead of
  sending a request through the test client, which skips routing, auth and
  serialisation.
- Replacing auth with a bare `Mock()`. Fake the identity provider or token at
  the edge instead.
- Starting patches at module level without stopping them, which leaks into
  every other test.
- A CLI helper that swallows `SystemExit`, so exit codes are never asserted.
- Exact matches on long whitespace-padded tables when a few key lines or the
  parsed values would prove the behaviour.
- Expected values recomputed with the code under test.
- Real hostnames, tokens or account identifiers in fixtures. Use placeholders.

## Review prompts

Investigation prompts, not automatic findings. Raise one only with evidence from
the diff.

- Does each new test drive the public boundary, or does it call internals that a
  boundary test would cover?
- Is anything mocked that belongs to the component itself?
- Is an external system faked by hand where an established library would be
  more faithful?
- Do the assertions check observable results, including the requests sent?
- Is a complex algorithm tested directly with its edge cases, and is its wiring
  covered at the boundary too?
- Do any of the anti-patterns above appear in the changed tests?

When a test is at the wrong level, suggest the boundary test that would replace
it, not only the problem.
