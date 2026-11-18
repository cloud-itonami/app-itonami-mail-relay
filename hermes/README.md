# hermes/ — the resident bots that act for this repository

This directory is the **source of truth** for the Hermes profiles listed below
(ADR-2609241200). The host's `~/.hermes/profiles/<profile>` is materialized
from `hermes/profiles/<profile>/` and checked against it:

```
kbb --backend sci scripts/hermes-profile-repo.cljk materialize <profile>   # repo -> host
kbb --backend sci scripts/hermes-profile-repo.cljk check <profile>         # 0 agree / 1 drift / 2 could not compare
kbb --backend sci scripts/hermes-profile-repo.cljk export <profile>        # host -> repo, then commit
```

(run from the com-junkawasaki/root superproject; registry
`manifest/hermes-profile-repos.edn`.)

Each profile directory holds SOUL.md, profile.yaml, config.yaml (host-local
blocks removed), cron/jobs.json (definitions only), scripts/ and the skills the
profile owns. **Never here:** `.env` or any secret value, workspace/ledgers,
sessions, memories, logs, caches, run state.

## Profiles

| profile | description |
|---|---|
| `mail-relay` |  |
| `tobari` |  |

## Inference rail

These are Hermes agents on Awai Otehon. They never name murakumo or mishima —
not in `model`, `providers`, `fallback_providers`, `auxiliary`, `moa` or a cron
`model`/`provider` pin (itonami-agent AGENTS.md "Two CLIs"; AWAI-3303). Their
default is the Awai node's own OpenAI-compatible `/v1` on loopback: provider
`awai-node`, `http://127.0.0.1:11434/v1`, model `local` (network-awai-apex ADR
0026 D4; owner decision on AWAI-320). `api.murakumo.cloud` is not a default.
Other providers already set (openrouter-free, kotoba) stay as they are.
Inference on Murakumo / mishima belongs to itonami agent profiles.

The node must serve a model under the name `local` on that port. The change is
to these definitions only. A host profile is updated when it is materialized
from here, which needs its own operator step.
