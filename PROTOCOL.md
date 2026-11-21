# ArkNetzWebWeaver Protocol — `arknetz-protocol` v0.1.0

> **Status: DRAFT for review — v0.1.0, 2026-11-20.**
> This document is the clean protocol specification for ArkNetzWebWeaver by
> STANBEYOND (MIT licence). It is written from the project's thread brief
> (`THREAD-SPEC.md`, 46 turns, Nov 15–18 2026), the repo topology
> (`REPO-TOPOLOGY.md`), and the shipped scaffold
> (`ArkNetzWebWeaver` v0.2.0). Anything this document says that the thread
> does not define is labelled **PROPOSED**; anything the thread claims but
> nobody has verified against a live provider is labelled **UNVERIFIED**.
> A fork or third-party client that implements this document must be able to
> natively render sites, stream media, play games, and exchange mail on the
> main network — no proprietary lock-in, no hard-coded single-network
> assumption.

## 0. Versioning and stability promises

- **Protocol version:** `0.1.0` (`PROTOCOL_VERSION` in `src/lib/config.ts`).
  The version is carried in the protocol-root manifest
  (`protocolVersion`) and in each site document (`version`, optional).
- **Semver for the protocol repo.** Breaking changes (a field removed or
  re-typed, a shard rule changed, an encryption format replaced) bump the
  major version. Additive changes (a new optional field, a new block type,
  a new backend prefix) bump the minor version and must be readable by
  older clients: **unknown fields are ignored, unknown block types render
  as a visible placeholder, never as raw HTML.**
- **Forks pin to a protocol version.** The client and the central data do
  not need to move when the protocol moves; a fork may run an older
  protocol version against its own registry.
- **No silent re-sharding.** Expanding the registry from 16 shards to 256
  (§5) changes only the shard-key width; no domain moves unless a
  re-shard is announced as a major version.
- **Zstd dictionary versions are explicit** (§4): a file compressed with
  dictionary vN names vN, and a client that does not know vN must fail
  loudly, never silently serve the wrong bytes.
- **Stability promise to forks:** the `.json.zst` schemas in §3, the
  `arknetz://` deep-link shape, and the sandboxed `["app", …]` postMessage
  API (§7) are public and versioned. A fork that implements v0.1.0 reads
  every v0.1.0 site on the main network.

## 1. Principles

1. **Zero server, $0/month.** All state lives in Git repos, the Internet
   Archive, and the other pluggable backends (§8). No central process
   ever has to be running for the network to work.
2. **Reads are free.** Routine reads use public CDNs
   (`raw.githubusercontent.com` and friends), never metered APIs.
3. **Text/JSON in Git, media in the Archive.** GitHub repos hold
   compressed text/JSON only. All media (images, audio, video,
   thumbnails) goes through the Internet Archive (thread turn 30).
4. **Exact-string domains.** Domains are exact full strings
   (`computers.edu`, `edu.orion.stanbeyond`, `alex.zine`) — never whole
   TLDs. No ICANN/DNS cost; the registry is its own root.
5. **Anonymous reading.** Browsing needs no sign-in. Writes (claim,
   edit/publish, teams, mail) need a signed-in account.
6. **Batched writes.** Autosave is local (IndexedDB); Git commits happen
   on Publish only — never a commit per keystroke.
7. **No Fediverse/ActivityPub, no RSS/feed.xml.** Both were considered in
   the thread and rejected by Stanley. Do not add them.
8. **Per-user keys — no master/shared upload credential, ever.** Every
   write/upload on every backend is done with the acting user's OWN
   account credential: their own GitHub/GitLab/Codeberg/Hugging Face
   token, their own IA S3 keys, their own Wikimedia/Filebase keys.
   Stanley's correction, 2026-11-20: a real IA upload is done with
   INDIVIDUAL user IA keys — everyone will NOT use his keys; that is
   what the sign-in flow for all backends is for. The same rule applies
   to `gh:` / `gl:` / `cb:` / `ia:` / `hf:` / `wm:` / `ipfs:`. There is
   no master upload key, no shared fallback token, no demo credential
   in the app. Any write attempted without the acting user's own
   credential fails loudly ("sign in / link your own account first")
   before any network call. The Root Master identity
   (`PropTheRtRevDrSirStanleyG05`) is an AUTHORIZATION check (who may
   moderate, §10.2) — it is not a credential and never stands in for
   one. (Enforced in code, §6.4, §8; regression-tested.)

## 2. Transports and file format

- **Every protocol file is Zstd-compressed JSON:** `<name>.json.zst`.
  Readers try the `.json.zst` first and fall back to plain `<name>.json`
  on 404. A `.zst` that downloads but fails to decompress is a corrupt
  file and throws — it must NOT silently fall back to the plain file.
- **Zstd engine:** `@bokuweb/zstd-wasm` (MIT), real Zstandard, loaded
  from the vendored copy in the app bundle — never from a CDN.
  **UNVERIFIED (performance):** the thread's compression claims (a 2 KB
  post batching to ~0.35 KB with a dictionary, ~14.2M posts in 5 GB) are
  Google AI's arithmetic, not measured numbers.
- **JSON rules:** UTF-8, objects only at the top level unless the schema
  says otherwise, Unix seconds for timestamps unless the schema says
  ISO 8601. Unknown fields are ignored.
- **Reads:** `raw.githubusercontent.com` only for GitHub (never
  `api.github.com` — 60/hr unauthenticated). ETag / `If-None-Match`
  caching in IndexedDB; a 304 means 0 bytes. User-Agent:
  `ArkNetzWebWeaver/1.0`.
- **Writes:** batched on Publish only. GitHub writes use the contents
  API (`PUT /repos/{owner}/{repo}/contents/{path}`) with the latest
  commit SHA to guard against overwrites.
- **Rate-limit posture (thread turn 31):** the client keeps a persistent
  IndexedDB media cache (1 fetch per asset, not per reload), a polite
  queue (2–3 concurrent, ~100–150 ms stagger — the scaffold uses 2
  concurrent, ~120 ms), and exponential backoff on 429/503 (max 5
  attempts, local copy kept until the upload succeeds).

## 3. `.json.zst` schemas

Field types: `string`, `number` (Unix seconds unless noted), `boolean`,
`string[]`, `object`. Fields marked **(required)** must be present;
all others are optional and must be tolerated when absent. Schemas that
the thread defines only by example or by name are marked **PROPOSED**.

### 3.1 Registry shard — `registry.json.zst`

One file per shard repo (`arknetz-registry-00` … `arknetz-registry-0f`).
Either shape is accepted. Shard repos also publish a plain
`registry.json` mirror alongside the canonical `registry.json.zst`
(`register.yml` commits both on every claim); **availability checks
read the canonical `.zst` first** (decompressed via the §4 dictionary
path) and consult the plain mirror only on `.zst` 404 — the plain
mirror is a secondary artifact a direct Root Master write can omit,
and a missing mirror must never read as "available" for a taken
domain. A corrupt `.zst` throws and never falls back.

```json
{ "computers.edu": { "domain": "computers.edu", "backend": "gh:alice/arknetz-site-computers-edu", "mirrors": ["cb:alice/site-mirror"], "shard": "a", "owner": "alice", "updated": 1794787200 } }
```

