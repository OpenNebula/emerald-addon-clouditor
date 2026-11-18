# EMERALD Evidence Add-on for OpenNebula

## Description

This add-on collects security evidence from an [OpenNebula](https://opennebula.io)
cloud and sends it to [Confirmate](https://github.com/confirmate/confirmate), the
compliance engine from Fraunhofer AISEC. Confirmate checks the evidence against
EUCS and CIS metrics, and the results appear in the EMERALD UI.

```text
┌──────────────────────────────────────────────────────────────────────┐
│  OpenNebula front-end                                                │
│                                                                      │
│  ┌───────────────────┐    ┌───────────────────────────────────────┐  │
│  │ VMs, NICs, images │    │ this add-on                           │  │
│  │ networks, users,  │    │                                       │  │
│  │ datastores        │    │  hooks/   one script per resource     │  │
│  │                   │──> │  lib/     HookCommon (shared steps)   │  │
│  │ a change fires an │    │           OntologyMapper              │  │
│  │ OpenNebula hook   │    │           (redacts secrets)           │  │
│  │                   │    │           ConfirmateClient            │  │
│  │                   │    │           TokenManager (OAuth2)       │  │
│  └───────────────────┘    └───────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │  POST /v1/evidence_store/evidence
                                   │  HTTPS + OAuth2 token (Keycloak in EMERALD)
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Confirmate (Fraunhofer AISEC), EMERALD assessment services          │
│                                                                      │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐  │
│  │  Evidence    │     │  Assessment      │     │  Orchestrator    │  │
│  │  Store       │ ──> │  (Rego metrics)  │ ──> │  (results)       │  │
│  └──────────────┘     └──────────────────┘     └──────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │  assessment results
                                   ▼
                       ┌────────────────────────┐
                       │  EMERALD UI            │
                       │  compliance dashboard  │
                       └────────────────────────┘
```


The add-on runs on the OpenNebula Front-end. OpenNebula hooks run it every time
a VM, NIC, image, network, user or datastore changes. The add-on maps the
resource to the Confirmate ontology, removes passwords, tokens and 2FA secrets
from the raw XML, and posts the evidence with an OAuth2 token.

## Development

To contribute bug patches or new features, you can use the GitHub Pull Request
model. It is assumed that code and documentation are contributed under the
Apache License 2.0.

Run the tests before you open a pull request:

```bash
ruby tests/test_ontology_mapper.rb
ruby tests/test_token_manager.rb
ruby tests/test_confirmate_client.rb
ruby tests/smoke.rb   # end-to-end POST, skipped if no Confirmate is reachable
```

## Authors

* Pablo del Arco, [OpenNebula Systems](https://opennebula.io)

## Compatibility

* Tested with OpenNebula 7.4 on the EMERALD Pilot 4 Front-end.

## Requirements

### OpenNebula Front-end

* Ruby from the OpenNebula packages. The add-on uses only the Ruby standard library.
* Network access to the Confirmate Evidence Store and to the OAuth2 token endpoint.

### Confirmate

* The URL of a Confirmate Evidence Store.
* An OAuth2 client for it. In an EMERALD deployment, Keycloak issues the token.
* The ID of a Target of Evaluation (ToE) from the EMERALD UI.

To test without a real deployment, start a local Confirmate first with
[docs/local-confirmate.md](docs/local-confirmate.md).

## Features

| OpenNebula event | Evidence sent |
|---|---|
| VM enters RUNNING or POWEROFF, or is terminated | VirtualMachine, one NetworkInterface per NIC, one BlockStorage per disk |
| NIC attached or detached | One NetworkInterface per NIC, and the VirtualMachine again |
| Image becomes READY | VMImage |
| Virtual network created | VirtualNetwork |
| User created or updated | Identity |
| Datastore created, updated or permissions changed | ObjectStorage |

The evidence covers disk encryption, public IPs, SSH and RDP exposure in
security groups, backups, log retention, MFA for administrators and public
access to images and datastores.
[docs/mapping.md](docs/mapping.md) lists every field and the controls it
supports.

## Limitations

* Hooks react to changes. A resource that exists before the installation is sent
  only when it changes state for the first time.
* The add-on doesn't keep a queue. If the Evidence Store doesn't answer after
  3 attempts, the evidence of that event is lost and the error is in the log.

## Installation

```bash
git clone https://github.com/OpenNebula/emerald-addon-clouditor
cd emerald-addon-clouditor
sudo ./install.sh
```

The installer copies the scripts to `/var/lib/one/remotes/hooks/`, the
configuration to `/etc/one/confirmate-evidence.conf`, and registers 12 hooks.

`sudo ./uninstall.sh` removes the hooks and scripts and asks before it deletes
the configuration.

## Configuration

Edit `/etc/one/confirmate-evidence.conf`. In an EMERALD deployment, Keycloak
issues the token, so ask the platform operator for a service client.

```yaml
confirmate:
  endpoint: "https://<evidence-store-host>"
  auth:
    enabled: true
    token_url: "https://<keycloak-host>/realms/<realm>/protocol/openid-connect/token"
    client_id: "<service-client-id>"
    client_secret: "<service-client-secret>"

evidence:
  target_of_evaluation_id: "<ToE UUID from the EMERALD UI>"
```

```bash
sudo chown oneadmin:oneadmin /etc/one/confirmate-evidence.conf
sudo chmod 640 /etc/one/confirmate-evidence.conf
```

| Parameter | Description | Default |
|---|---|---|
| `confirmate.endpoint` | Evidence Store URL | `http://localhost:8080` |
| `confirmate.auth.enabled` | Send a bearer token | `false` |
| `confirmate.auth.token_url` | OAuth2 token endpoint | `http://localhost:8080/v1/auth/token` |
| `confirmate.auth.client_id`, `client_secret` | OAuth2 client | `confirmate`, `confirmate` |
| `confirmate.auth.static_token` | Fixed bearer token instead of OAuth2 (testing) | empty |
| `confirmate.tls.ca_file` | Extra CA bundle, added to the system roots | empty |
| `evidence.target_of_evaluation_id` | ToE UUID (required) | all-zeros placeholder |
| `evidence.additional_target_of_evaluation_ids` | Other ToEs that receive a copy of every evidence | empty |
| `evidence.tool_id` | Tool name sent with every evidence | `opennebula-addon-confirmate-evidence` |
| `evidence.default_region` | Region label | `eu-south-1` |
| `evidence.logrotate_path` | Logrotate file used to report log retention | `/etc/logrotate.d/opennebula` |
| `logging.level`, `logging.file` | Add-on log | `info`, `/var/log/one/confirmate-evidence.log` |

The add-on doesn't start if `target_of_evaluation_id` is missing or is not a
UUID.

## Usage

Check that the 12 hooks are registered and follow the log:

```bash
sudo -u oneadmin onehook list | grep -c hook-confirmate   # expect 12
sudo tail -f /var/log/one/confirmate-evidence.log
```

Create or change any resource, for example a test Virtual Network. The log
shows `Evidence ... stored successfully (HTTP 200)` when the evidence reaches
Confirmate.

### Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `HTTP 401` in the add-on log | Wrong `client_id` or `client_secret`, or `auth.enabled` doesn't match the server |
| `HTTP 400` with a validation message | `target_of_evaluation_id` is not a real ToE. Copy it from the EMERALD UI |
| Fewer than 12 hooks registered | Register the templates by hand, as shown in [docs/troubleshooting.md](docs/troubleshooting.md) |

[docs/troubleshooting.md](docs/troubleshooting.md) covers the other cases.

## Acknowledgments

<img src="https://upload.wikimedia.org/wikipedia/commons/b/b7/Flag_of_Europe.svg" alt="EU Flag" width="50" align="left" style="margin-right: 10px;">

This project has received funding from the European Union's Horizon
Europe research and innovation programme under grant agreement No.
**101120688** ([EMERALD](https://emerald-he.eu)).

**Partners:** [Fraunhofer AISEC](https://www.aisec.fraunhofer.de/) (Confirmate) · [OpenNebula Systems](https://opennebula.io) (OpenNebula and this add-on) · [CaixaBank](https://www.caixabank.com) (Pilot 4 validation)

## License

Apache License 2.0. See [LICENSE](LICENSE).
