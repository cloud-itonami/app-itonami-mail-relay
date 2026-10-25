# Itonami Mail MVP

## Roles

| role | responsibility | must not do |
|---|---|---|
| static app origin | distribute verified HTML/JS/Wasm/Worker artifacts | process mail, hold user state, proxy providers |
| client | hold recipient keys, decrypt, build threads/search/JMAP projection, ask consent | mint or persist approval |
| SMTP gateway | SMTP ingress/egress, SPF/DKIM/DMARC/ARC/MTA-STS evidence, bounce/queue | treat transport auth as user identity |
| Itonami relay | route signed P2P ciphertext envelopes and enforce admission | receive plaintext subject/body/headers |
| Kotoba policy | deterministic persona, budget and admission decisions | perform ambient IO or hold provider secrets |

## Delivery paths

SMTP input is authenticated as a domain transport, mapped to a pairwise
persona, admitted, encrypted for the recipient, and stored. Unknown senders go
to first-contact quarantine; a high Human Passport score only changes review
priority.

Itonami-native input carries `itonami.mail.p2p-envelope.v1`. The signature
covers sender, recipient, ciphertext CID, timestamps and nonce. The relay
checks expiry and replay before admission. The ciphertext object uses the
existing X25519/AES-GCM seal path.

SMTP output remains the interoperability bridge. The caller chooses a persona,
never an arbitrary From address. Existing `ito` entries budget first contacts.

## Compatibility boundary

The relay cannot honestly provide server-side JMAP search or threads because
message headers are encrypted. `mail-relay.jmap` is therefore a client-side
projection after decryption. SMTP remains the Internet transport boundary;
JMAP is the local mailbox/API model. IMAP compatibility may be provided by a
local adapter over that decrypted model, not by making the relay read mail.

## Admission order

1. Reject explicit blocks, oversize payloads and rate-limit violations.
2. For P2P, reject invalid signatures, expired envelopes and replay.
3. Existing relationships deliver unless SMTP authentication has regressed;
   regressions quarantine.
4. Unknown senders quarantine by default.
5. Invite, refundable postage or PoW plus an explicit receiver policy may
   admit an unknown sender. A humanity score alone never does.

## Qualification state

Source contracts and deterministic tests exist, but the repository has no
qualified native browser/Worker test route as of 2026-10-25. The former
`npm test` route invoked nbb/JVM transitively and is retired as acceptance
evidence. No deployment or DNS/MX cutover is implied by this MVP.
