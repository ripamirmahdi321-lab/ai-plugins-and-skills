# Limits, gotchas, and the fine print

Everything that can hurt you, in one place. Compiled 2026-09-16 from the official docs
(snapshot 2026-09-05), the Bot API changelogs, and community reports (Hacker News,
Habr, Remio, cysecurity). **The platform is ~2 months old at compile time — re-verify
against https://core.telegram.org/bots/serverless before making production decisions.**

## Hard limits (as of 2026-09)

| Area | Limit | Notes |
|---|---|---|
| Language | JavaScript (ES modules) only | No TypeScript build step, no Python, no compiled languages |
| Packages | **No npm at all** — only `sdk` + your own modules | Re-implement or inline any helper you need in `lib/` |
| Filesystem | None | No reading/writing files on disk, no env files, no logs to disk |
| WebAssembly | No | |
| Node APIs | No `Buffer` (use `Uint8Array`), no `process`, no `fs`, no `net` | The runtime is a V8 isolate, not Node |
| HTTP response | **Text-only**, **30 MB** total per response | Was 32 MB at launch; streaming (`res.body`) doesn't raise it. No binary responses (e.g. you can't fetch a remote image binary — get text/JSON, or use Telegram-side media) |
| File upload to Telegram (Bot API) | **50 MB** per file; **10 MB** for photos | via `InputFile` / `api.send*` |
| File download from Telegram | **20 MB** | `getFile` cap; `api.getFileContent` / `api.getFileStream` |
| SQLite FKs | **Disabled** (`PRAGMA foreign_keys` off) | `.references()`/`foreignKey()` throw at declaration; app-level integrity only |
| Batch inserts | SQLite variable limit (`rows × columns`) | Chunk yourself |
| Scheduling | **No timers/cron documented** | Event-driven only (see gotcha below) |
| Secrets management | **None documented** | No secret store; keys live in code/modules |
| DB export | **No documented export mechanism** | Lock-in consideration |
| Pricing / quotas | Not documented on the platform page | No published fees, rate limits, or storage quotas as of 2026-09 |
| Cold start | "fast" (platform claim); community reports **< 5 ms** typical | V8 isolates start in ms, unlike container-based serverless; not independently benchmarked |

## Design gotchas

### 1. No cron / timers — rethink periodic work

The runtime wakes on updates only. There is no `setTimeout`-based scheduler service,
no cron, no documented "run every N minutes" primitive.

- **Reminders / digests / sweeps**: store `next_fire_at` in SQLite and act on the next
  relevant interaction (lazy evaluation), or:
- **External trigger**: a tiny always-on service (or another platform) that pings your
  bot periodically (e.g. sends a private message to a service bot that relays, or hits
  a public endpoint you expose via Mini App), or
- **Companion service**: keep one small always-on process whose only job is to
  generate the trigger updates.

### 2. No persistent storage outside the DB and Telegram

There is no disk. Long-lived artifacts must live either in SQLite (BLOB columns up to
SQLite's page limit — fine for small/medium data) or **back in Telegram**
(`file_id`s in the DB; re-send media to a private backup channel via `api.sendDocument`)
or in an **external HTTP service** (S3-compatible, your own server, …).

- A `file_id` obtained through your bot stays resolvable by your bot.
- Download cap 20 MB / upload cap 50 MB shape what you can move.
- For a **backup bot**: the natural design = metadata + `file_id`s in SQLite, media
  mirrored into a private channel; document the 20 MB download ceiling for large files.

### 3. No foreign keys — model relations deliberately

- Insert parents before children; delete children before parents.
- Index every logical FK column you query by.
- Sweep orphans when correctness demands it:
  `SELECT child.id FROM child LEFT JOIN parent ON child.parent_id = parent.id WHERE parent.id IS NULL`
  then delete (raw SQL or builder).
- CHECK constraints and `UNIQUE` are your friends — use them in the schema extras.

### 4. `await` everything DB

The #1 porting bug from Drizzle/Node: a forgotten `await` on `.all()`/`.get()`/
`.run()`/`db.$count()`/raw `db.*` gives you a Promise (or a builder), not data. Lint
or review for this specifically.

### 5. Raw SQL rows are unconverted

`db.all/sql` rows come back as raw SQLite values: boolean → 0/1, json → string
(JSON.parse it), timestamp → unix seconds (new Date(...*1000)). Only table-bound
builder queries convert via `mode`.

### 6. Bodies are read once (fetch)

Second `res.json()`/`res.text()`/stream throws `TypeError: body used already`.
Cloning a response isn't part of the documented surface — read once, store what you need.

### 7. Deploy is a mirror, not a copy

`npx tgcloud push` (no args) makes the cloud **exactly** match your folder — deleting a
file locally removes it from production on next push. That's a feature (no stale
modules) but means: don't leave half-deleted work in the folder, and be careful with
`reset`/`pull` in the wrong direction. Targeted pushes (`push handlers/message.js`)
don't delete untouched modules.

