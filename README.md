# Boundary Testing Skill

Write and review tests that drive a component through its public boundary and
fake only the systems beyond it, using established libraries such as
`requests-mock`, SQLite, `freezegun` and `pyfakefs`. Complex algorithms also get
direct edge-case tests.

## Contents

- `SKILL.md`: the rule, the library table, the algorithm exception,
  anti-patterns and review prompts.
- `references/python.md`: worked examples for a Click CLI calling an HTTP
  service, a Flask API with a database and a chained HTTP service, and a
  directly tested algorithm.
- `agents/openai.yaml`: agent UI metadata.

## Usage

The skill is model-invoked. It applies when tests are written or changed and
when a PR or diff that changes tests is reviewed. The `tdd` skill composes it for
test-first work.

Install or link this repository as a skill directory for an agent that supports
`SKILL.md` based skills.
