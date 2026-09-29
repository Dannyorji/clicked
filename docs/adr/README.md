# Architecture Decision Records

Short records of decisions this repo has already made and re-litigated in
review. New decisions use [0000-template.md](./0000-template.md)
(context / decision / alternatives rejected / consequences / status).

Cross-referenced from [`IMPLEMENTATION_DOCS.md`](../../IMPLEMENTATION_DOCS.md).

## Index

| ADR | Title | Status |
| --- | --- | --- |
| [0001](./0001-single-devices-table.md) | One canonical `devices` table | accepted |
| [0002](./0002-message-ordering-by-createdat-id.md) | `(createdAt, id)` ordering; no `sequenceNumber` | accepted |
| [0003](./0003-ciphertext-only-message-storage.md) | Ciphertext-only message storage | accepted |
| [0004](./0004-per-device-envelopes.md) | Per-device envelopes over a shared ciphertext | accepted |
| [0005](./0005-sealed-box-to-signal-migration.md) | Phase-1 sealed box → Phase-2 Signal | accepted |
| [0006](./0006-mls-for-groups.md) | MLS for groups | accepted |

None of these records is superseded. If a later change replaces one, set
that file to `superseded` and point at the new ADR; leave the old file in
this list.

## Adding a record

1. Copy `0000-template.md` to `NNNN-short-title.md` (next free number).
2. Fill every section, including **what was rejected and why**.
3. Add a row here with status `accepted` or `superseded`.
4. Link the ADR from the code or concept doc that implements it.
