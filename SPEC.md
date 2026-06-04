# Artsoft API Call-Log Specification (v1)

**Status:** reference implementation live on the Mega Loja side · proposed for Artsoft to mirror
**Author:** Descomplicador (Mega Loja Borja Reis integration)
**Last updated:** 2026-06-04

---

## Purpose (pt-PT)

Registo independente e imutável de **todas** as chamadas feitas à API de
Integração Artsoft, para que — se algum dia houver dúvida sobre se a nossa
integração causou lentidão ou indisponibilidade no sistema Artsoft — exista
prova datada e verificável de **exactamente** o que foi chamado, **quando**, e
**que estado o servidor Artsoft devolveu**.

A ideia central: o registo mais credível é o que pode ser **cruzado pelas duas
partes**. Por isso este formato é simétrico. Nós registamos o que enviámos; se a
Artsoft registar o que recebeu no mesmo formato, qualquer pessoa pode comparar
os dois lado a lado, linha a linha. Nenhuma das partes precisa de confiar
cegamente na outra.

## Purpose (EN)

An independent, append-only record of **every** call made to the Artsoft
Integration API, so that any dispute about whether an integration caused Artsoft
slowness or downtime can be settled with timestamped, verifiable evidence of
exactly what was called, when, and what status the Artsoft server returned.

The format is **symmetric** by design: each side logs its half (we log what we
sent; Artsoft would log what it received) in the *same shape*, so the two logs
can be compared row-for-row by any third party.

---

## 1. Canonical format — JSON Lines

