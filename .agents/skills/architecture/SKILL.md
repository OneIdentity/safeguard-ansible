---
name: architecture
description: >-
  Use when understanding safeguard-ansible's repository layout, Ansible entry
  points, authentication flows, and the boundary between the collection,
  credential type plugin, and PySafeguard.
---
# safeguard-ansible Architecture
This repository contains two distributable integrations for One Identity
Safeguard for Privileged Passwords (SPP):
- an **Ansible Galaxy collection** in `collection/oneidentity/safeguard/`
- an **AWX / AAP managed credential plugin** in `credential_type_plugin/`
Both are thin adapters over `pysafeguard>=8,<9`; the repo does not implement its
own Safeguard REST client.
## Directory map
```
safeguard-ansible/
├── collection/
│   ├── oneidentity/safeguard/
│   │   ├── galaxy.yml
│   │   ├── meta/runtime.yml
│   │   └── plugins/lookup/
│   │       ├── safeguardcredentials.py
│   │       └── safeguardaccessrequest.py
│   └── tests/
│       ├── conftest.py
│       ├── helpers/
│       └── playbooks/
├── credential_type_plugin/
│   ├── pyproject.toml
│   ├── safeguardcredentialtype/__init__.py
│   └── tests/
├── pipeline-templates/
└── versionnumber.ps1
```
## 1. Entry point / public API surface
### Collection entry points
The collection publishes namespace `oneidentity.safeguardcollection` from
`collection/oneidentity/safeguard/galaxy.yml`.
Public Ansible-facing entry points:
- `lookup('oneidentity.safeguardcollection.safeguardcredentials', ...)`
- `lookup('oneidentity.safeguardcollection.safeguardaccessrequest', ...)`
Both files live under `plugins/lookup/` and implement Ansible's
`LookupBase.run(self, terms, variables=None, **kwargs)` contract. They are meant
for playbooks, variables, templates, and inventory lookups.
Collection contract details:
- `DOCUMENTATION`, `EXAMPLES`, and `RETURN` are part of the public surface.
- Returned values are plain strings, not reusable session objects.
- The repo currently ships **no** modules under `plugins/modules`; this is a
  lookup-only collection.
### AWX / AAP credential plugin entry point
The second component is the Python package `safeguardcredentialtype`, defined in
`credential_type_plugin/pyproject.toml`.
Its AWX discovery point is:
```toml
[project.entry-points."awx.credential_plugins"]
spp_plugin = "safeguardcredentialtype:spp_plugin"
```
`credential_type_plugin/safeguardcredentialtype/__init__.py` exports:
- `CredentialPlugin` namedtuple
- `spp_plugin`
- `_get_spp_credential()` backend
- `_resolve_verify()` TLS helper
AWX reads `spp_plugin.inputs` to render the UI and calls `backend` when it needs
a runtime credential value.
### Collection vs credential type packaging
| Component | Metadata file | Runtime host | Distribution |
|-----------|---------------|--------------|--------------|
| Collection | `collection/oneidentity/safeguard/galaxy.yml` | Ansible controller / execution node | Ansible Galaxy tarball |
| Credential plugin | `credential_type_plugin/pyproject.toml` | AWX / AAP controller Python env | PyPI wheel / sdist |
The two components are versioned and released independently.
## 2. Authentication strategy and flow
The repo supports two Safeguard auth models.
### A2A / client certificate flow
Used by:
- `safeguardcredentials`
- `safeguardcredentialtype`
PySafeguard API:
```python
from pysafeguard import A2AContext, A2AType
```
Shared inputs:
- `spp_appliance`
- client certificate path
- client private key path
- API key for the retrievable credential
- optional CA bundle / TLS toggle
- credential type: `password` or `privatekey`
Flow:
1. Resolve TLS with `_resolve_verify()`.
2. Normalize the requested type against `A2AType.PASSWORD` or
   `A2AType.PRIVATEKEY`.
