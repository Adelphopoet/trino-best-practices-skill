---
title: Treat VARBINARY and VARCHAR as Different Domains
impact: HIGH
impactDescription: "Avoids broken comparisons, lossy casts, and fake textual handling of binary values."
tags:
  - trino
  - types
  - varbinary
description: "Agents MUST not compare `varbinary` and `varchar` as if they were interchangeable; use explicit encoding or decoding functions."
alwaysApply: true
---

## Treat VARBINARY and VARCHAR as Different Domains

Trino treats binary bytes and text as different domains. Reviews that collapse them into "strings" are wrong.

### Why it matters

Hashes, encoded payloads, and binary identifiers frequently live in `varbinary`. Comparing them to text without deliberate conversion produces invalid logic, brittle casts, or unreadable plans. Trino exposes dedicated functions such as `to_hex()`, `from_hex()`, `to_utf8()`, and `from_utf8()` for this reason.

### How to detect

- `sha256(...)`, `md5(...)`, or `from_hex(...)` outputs are compared directly to `varchar`.
- The query casts binary values to `varchar` just to compare them.
- Text data is hashed without an explicit `to_utf8(...)` conversion.

### Correct

```sql
SELECT id
FROM hive.stage.events
WHERE to_hex(sha256(to_utf8(payload))) = lower(expected_hash_hex);
```

This keeps the binary hash as binary and converts it to hex only for text comparison.

### Incorrect

```sql
SELECT id
FROM hive.stage.events
WHERE sha256(payload) = expected_hash_hex;
```

This compares `varbinary` output to text and assumes Trino will invent the right encoding.

### Fix

- Use `to_utf8()` when text must become bytes.
- Use `to_hex()` / `from_hex()` or base64 helpers when binary must meet text.
- Preserve `varbinary` until the exact boundary where text encoding is required.

### Reference

- https://trino.io/docs/current/functions/string.html
- https://trino.io/docs/current/functions/binary.html

