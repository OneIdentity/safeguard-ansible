# Contributing to safeguard-ansible

Thanks for your interest in improving safeguard-ansible, the Ansible Galaxy
collection and credential type plugin for the One Identity Safeguard Web
API.

## Reporting issues

- **Bugs and feature requests:** open a GitHub Issue.
- **Security vulnerabilities:** do **not** open a public issue — follow
  [SECURITY.md](SECURITY.md).

## Prerequisites

- [Python 3.10](https://www.python.org/downloads/) or later and
  [Ansible](https://docs.ansible.com/) (`ansible-core >= 2.14`).
- Development dependencies:

      python -m pip install "pysafeguard>=8,<9" pytest build twine ansible-core

- (Optional) a Safeguard for Privileged Passwords appliance for the
  integration tests.

## Building

    cd collection/oneidentity/safeguard && ansible-galaxy collection build
    cd credential_type_plugin && python -m build

## Testing

Hermetic unit tests require no appliance:

    python -m pytest collection/tests/test_unit.py -q
    python -m pytest credential_type_plugin/tests/test_unit.py -q

Integration tests **skip automatically** when `SPP_HOST` is unset. To run
them against a lab appliance:

    SPP_HOST=<appliance> python -m pytest collection/tests -q

## Coding conventions

Python 3.10+ style; mark secret values `no_log: True`. See
[AGENTS.md](AGENTS.md) for the full conventions.

## Submitting changes

1. Fork the repository and create a feature branch.
2. Keep commits focused with clear messages.
3. Ensure the unit tests pass and the collection builds
   (`ansible-galaxy collection build`, `twine check dist/*`).
4. Open a pull request describing the behavior you changed and the tests
   that prove it.