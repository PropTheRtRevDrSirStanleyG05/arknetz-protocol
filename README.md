# arknetz-protocol

Protocol specification, JSON schemas, and Zstd dictionary for ArkNetzWebWeaver
by STANBEYOND (MIT). Versioned with semver; current version **0.1.0**
(2026-11-20). Forks pin to a protocol version.

- `PROTOCOL.md` — the full spec (schemas, shard rules, encryption formats,
  backends, Wayback, accounts, deep links). Fields the thread does not define
  are marked PROPOSED; claims no live provider verified are marked UNVERIFIED.
- `schemas/*.json` — JSON Schemas derived from PROTOCOL.md §3 (v0.1.0).
  The spec text is authoritative; the schemas are a machine-readable mirror.
- `zstd-dictionary-v2.bin` — Zstd dictionary **v2 "v2-trained"**,
  32,768 bytes, ID `0x41524B02`, SHA-256
  `a1786956c8f9b919c393ba27347213c8db48a283615b23bfe03582f92544c96d`.
  Genuinely trained with `zstd --train` on a 3,200-file corpus of the
  project's own schemas and sample documents (see `dict-training/` in
  the client repo). Every .json.zst the client writes uses v2, so every
  frame carries the v2 ID. `zstd-dictionary-v2.b64` is the same bytes
  base64 (for transports that cannot carry binary). Verify with:
  `sha256sum zstd-dictionary-v2.bin`.
  The dictionary is reproducible from source in the client repo
  (`src/lib/zstd.ts`: `getZstdDictionaryV2()`).
- `zstd-dictionary-v1.bin` — the superseded v1 dictionary
  ("v1-corpus-derived", 26,300 bytes, ID `0x41524B01`, SHA-256
  `9e7346fce69bd694f438c83acced640aba7f804f5901ad4dd932d0d4816b19ae`), kept so frames compressed before the v2 switch
  still decode. New files MUST use v2.
- Protocol-root manifest: https://github.com/PropTheRtRevDrSirStanleyG05/arknetz-protocol-root