### 8. Conflicts and force

Optimistic concurrency via cloud revision. A rejected push ≠ failure of your code —
someone else deployed. `fetch` → review → `pull` (or rebase your work) → `push`.
`--force` overwrites the other party's work; use only with coordination.

### 9. The webhook is managed — don't touch it with the raw token

Calling `setWebhook` yourself (e.g. with the classic bot token) desyncs the
platform-managed webhook. Repair with `npx tgcloud webhook sync`. If you're migrating
an existing bot off polling/webhook, this is the cutover step: push handlers, then
`webhook sync` (optionally `--drop-pending`).

### 10. Secrets have no first-class home

No documented secret manager. Third-party keys (LLM API keys, payment keys) will live
in your modules (e.g. `lib/config.js`). Consequences:

- Anyone with the CLI token (or BotFather access) can read them — treat the project as
  containing secrets.
- Rotate by re-deploying; there's no separate secret versioning.
- For sensitive bots, consider a thin proxy service you own that holds the real key.

### 11. Privacy & lock-in (informed choices)

- Bot conversations were never E2E-encrypted (only Secret Chats are); with Serverless,
  **app logic and the bot's database** additionally live on Telegram's infrastructure.
  Data-residency/compliance requirements should be checked before storing sensitive PII.
- **No documented DB export** → plan an export path yourself if portability matters:
  a handler that dumps tables (JSON via `fetch` to your own storage, or as documents to
  a backup channel), scheduled manually. Code is portable (it's your files); the data
  is the lock-in.
- Availability: an outage of the platform execution/storage disables the whole app
  path (no independent recovery point). For revenue-critical bots, keep a fallback plan.

### 12. AI-specific

- The scaffold ships `AGENTS.md` + `docs/tgcloud-sdk.md` precisely so coding agents
  get the conventions right (bare imports, no FKs, async db, one handler per update
  type, push/migrate split). **Keep `AGENTS.md` updated** as the bot grows — the docs
  explicitly say it's part of your project.
- Streaming AI answers: `fetch` + `for await (const chunk of res.body)` +
  `api.sendRichMessageDraft` (Bot API 10.1+) — see `examples.md`.
- No CORS/origin issues server-side, but remember Bot API 10.2 (2026-07-20) hardened
  Mini App origin protection — Mini Apps can't call your backend from foreign origins;
  they talk to it through the platform's authenticated bridge (init data), so keep the
  Mini App domain stable.

## Launch timeline (what changed since July)

| Date | Event |
|---|---|
| 2026-07-12/13 | `@tgcloud/cli` 0.1.0–0.1.2 published on npm (maintainer: Telegram's `asmico`); docs page appears (first wayback capture 2026-07-13) |
| 2026-07-14 | Bot API 10.2: Rich Message blocks/media, ephemeral messages, Communities, `Update.subscription`, Mini App origin hardening (enabled 2026-07-20) |
| 2026-07 (mid) | Launch coverage: HTTP response cap quoted as **32 MB**; file upload/download from handlers reported as **not yet available** |
| 2026-08-24 | Bot API 10.3: more Rich Message features (buttons, compact tables, expandable quotations, document blocks) |
| ≤ 2026-09-05 | Docs snapshot shows: **30 MB** HTTP response cap; **file handling available** (`InputFile` upload up to 50/10 MB, `api.getFileContent`/`getFileStream` download up to 20 MB); `webhook`, `completion`, `reset`, `add` commands documented |
| 2026-09-16 | This skill compiled |

Treat any "as of launch" numbers from July articles as stale where the 2026-09 docs
differ.
