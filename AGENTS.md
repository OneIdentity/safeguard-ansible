# AGENTS.md — safeguard-ansible

Ansible integrations for One Identity Safeguard for Privileged Passwords (SPP).
This repo ships an Ansible Galaxy collection plus an AWX / AAP managed
credential plugin. Both components depend on
[PySafeguard](https://github.com/OneIdentity/PySafeguard) (`pysafeguard>=8,<9`).

Requires Python ≥ 3.10 and Ansible ≥ 2.14.

## Project structure

```
safeguard-ansible/
├── .agents/skills/                         # On-demand reference skills
├── collection/
│   ├── oneidentity/safeguard/
│   │   ├── galaxy.yml                      # Collection metadata/version
│   │   ├── meta/runtime.yml                # Ansible compatibility contract
│   │   └── plugins/lookup/
│   │       ├── safeguardcredentials.py     # A2A lookup plugin
│   │       └── safeguardaccessrequest.py   # Access Request lookup plugin
│   └── tests/                              # pytest + playbook-backed integration tests
├── credential_type_plugin/
│   ├── pyproject.toml                      # Packaging metadata + AWX entry point
│   ├── safeguardcredentialtype/__init__.py # Managed credential implementation
│   └── tests/
├── pipeline-templates/                     # Shared Azure Pipelines templates
└── versionnumber.ps1                       # Shared version stamping
```

Key public entry points:

- collection namespace: `oneidentity.safeguardcollection`
- lookup plugins:
  `oneidentity.safeguardcollection.safeguardcredentials` and
  `oneidentity.safeguardcollection.safeguardaccessrequest`
- AWX/AAP credential plugin entry point:
  `awx.credential_plugins -> spp_plugin = safeguardcredentialtype:spp_plugin`
- there are currently no Ansible modules under `plugins/modules`

## Setup and build

Install shared dependencies before validation:

```bash
python -m pip install "pysafeguard>=8,<9" pytest build twine
python -m pip install ansible-core antsibull-changelog
```

Build commands:

```bash
cd collection/oneidentity/safeguard
ansible-galaxy collection build

cd ../../../credential_type_plugin
python -m build
```

## Linting

This repo has **implicit** linting only: there is no committed `ruff`, `flake8`,
`ansible-lint`, or `pylint` config. Use packaging and tests as the quality gate.

Expected validation:

- collection: `python -m pytest collection/tests/test_unit.py -q` and
  `cd collection/oneidentity/safeguard && ansible-galaxy collection build`
- credential plugin:
  `python -m pytest credential_type_plugin/tests/test_unit.py -q` and
  `cd credential_type_plugin && python -m build && twine check dist/*`
- keep YAML syntactically correct; there is no repo linter to catch bad
  indentation or schema drift for you

## Testing

See `.agents/skills/testing-guide/SKILL.md` for the full matrix. Quick commands:

```bash
python -m pytest collection/tests/test_unit.py -q
python -m pytest credential_type_plugin/tests/test_unit.py -q
python -m pytest collection/tests -q
python -m pytest credential_type_plugin/tests -q
```

Notes:

- integration tests require a live SPP appliance and skip when `SPP_HOST` is unset
- collection integration tests run real `ansible-playbook` subprocesses
- both suites provision and clean up live SPP objects through PySafeguard

## Code conventions

- Python ≥ 3.10 only; no Python 2 compatibility shims
- use explicit imports and 4-space indentation
- keep `DOCUMENTATION`, `EXAMPLES`, and `RETURN` accurate for lookup plugins
- use reStructuredText-style docstrings (`:arg name:`, `:returns:`)
- keep copyright headers on source files
- mark secret-bearing options with `no_log: True` or AWX secret fields
- use `PkceAuth`, not `PasswordAuth`
- reuse the shared `_resolve_verify()` pattern for TLS-related options

## CI/CD

For pipeline layout, release triggers, publishing, and version stamping, see `.agents/skills/build-and-release/SKILL.md`.

## Security

- never commit appliance hostnames, API keys, passwords, private keys, CA bundles,
  or exported AWX credential payloads
- treat `spp_api_key`, `spp_password`, cert private keys, retrieved passwords,
  and retrieved SSH keys as secrets at every layer
- prefer environment variables or secure AWX fields over hardcoded examples
- keep TLS verification enabled by default; disable it only for controlled test
  environments with self-signed certs
- when examples write SSH keys to disk, keep `mode: '0600'`, use `no_log: true`,
  and document cleanup on the control node

## Versioning

Source-of-truth versions live in:

- `collection/oneidentity/safeguard/galaxy.yml`
- `credential_type_plugin/pyproject.toml`

Release tags:

- `collection-v<X.Y.Z>` → Ansible Galaxy
- `credplugin-v<X.Y.Z>` → PyPI

Do not hand-edit build-time dev suffixes; `versionnumber.ps1` computes them.

## On-demand skills

Read these only when relevant:

| Skill | When to read | File |
|-------|-------------|------|
| Architecture | Repository layout, plugin boundaries, auth flow, collection vs AWX plugin structure | `.agents/skills/architecture/SKILL.md` |
| Testing Guide | Running tests, writing tests, or investigating failures | `.agents/skills/testing-guide/SKILL.md` |
| PySafeguard API | Plugin code changes or SDK usage questions | `.agents/skills/pysafeguard-api/SKILL.md` |
| Build & Release | CI/CD, versioning, or release work | `.agents/skills/build-and-release/SKILL.md` |

## Keeping this file current

Update this file when entry points, build/test commands, security expectations,
CI triggers, or the skill inventory change.
