# How It Works

The add-on runs on the OpenNebula Front-end. It has no service or daemon.
OpenNebula runs one hook script for each resource change, and the script ends
after it sends the evidence.

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

## Code Modules

| **File** | **What it does** |
|---|---|
| `hooks/confirmate_*_evidence.rb` | One script per resource type. OpenNebula runs it when the resource changes |
| `lib/hook_common.rb` | Reads the configuration, decodes the resource XML from the hook and sets up the log |
| `lib/ontology_mapper.rb` | Converts the XML to a Confirmate ontology resource and removes passwords, tokens and 2FA secrets |
| `lib/token_manager.rb` | Gets an OAuth2 token with the client credentials flow |
| `lib/confirmate_client.rb` | Sends `POST /v1/evidence_store/evidence`, gets a new token on HTTP 401 and tries 3 times on other temporary errors |
| `templates/hook-*.tmpl` | The 12 hook definitions that `install.sh` registers with `onehook create` |

Refer to [Resource mapping](mapping.md) for every field the add-on sends and the
controls it supports.
