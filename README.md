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
