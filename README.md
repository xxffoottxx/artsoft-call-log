# Artsoft API Call Log — Mega Loja Borja Reis

A public, append-only, externally-timestamped record of **every call the Mega
Loja integration makes to the Artsoft Integration API**
(`megaloja-api.newsofter.pt`).

## Why this exists

So that any question about whether our integration affects the Artsoft system can
be settled with **evidence, not opinion**. Each day's file lists every API call
we made — when, which endpoint, the HTTP status Artsoft returned, the duration,
and the response size. The git commit timestamp (and GitHub's server-side push
record) anchor each day's data to a date we cannot back-date.

The records contain **no customer data, no monetary values, and no
credentials** — only operational metadata about our own outbound calls.

## Format

Each `YYYY-MM-DD.jsonl` file is [JSON Lines](https://jsonlines.org) — one JSON
object per API call. `daily_summary.csv` is a rolling per-day, per-endpoint
summary (call count, errors, latency, rows). The full field dictionary is in
[`SPEC.md`](SPEC.md).

Example record:

```json
{"consumer":"mega-loja","source":"artsoft_sync.py@mega-loja-sync","endpoint":"/documents","http_method":"GET","started_at":"2026-06-04T06:00:21Z","ended_at":"2026-06-04T06:00:32Z","duration_ms":10800,"http_status":200,"ok":true,"rows_returned":38642,"response_bytes":13219840,"attempt_no":1,"error_class":null,"error_text":null}
```

## The store's own Artsoft server (counts, since 2026-10-17)

Besides the cloud API, the integration talks to the Artsoft server inside the
shop: the XMLServer on the store PC (port 2600 = Materiais, 2601 = Mobiliário)
and the product service on port 3000 that the per-article stock screens read.
Since **17 October 2026, 20:50 (Azores)** every one of those calls — reads and
writes — is recorded on our side, one record per call, in a separate
append-only table.

Here we publish the **daily counts**: `caixa_resumo.csv` (date, port, reads,
writes, errors, calls made during the shop's opening hours that day) and a
section in each day's `-resumo.md`. The calls themselves are not published:
their queries carry customer and document numbers. Writes to the store server
are made with the store management's authorisation.

## Correction — 17 October 2026

Until 17 October 2026 every `-resumo.md` ended with the same fixed paragraph:
our system reads Artsoft once a day, at night, only reads, and does not contact
it while the shop is open. **That paragraph was wrong**, and had been for weeks:

- Small daytime reads of the cloud API run on a schedule — recent documents
  and products four times a day (since September), and the current-account
  poll once an hour during opening hours (since 31 August). These calls were
  always in the `.jsonl` files and counted on the same page; only the closing
  paragraph contradicted them.
- The per-article stock reads behind the shelf-label and stock screens (two
  calls per article opened, since 24 August 2026) **were not logged at all**
  until 17 October 2026. They are now, as `n8n-etiquetas@mega-loja-sync`.
- Occasional writes to the store's own server (with the management's
  authorisation) were not covered by this record. They are now counted.

From 17 October 2026 every sentence of the daily summary is computed from that
day's records; store hours come from the shop's own calendar. Summaries
published before that date are left exactly as they were.

## An open invitation to Artsoft / Newsofter

This is a **symmetric** format. The `consumer` field identifies who made the
call (`mega-loja` here). If Artsoft publishes the same records for the calls it
*receives* — per consumer — the two logs can be compared line-for-line by anyone.
No party has to be trusted blindly: either log is checkable on its own, and
together they are conclusive. If a consumer ever genuinely overloads the API,
Artsoft's log proves it instantly — and if it doesn't, the consumer's log clears
them just as fast.

Implementation effort to mirror this is small: log the [`SPEC.md`](SPEC.md)
fields per inbound call at the API gateway, append to a JSON-Lines file, commit
daily. Any language, any store — the format is the only contract.