3. Open an A2A session through PySafeguard.
4. Retrieve the secret.
5. Return `credential.value` from PySafeguard's `HiddenString`.
The collection plugin keeps one `A2AContext(...)` open per lookup call so it can
iterate over multiple API keys. The AWX plugin uses
`A2AContext.quick_retrieve_password()` / `quick_retrieve_private_key()` because
AWX resolves one credential at a time.
### Access Request / PKCE flow
Used only by `safeguardaccessrequest`.
PySafeguard API:
```python
from pysafeguard import SafeguardClient, PkceAuth, Service
```
Inputs:
- appliance host
- provider name (`spp_provider`)
- username and password
- asset name or IP
- optional account name
- credential type (`password` or `privatekey`)
- optional CA bundle / TLS toggle
- optional checkout timeout
Flow:
1. Resolve options through Ansible's option system.
2. Create `SafeguardClient(..., auth=PkceAuth(...), verify=verify)`.
3. `client.login()`.
4. Query `Me/RequestEntitlements`.
5. Create or reuse an `AccessRequests` item.
6. Poll checkout until success or timeout.
7. Return the password string or the `PrivateKey` field from the SSH-key payload.
The plugin intentionally does **not** check the credential back in after
retrieval, because immediate check-in would rotate it and invalidate the value
returned to Ansible.
### TLS behavior
All three plugin surfaces use the same `_resolve_verify()` shape:
```python
def _resolve_verify(tls_path, validate_certs=True):
    if isinstance(tls_path, str) and tls_path:
        return tls_path
    if validate_certs:
        return True
    return False
```
That maps to three modes: explicit CA bundle, system CA validation, or disabled
validation.
## 3. Connection lifecycle
### `safeguardcredentials`
Per lookup invocation:
1. `self.set_options(...)`
2. read `a2aconnection`
3. validate appliance / cert / key / type
4. `with A2AContext(appliance, cert, key, verify=verify) as a2a:`
5. loop over API-key terms
6. call `retrieve_password()` or `retrieve_private_key()`
7. append `.value` to the return list
8. exit the context manager
This is the only multi-key retrieval path in the repo.
### `safeguardaccessrequest`
Per lookup invocation:
1. `self.set_options(...)`
2. resolve direct kwargs, env vars, and ini values
3. validate terms and auth inputs
4. `with SafeguardClient(...) as client:`
5. `client.login()`
6. find entitlement
7. create or reuse access request
8. poll checkout endpoint
9. return a single credential in a single-element list
10. exit the client context
There is no connection cache across tasks or plays.
### `safeguardcredentialtype`
Per AWX credential resolution:
1. AWX discovers `spp_plugin`
2. AWX renders `inputs.fields`
3. AWX calls `_get_spp_credential(**kwargs)`
4. the backend lazily imports PySafeguard
5. it performs one A2A retrieval and returns a string
The lazy import is deliberate: AWX can still import the plugin module even when
`pysafeguard` is missing, and the error only appears when the backend runs.
## 4. Key abstractions and their relationships
### Main production abstractions
| File | Main abstraction | Role |
|------|------------------|------|
| `safeguardcredentials.py` | `LookupModule(LookupBase)` | A2A lookup by API key |
| `safeguardaccessrequest.py` | `LookupModule(LookupBase)` | Access Request checkout by asset/account |
| `safeguardcredentialtype/__init__.py` | `CredentialPlugin(name, inputs, backend)` | AWX managed credential type |
Shared production patterns:
- `_resolve_verify()` translates user TLS options into PySafeguard's `verify`
  parameter.
- all retrieval paths normalize to two credential types: `password` and
  `privatekey`.
- `Display().vvvv(...)` is used for verbose diagnostics, not state management.
### Credential-type mapping
| Surface | Type source | Dispatch |
|---------|-------------|----------|
| A2A lookup | `A2AType.PASSWORD` / `A2AType.PRIVATEKEY` | `retrieve_password()` / `retrieve_private_key()` |
| Access Request lookup | local `REQUEST_TYPES` + `CHECKOUT_ENDPOINTS` dicts | `CheckOutPassword` / `CheckOutSshKey` |
| AWX credential plugin | UI field choice | `quick_retrieve_password()` / `quick_retrieve_private_key()` |
If a new credential form is added, all three surfaces must change together.
### Test-side support layer
The collection test suite supplies most of the end-to-end support model:
- `collection/tests/helpers/certificates.py` generates client certs, SSH keys,
  and CA bundles
- `collection/tests/helpers/provisioning.py` creates and deletes live SPP
  objects
- `collection/tests/conftest.py` provisions session fixtures and runs real
  `ansible-playbook` subprocesses
- `credential_type_plugin/tests/conftest.py` reuses collection test helpers by
  adding `collection/tests` to `sys.path`
That relationship matters: the repo keeps production code thin and encodes much
of the integration behavior in fixtures and playbooks.
## 5. Event system
There is no event bus, callback plugin, or asynchronous worker system here.
Architecture is synchronous request/response:
- Ansible evaluates a lookup
- the plugin opens a PySafeguard connection
- the credential is returned
- the context manager exits
The closest thing to repeated background behavior is the fixed-interval checkout
retry loop in `safeguardaccessrequest`, but it is still synchronous polling.
## 6. Extension points
### Add a new collection lookup plugin
1. Add a file under `collection/oneidentity/safeguard/plugins/lookup/`.
2. Implement `LookupModule(LookupBase)`.
3. Add accurate `DOCUMENTATION`, `EXAMPLES`, and `RETURN` blocks.
4. Reuse existing option names and `_resolve_verify()` behavior where possible.
5. Add unit tests in `collection/tests/` and playbook-backed integration tests
   in `collection/tests/playbooks/` when live behavior changes.
6. Update collection docs in `collection/oneidentity/safeguard/README.md` and
   repo guidance if the public surface changed.
### Extend the AWX credential plugin
1. Update `credential_type_plugin/safeguardcredentialtype/__init__.py`.
2. Modify `spp_plugin.inputs.fields` / `inputs.required`.
3. Keep sensitive fields marked `secret: True`.
4. Preserve lazy PySafeguard import unless AWX discovery requirements change.
5. Update `credential_type_plugin/tests/test_unit.py` and
   `test_integration.py`.
### Add a new credential type
Update all coupled locations together:
- A2A type normalization in `safeguardcredentials.py`
- `REQUEST_TYPES` and `CHECKOUT_ENDPOINTS` in `safeguardaccessrequest.py`
- AWX field choices and backend dispatch in `safeguardcredentialtype`
- tests, playbooks, and docs for both components
### Add another distributable component
Follow the existing repository pattern:
- keep packaging metadata at the component root
- give the component its own tests and Azure pipeline
- reuse `pipeline-templates/` and `versionnumber.ps1`
- update `AGENTS.md` and this skill when the public surface changes