or `{ "entries": [ … ] }`.

| Field | Type | Notes |
|---|---|---|
| `domain` **(required)** | string | Exact full string, lowercase, ≥2 labels. |
| `backend` **(required)** | string | Backend URI (§8), e.g. `gh:alice/arknetz-site-computers-edu`. |
| `mirrors` | string[] | Mirror backend URIs, tried in order after `backend`. |
| `shard` **(required)** | string | `"0"`–`"f"` (16 shards); `"00"`–`"ff"` when expanded to 256 (§5). Clients compute it from SHA256(domain) and reject an entry whose shard disagrees. |
| `owner` **(required)** | string | GitHub login that owns the site repo. |
| `updated` | number | Unix seconds of the last claim/update. |
| `delegations` | object | Subdomain → backend URI, when delegations live in the registry rather than the site repo. Delegated subdomains normally live in the site's `delegations.json.zst` (§3.3). |

### 3.2 Site document — `site.json.zst`

Lives in the per-domain site repo (`arknetz-site-<domain-slug>`).
Rendered by the single shared block renderer.

| Field | Type | Notes |
|---|---|---|
| `domain` **(required)** | string | Exact full string. |
| `title` **(required)** | string | Site title. |
| `theme` | object | `{ "layout": "docs" \| "blog" \| "magazine" \| "portfolio", "accent": "#2563eb", "font": "sans" }`. |
| `nav` | `{label, href}[]` | Global navigation; adding a page updates it everywhere. |
| `header` / `footer` | BlockNode[] | Reusable components, persistent across pages. |
| `blocks` **(required)** | BlockNode[] | Typed semantic blocks (§3.2.1). No raw HTML/JS. |
| `version` | string | Protocol version this document was written with. |
| `updated` | number | Unix seconds. |

#### 3.2.1 BlockNode

| Field | Type | Notes |
|---|---|---|
| `type` **(required)** | string | Whitelist: `hero, nav, h1, h2, h3, p, cols, cards, series-grid, img, video, audio, code, quote, link, docs-sidebar, gallery, app, list, table`. Unknown types render as a visible placeholder. |
| `text` | string | Plain text; always kept in sync with `runs`. |
| `runs` | InlineRun[] | Inline rich-text runs (bold/italic/underline/strike/code/links) on text blocks. |
| `src` | string | Media hotlink — `https://archive.org/download/<item>/<file>` only. |
| `alt` | string | Alt text; empty + `decorative: true` means "mark as decorative". |
| `href` | string | Link target. Allow-listed to `https://`, `archive.org`, `raw.githubusercontent.com`, `arknetz:` deep links. |
| `title` / `subtitle` | string | Hero / card text. |
| `language` | string | Code block language. |
| `items` | BlockNode[] | Children (cards, gallery, series-grid). |
| `columns` | BlockNode[][] | Columns block. |
| `listStyle` | `("ul"\|"ol")` | "list": bulleted or numbered. Absent = "ul". |
| `listItems` | ListItem[] | "list": the items, nested via `ListItem.children` (a list is at least 2 levels deep). Items carry typed inline runs, never raw HTML. |
| `rows` | TableCell[][] | "table": rows of cells; every row has the same length. Cells carry typed inline runs, never raw HTML. |
| `headerRow` | boolean | "table": the first row renders as `<th scope="col">`. Absent = true. |
| `appSrc` / `appSrcdoc` | string | `["app", …]` bundle URL or inline srcdoc. Compressed cap **~150 KB** (thread turn 21). Rendered only as a sandboxed iframe (§7). |
| `visibleOn` | `("desktop"\|"tablet"\|"mobile")[]` | Per-block responsive visibility; absent = visible everywhere. |
| `focalPoint` | `{x, y}` | Image focal point, 0–100 on each axis. |
| `decorative` | boolean | Screen readers skip the image. |
| `caption` | string | Figcaption (defaults to `alt`). |
| `poster` | string | Video poster hotlink. |
| `id` | string | Unique block id, assigned by the editor; used by revision diff. |

### 3.3 Delegations — `delegations.json.zst`

Sub-delegation by domain owners (thread: `ai.computers.edu` →
`gh:sarah/ai-blog`).

```json
{ "ai.computers.edu": "gh:sarah/ai-blog", "blog.computers.edu": "gh:alice/blog" }
```

Shape: object mapping exact subdomain strings to backend URIs (§8).
Resolution order: registry → parent domain → this file → fetch the
target's `site.json.zst`. **PROPOSED:** the thread does not fix whether
delegation values may carry mirrors; clients should accept either a
string URI or `{ "backend": "…", "mirrors": ["…"] }`.

### 3.4 Moderation — `moderation.json.zst` and `sub_moderation.json.zst`

Two tiers (thread turn 12). `moderation.json.zst` lives in the central
`arknetz-moderation` repo (Root Master only — enforced by GitHub repo
write permissions, never by a secret in the app).
`sub_moderation.json.zst` lives in each site repo (registrars).
Resolution order before rendering: Root Master → Parent Registrar →
Post Author. The Root Master may also publish a content override /
redaction patch replacing offending text/media at render time.

**PROPOSED** (the thread fixes the authority model, not the exact
fields):

