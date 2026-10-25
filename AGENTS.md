# app-mail-relay agent rules

This repository follows `/Users/junkawasaki/github/AGENTS.md`.

## Native gate status

- Retired for acceptance: JVM, Clojure CLI, nbb, kbb/Sci, shadow-cljs and
  GraalVM gates. Historical files may remain temporarily as migration input,
  but they are not a fallback and must not be used as passing evidence.
- Required gate: Amu selfhost plus Kotoba native, with a negative control that
  proves forbidden launchers were not invoked.
- Current blocker (2026-10-25): the qualified public Amu image is a
  loaderless `x86_64-linux-static` ELF and browser/Worker project linking is
  not yet qualified. Source and tests in this repository therefore remain
  unqualified until that native route exists.
- A source review, parser result, legacy test result or generated Worker file
  is not native qualification, deployment, or live behavior.

## Architecture boundary

- SMTP/Resend and P2P are transports. Pure policy belongs in `mail-relay.*`;
  network, storage and cryptography are host capabilities.
- Human/Sybil scores are optional, expiring policy inputs. They never prove a
  person, grant authority, or by themselves deliver/reject mail.
- P2P envelopes contain ciphertext references only. Subject, body, headers and
  attachments must remain inside recipient-encrypted objects.
- Browser applications are static artifacts and must use the canonical shared
  client runtime and fresh operation-specific consent. This relay is an
  external protocol service, not app-origin logic.