The **canonical, vendor-neutral format is JSON Lines** (one JSON object per
line, UTF-8, newline-separated — [jsonlines.org](https://jsonlines.org)). Any
stack in any language can emit it. The Postgres table in §3 is a *reference
implementation* of the same fields, not the canonical form.

One record = one HTTP attempt (see `attempt_no` — retries each get their own
record, so retry-storms are visible).

### Example record

```json
{"consumer":"artsoft","source":"gateway@artsoft-prod","endpoint":"/documents","http_method":"GET","started_at":"2026-06-04T06:00:01Z","ended_at":"2026-06-04T06:00:11.800Z","duration_ms":10800,"http_status":200,"ok":true,"rows_returned":38642,"response_bytes":13219840,"attempt_no":1,"error_class":null,"error_text":null}
```

### Field dictionary

| Field            | Type            | Req | Meaning |
|------------------|-----------------|-----|---------|
| `consumer`       | string          | yes | Who made the call. **This is the symmetry key.** Our rows = `"mega-loja"`; Artsoft's rows = `"artsoft"` (or per-client id if they log multiple consumers). |
| `source`         | string          | yes | The specific program + host, e.g. `"artsoft_sync.py@megahub"` / `"n8n:Artsoft Sales Sync"`. Free text; identifies the emitter. |
| `endpoint`       | string          | yes | API path **only** — `"/documents"`, `"/products"`, `"/partners/customers"`. **Never the full URL** (no host, no query string carrying secrets). |
| `http_method`    | string          | yes | `"GET"` etc. |
| `started_at`     | string (ISO8601, UTC, `Z`) | yes | When the request was sent. |
| `ended_at`       | string (ISO8601, UTC, `Z`) | no  | When the response/error came back. Null only if the process died mid-call. |
| `duration_ms`    | integer         | no  | `ended_at − started_at` in milliseconds. |
| `http_status`    | integer \| null | no  | The HTTP status the **server returned**. `null` on a network-level failure (no response: timeout, connection refused, DNS). |
| `ok`             | boolean         | yes | `true` iff a usable 2xx response was received. This is the single field a dispute reads first. |
| `rows_returned`  | integer \| null | no  | If the body is a JSON array, its length. Null otherwise. |
| `response_bytes` | integer \| null | no  | Raw response body size in bytes. Shows payload weight. |
| `attempt_no`     | integer         | yes | 1-based. Each retry is a **separate record** with the next number. |
| `error_class`    | string \| null  | no  | Machine-friendly error bucket: `"http_500"`, `"http_502"`, `"http_404"`, `"timeout"`, `"conn_refused"`, `"dns"`, `"parse"`, `"other"`. Null when `ok`. |
| `error_text`     | string \| null  | no  | Short human error detail, ≤2000 chars. **MUST NOT contain credentials** (see §4). |

### What the fields prove in a dispute

- **"When did you call us?"** → `started_at` / `ended_at`, every call.
- **"Were you calling during the outage / work hours?"** → query the time window; **zero rows = proof of silence**.
- **"Did your calls storm us with retries?"** → `attempt_no > 1` rows make every retry visible.
- **"Whose fault was the error?"** → `http_status` + `error_class`. A `500`/`502`/`timeout` is the **server** failing, recorded from the caller's side.

---

## 2. Emission & publication conventions

1. **Append-only.** Records are only ever *added*. Never edited or deleted. A
   correction is a new record, not a mutation of an old one.
2. **Emit one record per attempt**, written the moment the call returns (success
   or failure). Best-effort: logging must never break the actual integration —
   if the log write fails, the call still proceeds.
3. **Off-site within seconds.** Our records land in Postgres, which is
   stream-replicated to a second datacentre (Phoenix) within ~140 ms — so each
   record is independently off-site almost immediately. (Artsoft's mirror should
   have an equivalent off-site property; the point is that neither side can
   silently back-date.)
4. **Publication for independent review (optional, both sides).** Each side may
   publish a daily export to a neutral, externally-timestamped location — e.g. a
   git repository whose host-received commit time is a third-party timestamp.
   Once published, a day's file is immutable. Anyone can then `git log` both
   sides and compare. Our publisher writes a per-day `YYYY-MM-DD.jsonl` plus a
   rolling `daily_summary.csv`.

> **Honest scope of the guarantee.** The strong, non-repudiable property is the
> **near-instant off-site replication** plus (once publishing is on)
> **third-party commit timestamps** — not any database trick. An in-database
> "no UPDATE/DELETE" rule is a useful guard but a superuser can override it; we
> do not claim it as cryptographic proof. Timestamps (`started_at`) are the
> emitter's local clock; the externally-anchored times are the replication
> `recorded_at` and the git commit time.

---

## 3. Reference implementation — Postgres DDL

This is *one* way to store the §1 records; it is the shape we run. Porting to
any other store is fine as long as the JSON-Lines export in §1 is preserved.

```sql
CREATE TABLE artsoft_api_call (
    id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    consumer       text        NOT NULL DEFAULT 'mega-loja',
    source         text        NOT NULL,
    endpoint       text        NOT NULL,
    http_method    text        NOT NULL DEFAULT 'GET',
    started_at     timestamptz NOT NULL,
    ended_at       timestamptz,
    duration_ms    integer,
    http_status    smallint,
    ok             boolean     NOT NULL,
    rows_returned  integer,
    response_bytes bigint,
    attempt_no     smallint    NOT NULL DEFAULT 1,
    error_class    text,
    error_text     text,
    recorded_at    timestamptz NOT NULL DEFAULT now()  -- when the row hit the DB
);
```

The daily summary view (`artsoft_api_call_daily`) and the append-only guards are
in the migration `mega-obras/database/027_artsoft_api_call.sql`.

---

## 4. Security — never log credentials

The bearer token MUST NOT appear in any record. Enforced at three layers:

1. **`endpoint` is the path only** — never the full URL, never the
   `Authorization` header.
2. **`error_text` is sanitised** — any `Bearer <token>` substring is replaced
   with `Bearer [REDACTED]` before writing, and truncated to 2000 chars.
3. **A standing assertion** rejects publication if any row's `error_text`
   matches `Bearer` / a JWT prefix (`eyJ`).

---

## 5. Proposal to Artsoft (Softer)

We already run the producer half of this spec. The ask is simple and symmetric:
**Artsoft emits the same JSON-Lines records for the calls it receives from each
integration consumer**, and (ideally) publishes a daily export the same way.

Benefit to both sides: no future argument about API load can become a matter of
opinion. Either log, on its own, is checkable; together they are conclusive. If
a consumer ever genuinely overloads the API, Artsoft's log proves it
immediately — and if it doesn't, the consumer's log clears them just as fast.

Implementation effort on the Artsoft side is small: log the §1 fields per
inbound call at the API gateway, append to a JSON-Lines file or table, publish
daily. Any language, any store — the format is the only contract.