```json
{
  "patches": [
    { "target": "computers.edu", "action": "ban_domain", "reason": "…", "by": "PropTheRtRevDrSirStanleyG05", "at": 1794787200 },
    { "target": "computers.edu/post-42", "action": "redact", "replaceText": "[removed]", "by": "…", "at": 1794787200 }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `patches` | object[] | One entry per moderation action. |
| `patches[].target` | string | Domain, subdomain, or `domain/post-id`. |
| `patches[].action` | string | `ban_domain`, `redact`, `remove_post`. **PROPOSED** values; the thread names "edit/delete/redact" as capabilities. |
| `patches[].by` / `patches[].at` | string / number | Moderating account and Unix time, for audit. |

Early thread drafts used signed tombstones
(`{"action": "ban_domain", "target": "alice.ink"}` with Ed25519 keys);
that design was superseded by GitHub OAuth + repo-permission control.
Do not ship the Worker relay or a hard-coded master public key.

### 3.5 Contacts — `contacts.json.zst`

Global contacts / public-key directory, sharded like the registry
(16 shards by address hash, §5). **Measured with X25519 keys
(2026-11-20): ~35.3 MiB per 1M users in the stored `.json.zst` form**
(37.0 bytes/entry measured on 2,000 real entries) — the thread's
~40 MB holds for compressed storage, not for uncompressed JSON
(157.4 MiB/1M; raw 32-byte keys alone are 32 MB/1M). Used for protocol
mail addressing.

**PROPOSED** (the thread fixes the size model and the sharding, not the
exact fields):

```json
{ "alice@computers.edu": { "address": "alice@computers.edu", "displayName": "Alice", "publicKey": "<base64 JWK/SPKI>", "site": "computers.edu", "updated": 1794787200 } }
```

| Field | Type | Notes |
|---|---|---|
| `address` **(required)** | string | Protocol mail address, `user@custom.domain`. |
| `displayName` | string | Shown in the address book. |
| `publicKey` **(required)** | string | Web Crypto public key (base64) for the hybrid mail envelope (§6). |
| `site` | string | The contact's home domain. |
| `updated` | number | Unix seconds. |

### 3.6 Directory — `directory-<category>.json.zst`

Federated local search (thread turn 37). On publish, the client pushes
a ~150-byte metadata card to the central index shard for its category
(`directory-tech`, `directory-cinema`, `directory-games`, …). Directory
shards live in **per-category repos** `arknetz-directory-<category>`
(REPO-TOPOLOGY.md §8) as `directory-<category>.json.zst` — never in
protocol-root, which holds no directory file. The app
downloads shards (a few hundred KB for tens of thousands of sites) into
IndexedDB and searches locally with MiniSearch (~7 KB).

**The category list is manifest-discoverable (fix round 3,
2026-11-20).** The protocol-root manifest publishes
`directoryCategories`: an array of lowercase category slugs, e.g.
`["tech", "cinema", "games", "music"]`. The client loads exactly the
categories the manifest names — a new category appears in search as
soon as the Root Master publishes it in the manifest, with no client
update. When the manifest has no `directoryCategories` field (old
manifest), the client falls back to the documented default
`["tech", "cinema", "games"]`; the default exists only as that
fallback. A present-but-invalid field (not a non-empty array of valid
slugs) is rejected at manifest-parse time, so a half-published
category list fails loudly instead of silently shrinking search.
Category slugs match `/^[a-z0-9]+(-[a-z0-9]+)*$/` — the same rule
that keeps shard repo names safe to build from the slug.

```json
{ "cards": [ { "domain": "computers.edu", "title": "Computers", "tags": ["edu", "tech"], "category": "tech", "updated": 1794787200 } ] }
```

| Field | Type | Notes |
|---|---|---|
| `cards` | object[] | One per site in the category. |
| `cards[].domain` **(required)** | string | Exact domain. |
| `cards[].title` | string | Site title. |
| `cards[].tags` | string[] | Search tags. |
| `cards[].category` | string | Shard category. |
| `cards[].updated` | number | Unix seconds; drives Trending / Newly Updated feeds. |

### 3.7 Feed — `feed-index.json.zst` / monthly `feed-YYYY-MM.json.zst`

Global feed pointers, time-sharded monthly (append-only; a new month
never rewrites last month's file). ~30 MB per 1M posts (thread number,
**UNVERIFIED**). **PROPOSED** (the thread fixes the sharding and the
pointer size, not the exact fields):

```json
{ "month": "2026-11", "pointers": [ { "domain": "computers.edu", "post": "post-42", "at": 1794787200 } ] }
```

| Field | Type | Notes |
|---|---|---|
| `month` | string | `YYYY-MM` shard key. |
| `pointers` | object[] | One ~30-byte pointer per post: domain + post id + time. |
| `pointers[].domain` / `pointers[].post` | string | Where the post lives; the reader fetches the site's posts file. |
| `pointers[].at` | number | Unix seconds. |

There is intentionally no `feed.xml` / RSS: the static-feed compromise
was rejected by Stanley.

### 3.8 Posts — `posts.json.zst` / `posts-YYYY-MM.json.zst`

Per-domain site repo. The thread's canonical example block payload is:

```json
{ "t": "My First Post", "d": 1794787200, "b": [["p", "Hello decentralized world."], ["img", "ia:stanbeyond-media/photo.webp", "Sunset"]] }
```

**PROPOSED** as a file envelope (the thread defines the post shape,
not the file wrapper):

```json
{ "domain": "computers.edu", "posts": [ { "id": "post-42", "t": "…", "d": 1794787200, "b": [["p", "…"]] } ] }
```

| Field | Type | Notes |
|---|---|---|
| `domain` | string | Owning domain. |
| `posts` | object[] | Posts, newest first. |
| `posts[].id` | string | Stable post id, referenced by feed pointers and comments. |
| `posts[].t` | string | Post title. |
| `posts[].d` | number | Unix seconds. |
| `posts[].b` | array | Semantic blocks in the thread's compact tuple form (`["p", "text"]`, `["img", "ia:…", "alt"]`). The site document (§3.2) uses the expanded BlockNode form; both serialise to the same renderer. |

### 3.9 Team — `team.json.zst`

Per-domain site repo. Granular roles (thread turn 39, final list of 12
flags). The shipped scaffold's `TeamRole` is `{ name, flags,
allowedPaths? }`.

```json
{
  "owner": "alice",
  "roles": {
    "managing_editor": ["pages.write", "pages.publish", "media.upload", "mail.read", "mail.send"],
    "writer": ["pages.draft", "media.upload"],
    "subdomain_registrar": ["subdomains.manage", "moderation.sub"]
  },
  "members": {
    "bobdev": { "role": "managing_editor" },
    "charlie": { "role": "writer", "allowed_paths": ["blog/*"] },
    "dana": { "role": "subdomain_registrar" }
  }
}
```

Flags (all 12): `pages.draft`, `pages.publish`, `allowed_paths`,
`apps.embed`, `media.upload`, `subdomains.manage`, `moderation.sub`,
`comments.moderate`, `mail.read`, `mail.send`, `stats.view`,
`team.manage`. Enforcement is two-layer: client UI gating + an optional
GitHub Action validating that commits to `site.json.zst` /
`team.json.zst` came from an authorised username. Invites use the
GitHub Collaborators API. Drafts go to `drafts.json.zst` or inside
`allowed_paths` only. **PROPOSED:** the `members[].allowed_paths`
override shape above follows the thread's example; the scaffold stores
`allowedPaths` on the role, not the member — clients must accept both.

### 3.10 Stats — `stats.json.zst`

Privacy-first anonymous view counts (thread turn 39). ~400 bytes.
Built nightly by `stats.yml` from batched `VIEW_PING` Issues, which are
then deleted. Zero IP logging; a Strict Offline/No-Telemetry mode opts
the visitor out entirely.

**PROPOSED:**

```json
{ "domain": "computers.edu", "days": { "2026-11-15": { "index": 12, "about": 3 } }, "updated": 1794787200 }
```

| Field | Type | Notes |
|---|---|---|
| `domain` | string | Owning domain. |
| `days` | object | `YYYY-MM-DD` → `{ page: views }`. Only pages the creator enabled a Public View Counter for. |
| `updated` | number | Unix seconds of the last rollup. |

Comment/reaction counts come from GitHub Discussions/Issues and media
play counts from `archive.org/metadata/<id>` — those are not in this file.

### 3.11 Mailing lists — `mail_lists.json.zst`

Protocol mail aliases (thread turn 18), e.g. `staff@computers.edu`.
**PROPOSED:**

```json
{ "staff@computers.edu": ["alice@computers.edu", "bob@computers.edu"] }
```

Shape: object mapping a list address to member addresses. A list
expands at send time; the hybrid envelope (§6) wraps the key once per
recipient. Large lists (e.g. 50) store the payload once in the sender
outbox and drop a pointer + wrapped key per inbox — **implemented
(2026-11-20): the pointer JSON is ~316 bytes measured on a 55-recipient
send, not the thread's ~120** (full-length ids), and the payload is
stored exactly once (12,694 bytes once vs 698,170 duplicated).

### 3.12 Comments archive — `comments-archive.json.zst`

**PROPOSED.** Threaded comments live in GitHub Discussions/Issues in
the site owner's repo (Giscus/Utterances model); guests read free,
signed-in users comment. This file is the portable archive/export of
that thread state, so a site keeps its comments if it migrates off
GitHub Issues:

```json
{ "domain": "computers.edu", "threads": [ { "post": "post-42", "comments": [ { "id": "c1", "author": "bob", "at": 1794787200, "text": "…", "replies": [] } ] } ] }
```

| Field | Type | Notes |
|---|---|---|
| `threads` | object[] | One per post. |
| `threads[].comments` | object[] | Nested `replies` carry the thread. `author` is a provider login, never an email. |

### 3.13 Vault index — `vault-index.enc.zst`

ArkDrive (thread turn 46): the encrypted manifest of a private vault
repo. The file on disk is **Zstd-compressed, then AES-256-GCM
encrypted** — the name is literal: compress first, encrypt second.
A repo inspector sees only this file and random-named encrypted chunks
(`blob_8f9a2c.enc`); zero filenames, zero previews, zero metadata leaks.

Decrypted + decompressed, **PROPOSED**:

```json
{
  "vault": "arknetz-vault",
  "files": [
    { "name": "report.pdf", "path": "docs/report.pdf", "size": 123456, "chunks": ["blob_8f9a2c.enc"], "sha256": "…" }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `vault` | string | Vault label (decrypted only). |
| `files` | object[] | File entries; `name`/`path` exist ONLY inside the encrypted manifest. |
| `files[].chunks` | string[] | 10 MB encrypted chunk names; files >10 MB split, SHA-256 hashed, duplicates not re-uploaded. Chunks stay <25 MB so Git never chokes. |
| `files[].sha256` | string | Whole-file hash, for integrity on download/reassembly. |

Encryption: §6.3. Backends allowed for ArkDrive: GitHub / GitLab /
Codeberg private repos, and Hugging Face ONLY for open-science/AI
datasets and model weights. **Internet Archive and Wikimedia are
EXCLUDED from ArkDrive** — uploading opaque encrypted personal blobs
there violates their ToS.

### 3.14 Collab state — `peers/<login>.json` on `collab/<domain-slug>`

**PROPOSED (implemented 2026-11-21, mock-verified only).** Real-time
multi-user collaboration for the editor. One session per site: a
`collab/<domain-slug>` branch in the per-domain site repo, one state
file per peer at `peers/<login>.json`, read/written through the GitHub
Contents API with the acting user's OWN token (`public_repo` scope).
Reads use `api.github.com`, never `raw.githubusercontent.com` (the raw
CDN caches branch content). Each peer writes only its own file, so
peers never contend for the same SHA. The branch never touches `main`
or `site.json.zst` until someone Publishes.

```json
{
  "v": 1, "session": "computers-edu", "login": "alice",
  "colour": "#2563eb", "heartbeatAt": 1794787200000,
  "cursor": { "blockId": "b2", "start": 4, "end": 4 },
  "blocks": [ { "type": "h1", "text": "Hello", "id": "b1" } ],
  "versions": { "b1": { "updatedAt": 1794787200000, "author": "alice" } },
  "placement": { "b1": { "updatedAt": 1794787200000, "author": "alice", "blockId": "b1" } },
  "deleted": {}
}
```

| Field | Type | Notes |
|---|---|---|
| `v` **(required)** | number | State-file version (1). Readers reject unknown versions loudly. |
| `session` **(required)** | string | Domain slug; one session per site. |
| `login` / `colour` | string | Peer identity and deterministic display colour. |
| `heartbeatAt` **(required)** | number | Unix ms of the write. A peer gone >30 s drops from presence. |
| `cursor` | object \| null | `{ blockId, start, end }` — live caret/selection. |
| `blocks` **(required)** | BlockNode[] | The peer's full block list (snapshots, not an op log). |
| `versions` **(required)** | object | blockId → `{ updatedAt, author }`. Per-block LWW on `(updatedAt, author)`. |
| `placement` | object | blockId → placement key for deterministic ordering of concurrent inserts at the same index. |
| `deleted` | object | blockId → `{ updatedAt, author }` tombstones. Absence from `blocks` alone is never a delete. |

Honest limits (binding): poll-based, ~5 s latency, never sub-second;
no character-level OT — same-block edits resolve whole-block, newer
wins, loser recorded in a visible conflict note, never silently
dropped; offline peers' blocks stay, marked stale. This is NOT Google
Docs parity. Live GitHub round-trips have not run (mock-verified only).

## 4. Zstd dictionary versioning

- The thread specifies a **~32 KB pre-trained dictionary** for the
  `.json.zst` files, so small posts compress far below their raw size.
- **Dictionary version field.** Every dictionary is versioned. The
  dictionary version that compressed a file is recorded in the
  protocol-root manifest (`zstdDictionaryVersion`) and, where the
  transport allows it, alongside the file. **PROPOSED:** the manifest
  field is `zstdDictionary: { "version": 2, "url": "…", "sha256": "…" }`.
- **How a client upgrades.** A client fetches the dictionary named by
  the manifest, verifies its SHA-256, and caches it in IndexedDB. When
  the manifest announces a new dictionary version, the client downloads
  the new dictionary, keeps the old one until no cached file needs it,
  and re-compresses on next Publish with the new version. A file whose
  dictionary version the client does not know is an error, never a
  silent mis-decode.
- **Status in the scaffold — WIRED (2026-11-20), dictionary v2
  "v2-trained":** 32,768 bytes, ID `0x41524B02`, SHA-256
  `a1786956c8f9b919c393ba27347213c8db48a283615b23bfe03582f92544c96d`
  (pinned in code and recomputed by test). **Genuinely trained:**
  `zstd --train --maxdict=32768 --dictID=1095912194` (fastcover,
  zstd 1.5.5) on a 3,400-file / ~2 MB corpus generated from the
  project's own JSON schemas and sample site/registry/contact/mail/
  directory documents (`dict-training/`, generator + training script
  kept in the tree). The corpus prose is combinatorially unique, so
  the trainer learns JSON structure, backend/URL shapes and common
  words — not memorised sentences (the first attempt reused a dozen
  fixed sentences and lost to v1 on held-out documents; it was
  discarded and the corpus rebuilt). The manifest announces v2:
  `zstdDictionary: { "version": 2, "url": "…/zstd-dictionary-v2.bin",
  "sha256": "a178…" }`. Measured
  (in-app engine, 100 sampled corpus files, 104,600 raw bytes):
  plain 48,227, v1 36,962, **v2 22,769** — v2 wins on all 100 files.
  Held-out documents (nothing in them appears in the corpus): novel
  registry v2 104 vs v1 139, novel mail v2 184 vs v1 208 — v2 wins.
  Honest exception: on long-prose site documents v1 still wins
  (held-out harbour site v1 497 vs v2 574, plain 711; the demo site
  v1 152 vs v2 619, because the demo site's text is verbatim inside
  v1's hand-built content — pure overfit, not a fair comparison).
  v1 wins those shapes because its content repeats the JSON
  structural fragments with a density no trainer reproduced
  (tripling the fragment share and switching to the cover algorithm
  were both tried and changed nothing on held-out docs). The
  previous dictionary **v1 "v1-corpus-derived"** (26,300 bytes, ID
  `0x41524B01`, SHA-256
  `9e7346fce69bd694f438c83acced640aba7f804f5901ad4dd932d0d4816b19ae`)
  is kept for decoding old files: the decoder tries v2, then v1,
  then plain, and an unknown dictionary throws naming the
  dictionaries it knows. A formatted dictionary (RFC 8878 §5) is
  required — a raw-content dictionary was proven to mis-decode
  silently, so the decoder checks the dictionary ID and a
  wrong/missing dictionary throws. Pre-dictionary plain frames still
  decompress.
- **DEPRECATED 2026-11-21 (Stanley's decision — the dictionary is
  dropped, new files are plain zstd).** The benchmark
  (BENCHMARK-0.4.0.md) proved the v2 dictionary saves ZERO bytes on
  every realistic payload tested (registry 0/100/1000/4096 entries,
  50 KB site, demo site), and re-measurement during the drop
  (2026-11-21, in-app engine) confirmed the advantage is negligible
  at realistic scale: 1,000-entry registry (126,451 raw bytes)
  plain 4,224 vs v2 3,970 — 254 bytes, 0.2% of raw; 92 KB site
  plain 182 vs v2 141 — 41 bytes; and on a 100-entry registry v2 is
  WORSE by 577 bytes (1,306 vs 729). The dictionary only ever won
  on tiny documents (tens of bytes on ~150-byte frames) — never
  enough to justify a 32 KB download plus decode complexity. So:
  new files are written with **plain zstd, no dictionary** (frames
  carry no dictionary ID); the `register.yml` workflow no longer
  fetches any dictionary and uses bare `zstd -d` / `zstd`; the
  manifest's `zstdDictionary` field is deprecated and ignored by
  the client (it never fetched the dictionary at runtime — the
  dictionaries were embedded in the client). The v1/v2 dictionary
  `.bin` files stay published in the `arknetz-protocol` repo
  (marked deprecated there, never deleted) and the decoder keeps
  trying v2, then v1, then plain — every frame already published
  still decodes. `dict-training/` is kept as the training record.

## 5. Registry and shard rules

- **Sharding:** 16 hash shards to start (`arknetz-registry-00` …
  `arknetz-registry-0f`), expandable to 256 (`00`–`ff`) without moving
  domains. Contacts shards (`arknetz-contacts-00` … `0f`) use the same
  key, by address.
- **Shard key:** first hex digit of `SHA256(domain)` (lowercased) for
  16 shards; first two hex digits for 256. Clients compute it — never
  trust a server to — and reject an entry whose `shard` field disagrees.
- **Shard width is in the manifest.** `registryShards` lists the shard
  repo names; its length (16 or 256) tells the client the width.
- **Claims:** completely open — no minimum account age, no per-user
  domain-claim limit (Stanley rejected both). Claims go via a GitHub
  Issue (`REGISTER: computers.edu`); a free `register.yml` Action checks
  availability / banned words, commits the registry, closes the issue
  (~5–10 s). The Master writes direct to main (<1 s).
- **Central growth (thread numbers, UNVERIFIED, except contacts):**
  registry ~15 MB per 1M domains, contacts **~35.3 MiB per 1M users
  measured, stored form** (see §3.5), feed pointers ~30 MB per 1M
  posts, central ~20–50 MB over 10 years at 1M+ domains. A monthly
  `cleanup.yml` Action squashes history (`git checkout --orphan`) to
  keep each central repo under ~50 MB.
- **Mirrors and failover:** a registry entry may list `mirrors`
  (e.g. `["gh:alice/site", "cb:alice/site-mirror"]`). Clients try them
  in order with a per-mirror timeout and automatic failover. Central
  data is mirrored to GitLab (`gl:`) and Codeberg (`cb:`) by a scheduled
  Action, and each repo is snapshotted to the Wayback Machine (§9).
- **Per-user repos (auto-created by the app):** 3 shared repos
  (`arknetz-inbox`, `arknetz-mail-archive` (private), `arknetz-vault`)
  + 1 repo per domain (`arknetz-site-<domain-slug>`), holding
  `site.json.zst`, `posts.json.zst`, `delegations.json.zst`,
  `sub_moderation.json.zst`, `team.json.zst`, `stats.json.zst`, and
  comments (Discussions/Issues).

## 6. Encryption formats

All encryption is client-side Web Crypto (or Node `crypto.subtle` in
tests). No key material is ever in a repo, a URL, or a log.

### 6.1 Mail — hybrid envelope (thread turns 16–18)

- Compress the message, encrypt once with a random **AES-256-GCM**
  key, wrap only the 32-byte key per recipient. **Key wrapping is
  X25519 (implemented 2026-11-20, replacing RSA-OAEP-2048):** per
  recipient a fresh ephemeral X25519 keypair → `shared =
  X25519(ephemeralPriv, recipientPub)` → `KEK = HKDF-SHA-256(shared,
  salt = ephemeralPubRaw, info = "arknetz-mail/1
  key-wrap:<fingerprint>")` → `wrapped = AES-KW(KEK, CEK)`. All-zero
  shared secrets are rejected. **Measured wrapped-key entry: 72 bytes**
  (32 ephemeral + 40 AES-KW) — the thread's ~64 did not count AES-KW's
  8-byte integrity value; 72 is the sound minimum for an authenticated
  wrap. Engine floor: standard `{ name: "X25519" }` (Chrome/Edge 133+,
  Firefox 130+, Safari 17+). No envelopes exist in the wild (nothing
  shipped), so no migration was needed.
- `To:` / `CC:` live inside the encrypted header (so Reply All works);
  `BCC:` recipients get a wrapped key but are stripped from the inner
  header.
- Small groups (1–10): payload + wrapped key per inbox. Large lists:
  payload stored once in the sender outbox, pointer (~316 bytes
  measured, §3.11) + wrapped key per inbox.
- Delivery: `gh:<user>/protocol-inbox` Issues (any authenticated user
  can drop in, cannot delete others); the recipient decrypts locally,
  moves the mail to IndexedDB or the private `arknetz-mail-archive`,
  and closes/deletes the inbox item so the inbox stays near 0 KB.
- Public keys come from the contacts directory (§3.5), generated via
  Web Crypto on first login. There is **no Gmail/SMTP bridge**, by
  design — bridging needs a paid relay, SPF/DKIM/DMARC, and spam
  monitoring, and Gmail blacklists the sender within days.
- **PROPOSED envelope fields** (thread fixes the construction, not the
  serialisation): `{ "v": 1, "alg": "AES-256-GCM", "iv": "<base64>",
  "ciphertext": "<base64>", "keys": { "<address>": "<base64 wrapped key>" },
  "header": { "to": ["…"], "cc": ["…"], "subject": "…" } }`
  with `header` itself inside the ciphertext in the final form — the
  exact field layout is to be fixed when the mail phase lands, and must
  be recorded here before it ships.

### 6.2 Keyring — AccountKeyring (thread turn 45)

- **AES-256-GCM**, key derived from the vault passphrase via
  **PBKDF2, 100,000 iterations, SHA-256** (same construction as the IA
  S3 keys in the scaffold). Salt 16 bytes, IV 12 bytes, fresh per secret.
- Envelope (JSON string): `{ "v": 1, "salt": "<base64>", "iv":
  "<base64>", "data": "<base64>" }`. The passphrase is never stored.
- Profiles: one labelled Account Profile per provider —
  `github`, `gitlab`, `codeberg`, `internet_archive`, `huggingface`,
  `wikimedia` — each holding an encrypted token, or for IA an encrypted
  `{"accessKey","secretKey"}` JSON blob. A wrong passphrase fails at
  unlock, with the accounts still stored (encrypted).
- Per-site auto-binding (§10.3) is stored alongside the keyring, in the
  clear (it names accounts and domains, never secrets).

### 6.3 ArkDrive vault — zero-knowledge (thread turn 46)

- **Compress with Zstd, then encrypt both file contents AND
  filename/folder metadata with AES-256-GCM.** Key derived from a
  master vault passphrase via **Argon2id** (OWASP first-factor
  recommendation: memory 19 MiB, iterations 2, parallelism 1,
  output 32 bytes, version 0x13), implemented with `@noble/hashes`
  (MIT). New vaults are Argon2id; the envelope carries a version
  flag (`ARKV` magic + version byte `0x02` + salt + Argon2id
  parameters + IV + ciphertext), so the decoder detects the KDF.
  **Backward compatibility:** legacy PBKDF2 vaults (100,000
  iterations, SHA-256; the v1.0 construction — salt 16 + IV 12 +
  ciphertext, no magic) still open, and are **migrated to Argon2id
  on unlock**: the exact manifest bytes are re-encrypted under a
  fresh Argon2id key and fresh salt and stored before any in-memory
  state changes, so a failed migration leaves the old envelope
  untouched. A wrong passphrase fails before any migration, with the
  stored bytes byte-identical. (The account keyring in §6.2 and the
  IA S3 keys in §6.4 still use PBKDF2 — the Argon2id upgrade covers
  ArkDrive vaults only.)
- The vault manifest is `vault-index.enc.zst` (§3.13); chunks are
  random-named `blob_*.enc`, 10 MB each, SHA-256 hashed for dedup.
- Unified "Pooled Drive" view: all linked private vault repos in one
  folder tree, available space per backend.

### 6.4 IA S3 keys

- The user's own keys from `archive.org/account/s3.php`, stored
  AES-256-GCM encrypted in IndexedDB (PBKDF2, 100k iterations) — never
  hard-coded, never in code. If IA is not linked, uploads report
  `ia_not_linked` and upload nothing.
- **Per-user key rule (Stanley's correction, 2026-11-20):** a real IA
  upload is done with the uploading user's INDIVIDUAL IA S3 keys.
  Everyone will NOT use Stanley's keys — each user signs in with their
  own Internet Archive account (Settings → link IA account) and the
  stealth upload uses only that account's keys, into that account's
  own monthly item (`arknetz-media-<username>-<YYYY-MM>`). There is no
  shared or master IA key anywhere in the app, and no fallback to any
  other account's keys. `uploadToIa` throws before any network call
  when any key field is blank; `attachMedia` with blank keys returns
  `ia_not_linked` and uploads nothing; `testIaConnection` with blank
  keys reports not-ok without a network call. A live IA upload has
  never been performed from this scaffold (mock-verified only) — it
  needs a test account's own keys (see INTEGRATION.md, fix round 3).
- The same per-user rule applies to every backend (§1 principle 8):
  `gh:` / `gl:` / `cb:` / `hf:` writes need the acting user's own
  token, ArkDrive vault clients refuse every operation without the
  vault owner's own token (Hugging Face reads are the one exception:
  open-science datasets are public on the Hub, so reads need no token;
  Hub writes still do), and protocol-mail delivery refuses an empty
  sender token before building the call.

## 7. Sandboxed `["app", …]` postMessage APIs

HTML5 apps/games run in `<iframe sandbox="allow-scripts
allow-pointer-lock">` with a strict CSP (only
`raw.githubusercontent.com` / `archive.org`), `referrerpolicy=
"no-referrer"`, no top navigation / popups, `srcdoc` single bundle,
~150 KB compressed cap (larger sprites/audio hotlinked from IA). Raw
HTML/JS is never injected into the main DOM — it could steal the GitHub
OAuth token / IA S3 keys from IndexedDB.

**Save-state bridge (thread turns 15, 21) — IMPLEMENTED in the scaffold
(2026-11-20, `src/lib/viewer/app-bridge.ts`, wired into the shell reader
in `src/main.ts`):** per-domain IndexedDB save slots via a sandboxed
`postMessage` bridge. The specified message shape is:

```js
// App → parent (inside the sandboxed iframe):
window.parent.postMessage({ arknetz: 1, op: "save", slot: "slot-1", state: { /* JSON */ } }, "*");
// App → parent: load a slot
window.parent.postMessage({ arknetz: 1, op: "load", slot: "slot-1" }, "*");
// Parent → app (reply, same origin-checked channel):
window.postMessage({ arknetz: 1, op: "loaded", slot: "slot-1", state: { /* JSON */ } | null }, "*");
```

Rules: the parent accepts messages only from the iframe it created,
only with `arknetz: 1`, only `save` / `load` / `delete` / `list` ops
(**`delete` / `list` are now the fixed names**, with replies `saved` /
`loaded` / `deleted` / `listed` / `error` and a `requestId` echoed in
every reply), only JSON state, and only into the save slot namespaced
by the owning domain — an app on `games.example` can never read or
write `games.other`'s slots. Authentication is three checks, all
required: origin `"null"` (srcdoc's opaque origin, never trusted
alone), a per-iframe session id baked into that iframe's srcdoc only,
and the `event.source` the parent created (first valid message binds
it). Unauthenticated messages are dropped silently; authenticated but
invalid payloads get an `error` reply and write nothing. Limits: one
slot ≤ 64 KiB serialised, ≤ 16 slots per domain, slot names
`^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`, state must be plain JSON (max depth
64). The parent never exposes tokens, keys, or any IndexedDB store
other than that domain's save slots to the iframe. The iframe side
exposes exactly `window.ArkNetzSave = { save, load, delete, list }`.
URL-sourced apps (`appSrc`) get no shim — they can use their own
origin's storage.

## 8. Pluggable backends

Storage is addressed by URI prefix, never hard-coded to one provider.
Implemented in `src/lib/network/backends.ts`.

| Prefix | Provider | Read shape | Write shape |
|---|---|---|---|
| `gh:` | GitHub (default) | `raw.githubusercontent.com/{owner}/{repo}/main/{path}` | Contents API `PUT /repos/{owner}/{repo}/contents/{path}` |
| `gl:` | GitLab | `gitlab.com/{owner}/{repo}/-/raw/main/{path}` | Repository Files API `PUT /projects/{id}/repository/files/{path}` |
| `cb:` | Codeberg | `codeberg.org/{owner}/{repo}/raw/branch/main/{path}` | Forgejo Contents API `PUT /repos/{owner}/{repo}/contents/{path}` |
| `ia:` | Internet Archive | `archive.org/download/{item}/{file}` | IA S3 API only (see `ia-media.ts`); the backend layer refuses generic writes |
| `hf:` | Hugging Face Hub | `huggingface.co/datasets/{owner}/{repo}/resolve/main/{path}` | Hub preupload + NDJSON commit: `POST /api/datasets/{owner}/{repo}/preupload/{branch}`, then `POST /api/datasets/{owner}/{repo}/commit/{branch}` (shape **verified** 2026-11-20 against the official Hub clients; **NOT live-verified** against a real HF account — no live upload has ever run) |
| `wm:` | Wikimedia Commons | `commons.wikimedia.org/wiki/Special:FilePath/{file}` | MediaWiki upload API only; the backend layer refuses generic writes |
| `ipfs:` | Filebase IPFS | `ipfs.filebase.io/ipfs/{cid}/{path}` | Filebase S3-compatible API only; the backend layer refuses generic writes |

- **Registry entries list mirror arrays** with automatic failover in
  order (try the next mirror on failure, per-mirror timeout).
- **Per-user credentials on every write (fix round 3, 2026-11-20):**
  every backend write is authenticated with the acting user's OWN
  account credential (§1 principle 8, §6.4). `gh:` / `gl:` / `cb:` /
  `hf:` writes in this layer throw without a token ("sign in first");
  `ia:` / `wm:` / `ipfs:` writes are refused here outright and go
  through their dedicated flows with the user's own keys. There is no
  master/shared credential and no fallback to another account.
- **ToS-aware routing:** the client only offers `hf:` or `wm:` as
  upload targets when the creator marks the project as an open dataset /
  educational resource / Creative Commons work — enforced in code
  (`assertUploadAllowed`), not just in the UI. Hugging Face public repos
  must be useful to AI/ML / open science, NOT arbitrary personal cloud
  drives. Wikimedia Commons accepts strictly freely licensed
  (CC-BY / CC-BY-SA / Public Domain) educational / encyclopedic media.
- **UNVERIFIED:** no live GitLab, Codeberg, Hugging Face, Wikimedia, or
  Filebase account was used while writing these clients; every read/write
  shape above is the provider's documented public URL, exercised only
  against mocks in `tests/backends.test.ts`. Verify each against a real
  account before claiming it works end to end.

## 9. Wayback Machine

- **Snapshot to Wayback Machine / Preserve Snapshot** (Publish or Site
  Settings): Save Page Now, `POST https://web.archive.org/save/
  <public-raw-URL>` — pointed at the site's underlying public raw URLs
  (e.g. `raw.githubusercontent.com/<user>/<repo>/main/site.json.zst`,
  or the exported gh-pages static HTML mirror, which then becomes
  viewable on the traditional web). **Wayback archives public
  `https://` URLs only; it does not crawl `arknetz://` directly.**
- **Time Machine viewer:** the free Wayback CDX API,
  `https://web.archive.org/cdx/search/cdx?url=<url>&output=json&
  fl=timestamp,original,statuscode,mimetype,digest&filter=statuscode:200
  &collapse=digest`, lists every historical snapshot; readers/owners
  open a Wayback History dropdown to view a domain months/years ago.
- **Automatic read-only failover:** if the primary GitHub/Codeberg repo
  is deleted or unreachable, the client falls back to the latest
  preserved `.json.zst` snapshot from
  `https://web.archive.org/web/<timestamp>id_/<original>` — the `id_`
  form returns the captured bytes un-rewritten, so a preserved
  `site.json.zst` comes back byte-identical. The UI labels restored
  content as from a Wayback snapshot, with its date.
- **UNVERIFIED:** no real Wayback snapshot was taken or read while
  building this; the request/CDX shapes are the documented public API,
  exercised only against mocks.

## 10. Accounts, root registries, and moderation authority

### 10.1 Multi-account (thread turn 45)

- All tokens/S3 keys live encrypted locally in IndexedDB (§6.2), never
  in one rigid server session. Personal, organisation/school, and
  side-project accounts stay signed in simultaneously; switching is
  one-click and never logs anything out.
- The Account Switcher Pill (top bar) shows the active GitHub account,
  organises the dropdown by backend with avatar/username, and offers
  "+ Add Another Account".
- One-click switch works across **every** backend simultaneously.

### 10.2 Master Admin Badge

- Root Master controls appear only when the **active GitHub account**
  is `PropTheRtRevDrSirStanleyG05` (`MASTER_GITHUB_USER`). Switching to
  a regular alt account hides them, so the Root Master can test the
  normal user experience. Enforcement is GitHub's 403 on protocol-root
  writes, never the UI.

### 10.3 Smart per-site auto-binding

- Each site remembers which backend accounts own it (e.g.
  `edu.orion.stanbeyond` → `gh:stanbeyond` + `ia:ia-edu`;
  `indie.films` → `gh:stan-personal` + `ia:ia-cinema`). Opening a site
  auto-selects the matching accounts, so a commit/upload never goes to
  the wrong account. If the bound account is not signed in, opening the
  site fails with the exact reason — it never silently falls back to
  whatever account happens to be active.

### 10.4 Configurable root + multi-root peering (thread turn 40)

- `DEFAULT_ROOT_REPO = "PropTheRtRevDrSirStanleyG05/arknetz-protocol-root"`
  by default, so forks see Stanley's sites. (The thread's
  `stanbeyond/protocol-root` is superseded: `github.com/stanbeyond` is
  404; the confirmed master account is `PropTheRtRevDrSirStanleyG05`.)
- Settings holds a **Root Registries** list: the primary default plus
  optional secondary community registries (e.g. `community/alt-root`),
  without fragmenting the main web. A fork points its primary at its
  own registry and may keep the main one as a secondary.
- The protocol-root manifest is tiny: current protocol version,
  `DEFAULT_ROOT_REPO`, master public identity, shard-repo list and shard
  keys, mirror locations, and the `directoryCategories` list (§3.6).
  It never grows. Clients fetch it first, then
  go straight to the shard they need; if it is down, they fall back to
  a cached copy in IndexedDB.
- **Peering boundary (open question from the thread, still open):** a
  fork's users connect to Stanley's sites, but his Root Master authority
  does not extend into a fork's own registry. A fork's Root Master
  governs only its own registry.

### 10.5 Bring-your-own OAuth (thread turn 40)

- `GITHUB_OAUTH_CLIENT_ID` is isolated in a single `src/lib/config.ts`.
  A fork plugs in its own free GitHub OAuth App ID in 30 seconds and
  still connects to the default root. The client secret must never be
  in the app bundle — the code exchange happens in the Tauri Rust
  command, which returns `not_configured` until a secret is supplied
  via the build environment on the builder's own machine.

## 11. Terms of Service (pass-through)

ArkNetzWebWeaver adds no separate legal regime. Users comply with the
**GitHub Terms of Service** (`https://docs.github.com/en/site-policy/
github-terms/github-terms-of-service`) and the **Internet Archive Terms
of Use** (`https://archive.org/about/terms.php`). Short plain-English
summaries live in `src/lib/network/tos.ts`; the summaries are not the
terms — the links are.

- **Guests** see a passive Acceptable Use notice (footer / about modal).
- **First-time creators** get a one-click
  "I agree to the ArkNetzWebWeaver Terms, GitHub ToS, and Internet
  Archive Terms" checkbox before their first domain claim or publish,
  with a **timestamped acceptance flag** stored per account.
  Acceptance is required before a first publish/claim can proceed; the
  gate returns the exact missing-acceptance reason.
- Users keep copyright and bear sole legal responsibility. Registrars
  and the Root Master hold takedown authority (§3.4).

## 12. Web standards (thread turn 41)

Tauri uses the OS native standards-compliant engine (WebView2/Chromium
on Windows, WebKitGTK on Linux, Android System WebView): HTML5, CSS3
(Grid, Flexbox, Container Queries, `@layer`, Custom Properties),
ES2025+ JavaScript, WebAssembly, WebGL 2/WebGPU, WebAudio, WebCodecs,
Web Crypto, ARIA accessibility. Speed comes from zero network
waterfall, not from dropping standards: layout engines are bundled
locally in the ~7 MB binary, page data arrives as a single ~10 KB Zstd
payload. Standard URIs/MIME types, W3C Fetch API, ETag/If-None-Match,
IndexedDB, Web Crypto, WebCodecs, standard WebP/AVIF, Opus, H.264/AV1
MP4. `https://` links open cleanly in an external browser or standard
web tab; `arknetz://` resolves through the protocol. Static HTML export
outputs clean W3C-valid semantic HTML5 + CSS3, target 100/100 on
Google Lighthouse. **UNVERIFIED:** the Lighthouse target and the
"50–100× lighter" claim are thread numbers, not measured results.

## 13. Deep links and offline bundles

- `arknetz://<domain>`, e.g. `arknetz://edu.orion.stanbeyond`,
  OS-registered by Tauri on Linux/Windows/Android.
- **Registration (wired in the client, fix round 3, 2026-11-20):**
  `tauri-plugin-deep-link` v2, registered in `tauri.conf.json`
  (`plugins.deep-link.desktop.schemes: ["arknetz"]`, mobile
  `scheme: ["arknetz"]`) with `withGlobalTauri: true`, the plugin
  initialised in `src-tauri/src/main.rs`, and the `deep-link:default`
  capability permission. Incoming URLs are handled in
  `src/lib/deep-link.ts`: `parseDeepLinkUrl` accepts only the
  `arknetz` scheme with a valid domain (query/fragment dropped,
  credentials rejected), `handleDeepLinkUrls` opens valid links in
  order and skips the rest, and `installDeepLinkListener` subscribes
  to Tauri's `onOpenUrl` at startup (plus any URL the app was
  launched with). **Manual check — NOT yet verified by launching
  the app:** install the built app, then run
  `xdg-open 'arknetz://computers.edu'` (Linux) / `start
  arknetz://computers.edu` (Windows) / tap a link on Android, and
  confirm the app opens the site. Debug builds register the scheme
  only in development (`register_all()` in debug) so a dev binary
  can be tested without hijacking the scheme. **Known gap:** no
  single-instance plugin is wired, so on Linux/Windows a deep link
  clicked while the app is already running spawns a second process
  (the URL still arrives as a CLI argument and that window opens
  it); focusing the running window needs `tauri-plugin-single-instance`
  with the deep-link feature. Android registration is config-only —
  no .apk has been built or tested.
- `.arknetz` offline site bundles: one-click Save Site Offline, a
  single portable archive shareable over USB/Bluetooth/Discord, opening
  with zero internet. **PROPOSED:** the bundle format is a ZIP
  containing the site's `.json.zst` files plus its IA media manifest;
  the exact layout is to be fixed when bundles land.

## 14. What this document deliberately does NOT specify

- **No AI builder, no MCP server, no x402 payments.** Excluded by
  Stanley, 2026-11-20 — out of scope, not missing.
- **No Fediverse/ActivityPub, no RSS/feed.xml.** Rejected in the thread.
- **No Gmail/SMTP bridge.** Rejected in the thread; mail is
  protocol-native only.
- **No central proxy** for IA rate-limiting (a central proxy would
  itself get rate-limited and need server bills). The fix is client-side
  caching + polite queue + authenticated IA session.

## 15. Changelog

- **v0.1.0 (2026-11-20):** first clean protocol document. Schemas for
  registry, site, posts, delegations, moderation, contacts, directory,
  feed, team, stats, mail_lists, comments-archive, vault-index;
  Zstd dictionary versioning; sandboxed postMessage save-state bridge
  (specified, not yet implemented); encryption formats (mail hybrid
  envelope, vault, keyring); shard rules (16 → 256); configurable root
  + multi-root peering; bring-your-own OAuth; Wayback Machine;
  pluggable backends with ToS-aware routing; multi-account keyring.
  Fields the thread does not define are marked PROPOSED; claims no
  live provider verified are marked UNVERIFIED.
- **Fix round 3 (2026-11-20, doc clarification, no schema change):**
  per-user key rule stated explicitly (§1 principle 8, §6.4, §8):
  every backend write/upload uses the acting user's OWN credential;
  no master/shared upload key exists; writes without the user's own
  credential fail loudly before any network call. Live IA upload
  still not performed — needs a test account's own keys.
- **Fix round 3 (2026-11-20, implementation):** (1) `arknetz://`
  deep-link registration wired via `tauri-plugin-deep-link` v2 (§13;
  launch verification still pending — see the manual check there).
  (2) ArkDrive vault KDF is now Argon2id (OWASP 19 MiB / 2 / 1),
  with PBKDF2 legacy vaults still opening and auto-migrating on
  unlock (§6.3; keyring and IA keys still PBKDF2). (3) Zstd
  dictionary v2 is genuinely trained (§4: 32,768 bytes, ID
  `0x41524B02`, SHA-256 `a1786956…`; v1 still decodes old files).
  (4) Directory categories are manifest-discoverable:
  `directoryCategories` in the protocol-root manifest (§3.6,
  §10.4), with `["tech", "cinema", "games"]` kept only as the
  fallback for manifests without the field.
- **Collaboration (2026-11-21, §3.14):** real-time multi-user editing
  via a `collab/<domain-slug>` branch in the site repo, one state
  file per peer (`peers/<login>.json`), Contents API with the user's
  own token (`public_repo`). Per-block LWW, conflict notes, 30 s
  presence expiry, ~5 s poll latency, no character-level OT — NOT
  Google Docs parity. Mock-verified only; no live round-trip has run.
- **Tables and lists (2026-11-21, §3.2.1):** `list` and `table` are
  first-class block types (`listStyle`/`listItems`, `rows`/`headerRow`),
  with the in-editor rich-text toolbar for both. Typed inline runs
  only — never raw HTML.
