---
name: telegram-serverless
description: Build, test, and deploy Telegram bots and Mini App backends on Telegram's serverless platform (tgcloud) — plain JavaScript modules running in V8 isolates on Telegram's own infrastructure, with a built-in SQLite database and one-command deployment. Use this skill whenever you are working in a tgcloud project (a folder containing schema.js, handlers/, lib/, or a .tgcloud/ directory), when asked to create, modify, test, deploy, or migrate a serverless Telegram bot or Mini App backend, or when the user mentions tgcloud, BotFather "Serverless", or running bot code on Telegram's infrastructure.
---

# Telegram Serverless (tgcloud)

Telegram Serverless runs your bot's backend code **on Telegram's own infrastructure** —
no VPS, no cloud functions, no webhook configuration, no scaling. You write plain
**JavaScript modules**, deploy them with one command (`npx tgcloud push`), and Telegram
executes them in a fast, isolated **V8 isolate** sitting right next to the Bot API and a
built-in **SQLite** database.

- Official docs: https://core.telegram.org/bots/serverless
- CLI package: `@tgcloud/cli` (command `tgcloud`), project creator: `npm create @tgcloud/bot`
- Launched ~July 13, 2026 (CLI 0.1.x). Requires **Node.js 18+**.
- This skill was compiled from the official docs (snapshot 2026-09-05), the official
  scaffold templates (`AGENTS.md`, `docs/tgcloud-sdk.md`), the Bot API 10.1–10.3
  changelogs, and community reports. **Verify limits against the live docs before
  shipping anything critical** — the platform is young and changing fast.

## When to use this skill

- You are in a folder that contains `schema.js` + `handlers/` + `lib/` (a tgcloud project).
- The user wants a new serverless Telegram bot, or to move an existing bot to serverless.
- The user asks about `tgcloud`, `tgcloud push/run/migrate`, the BotFather "Serverless"
  section, Mini App backends on Telegram, or "run the bot without a server".
- You are debugging bot code that imports from `'sdk'`, `'sdk/db'`, `'schema'`, or `'lib/…'`.

## Mental model

Three places, one loop:

| Where              | What lives there                                              |
|--------------------|---------------------------------------------------------------|
| Your project folder| JavaScript modules: `schema.js`, `lib/`, `handlers/`          |
| The cloud          | The deployed copy of those modules + the bot's SQLite database |
| The `tgcloud` CLI  | The bridge — shows diffs, syncs code, applies migrations      |

Event-driven loop: an update arrives (message, button press, inline query, …) →
Telegram routes it to the matching handler file → your `export default` function runs in
a V8 isolate → it talks to the Bot API and the database through the SDK → it returns.
**No matching handler ⇒ the update is silently ignored** (and never wakes your code).

## Project layout (memorize this)

```
my_bot/
├─ schema.js            # database schema — tables as NAMED EXPORTS. One file, at root.
├─ handlers/            # entry points — FLAT, one file per Telegram update type
│  ├─ message.js        #   name === update type (message, callback_query, …)
│  └─ callback_query.js
├─ lib/                 # shared modules — subdirectories ALLOWED (lib/payments/stripe.js)
├─ docs/                # reference docs — local only, never deployed
├─ AGENTS.md            # orientation for AI coding assistants — keep it accurate
├─ package.json
└─ .tgcloud/            # CLI state (credentials, snapshot, cache) — git-ignored, NEVER edit by hand
```

Only `schema.js` and `.js` files under `lib/` and `handlers/` are **deployed**.
Markdown, dotfiles, config files, and `.tgcloud/` stay on your machine. A stray `.js`
at the project root is flagged by `status` — the root is meant to hold only serverless
content.

## The rules that bite (read before writing any code)

1. **Import by bare module name, never relative paths or `.js` extensions.**
   The platform resolves names in a module space, not the filesystem.
   - ✅ `import { users } from 'schema'`
   - ✅ `import { addItem } from 'lib/cart'`
   - ✅ `import { db, api, fetch } from 'sdk'` / `import { eq, sql } from 'sdk/db'`
   - ❌ `import { users } from './schema'` — won't compile
   - ❌ `import x from 'lib/cart.js'` — drop the `.js`
2. **Only two things exist at runtime**: `sdk` (+ submodules `sdk/db`, `sdk/api`,
   `sdk/fetch`) and your own modules. **No npm packages, no filesystem, no WebAssembly,
   no Node `Buffer`** (use `Uint8Array`). If you `import` anything else, it won't resolve.
3. **Every database call is async — always `await`.** `.all()`, `.get()`, `.values()`,
   `.run()`, `db.$count()`, and raw `db.run/all/get` all return Promises. A forgotten
   `await` returns a builder, not rows. (Builders are themselves awaitable:
   `await db.select().from(t)` is equivalent to `.all()`.)
4. **No foreign keys.** `.references()` and table-level `foreignKey()` **throw at
   declaration** — a schema using them won't deploy. Model relations with plain columns
   (`userId: integer('user_id')`) and enforce integrity in app code: insert parents
   before children, delete children before parents, sweep orphans with
   `LEFT JOIN … WHERE parent.id IS NULL` when needed.
5. **Deleting a declaration does NOT drop a table/column/index.** Drops happen only via
   `.deprecated('reason')`; the next `migrate` shows the drop as a warning you confirm
   individually. Once gone, remove the declaration.
6. **Column type changes are manual** — do them yourself with raw SQL (`db.run(...)`),
   typically: add new column → copy data → drop old (via `.deprecated()`).
7. **A handler's `export default (payload, ctx)`** receives the update's *payload*, not
   the `Update`: `handlers/message.js` gets the `Message` (`update.message`),
   `handlers/callback_query.js` gets the `CallbackQuery`, etc. The full `Update`
   (with `update_id`) is on the second argument: `ctx.update`.
8. **Deploying never touches the database.** `push` uploads code; `migrate` applies
   schema changes. These are deliberately separate steps.
9. **`api` results are unwrapped and failures throw.** `api.getMe()` resolves to the
   user object, not `{ ok, result }`. On `{ ok: false }` it throws `BotApiError`
   (`.code`, `.description`, `.method`, `.parameters`).
10. **HTTP responses are text-only and capped (30 MB total).** No binary response
    bodies; stream with `res.body` for large payloads or AI token streams.

## Setup (one-time, per bot)

1. **Enable Serverless in BotFather**: open @BotFather → your bot → **Serverless** →
   turn it on. This unlocks the CLI access token, handlers, library, and database.
   (Everything can also be managed from BotFather's phone UI later.)
2. **Scaffold** (needs Node 18+):

   ```bash
   npm create @tgcloud/bot my_bot     # creates the project + installs the CLI locally
   cd my_bot
   ```

   Pass `.` to scaffold into the current folder. Never overwrites existing files.
   Alternative: `npm install -g @tgcloud/cli` then `tgcloud init`.
3. **Link the bot**:

   ```bash
   npx tgcloud login    # asks for the CLI access token (app<id>:<secret>)
   ```

   Token: BotFather → your bot → Serverless → **CLI Access** → Access token. It is a
   *separate* token from the bot's API token. Stored in `.tgcloud/credentials`
   (git-ignored); the secret is never printed.
4. **Deploy**:

   ```bash
   npx tgcloud push     # upload modules — bot is live
   npx tgcloud migrate  # apply schema.js changes to the database (only when schema changed)
   ```

## Writing code — quick reference

### Schema (`schema.js`)

```js
import { table, integer, text, boolean, json, index, check, sql } from 'sdk/db';

export const users = table('users', {
  id:      integer('id').primaryKey({ autoIncrement: true }),
  tgId:    integer('tg_id').unique(),
  name:    text('name').notNull(),
  lang:    text('lang').default('en'),
  isAdmin: boolean('is_admin').default(false),
  prefs:   json('prefs'),
  created: integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
}, (t) => ({
  createdIdx: index('idx_users_created').on(t.created),
}));
```

- Column factories: `text()`, `integer()`, `real()`/`float()`, `numeric()`, `blob()`
  (Uint8Array), `boolean()` (0/1 ↔ bool), `json()` (auto stringify/parse).
- Column name argument is optional (defaults to the JS key).
- `opts.mode`: `boolean`, `json`, `timestamp` (unix sec ↔ Date), `timestamp_ms`
  (unix ms ↔ Date), `bytes` (BLOB ↔ Uint8Array). Mode controls *encoding*, not storage:
  `blob('c', { mode: 'json' })` = JSON in a BLOB column.
- Modifiers (chainable): `.primaryKey({ autoIncrement: true })`, `.notNull()`,
  `.unique()`, `.default(v)` (`.default(sql`(unixepoch())` for SQL expressions),
  `.generatedAlwaysAs(sql`…`, { mode: 'stored'|'virtual' })`, `.constraint('COLLATE NOCASE')`,
  `.deprecated('reason')` (terminal — nothing after it).
- Table-level (in the extras callback, columns in scope as `t.col`):
  `primaryKey({ columns: [t.a, t.b] })`, `unique('uq').on(t.col)`,
  `check('chk', sql`…`)`, `index('idx').on(t.col)` / `.on(sql`lower(${t.email})`)` /
  `.on(t.a, t.b)` / `.where(sql`…`)` (partial), `uniqueIndex('uidx').on(t.col)`.
  `foreignKey(...)` **throws** — don't use it.
- Table modifiers: `.strict()`, `.withoutRowid()`, `.constraint('CHECK (x > 0)', 'name')`,
  `.deprecated('reason')`.

### Queries (Drizzle-like)

```js
import { db } from 'sdk';
import { users, todos } from 'schema';
import { eq, and, desc, asc, count, sql, inArray, or } from 'sdk/db';

// SELECT
await db.select().from(todos).all();                          // all rows
await db.select().from(todos).where(eq(todos.id, 1)).get();   // first row or null
await db.select().from(todos).values();                       // rows as value arrays
await db.select().from(todos)
  .where(and(eq(todos.userId, uid), eq(todos.done, false)))
  .orderBy(desc(todos.priority), asc(todos.id))
  .limit(10).offset(20).all();
await db.select({ id: todos.id, n: count() })
  .from(todos).groupBy(todos.userId).having(sql`count(*) > ${1}`).all();
await db.$count(todos, eq(todos.done, false));                // COUNT helper

// INSERT / UPDATE / DELETE — plain ones resolve to []; use .returning() to get rows
await db.insert(todos).values({ userId: 1, text: 'Buy milk' }).run();
await db.insert(todos).values([{ text: 'A' }, { text: 'B' }]).run();   // batch = ONE statement
await db.insert(users).values({ tgId: 42, name: 'Ann' })
  .onConflictDoUpdate({ target: users.tgId, set: { name: 'Ann' } }).returning().run();
await db.update(todos).set({ done: true }).where(eq(todos.id, 1)).run(); // .set() required
await db.delete(todos).where(eq(todos.id, 1)).run();

// Raw SQL — mode by method: run=write, all=rows, get=one row, values=positional arrays
await db.run('UPDATE todos SET done = 1 WHERE id = :id', { ':id': 5 });
await db.all(sql`SELECT * FROM todos WHERE done = ${false}`);
```

- Operators from `sdk/db`: `eq ne gt gte lt lte like notLike isNull isNotNull and or not
  between notBetween inArray notInArray count sum avg min max asc desc`.
- `.where(a, b)` with multiple args = `and(a, b)`. Comparison RHS may be a value, another
  column, or a `sql` fragment. Aggregates are `sql` fragments for projections.
- ``sql`…``` `` tag: `${value}` → bound parameter, `${table.col}` → identifier, nested
  `sql` spliced. `sql.raw('…')` for literals. **In DDL contexts (DEFAULT/CHECK/GENERATED)
  parameters don't work — use literal SQL.**
- **Raw SQL rows have no mode conversion** (boolean → 0/1, json → string, timestamp →
  number). Only the table-bound builder converts.
- Batch inserts are one statement — capped by SQLite's variable limit (`rows × columns`);
  chunk large batches yourself.
- Full details: `references/sdk.md`.

### Handlers

```js
// handlers/message.js
import { api } from 'sdk';

export default async function (message, ctx) {
  const chatId = message.chat.id;
  await api.sendMessage({ chat_id: chatId, text: `You said: ${message.text ?? '(no text)'}` });
}
```

- One file per update type, **flat** (no subdirectories). File name = update type.
- Common update types (the platform advertises the canonical set — run
  `npx tgcloud add handlers` with no name to list what's valid): `message`,
  `edited_message`, `channel_post`, `edited_channel_post`, `business_connection`,
  `business_message`, `edited_business_message`, `deleted_business_messages`,
  `my_chat_member`, `chat_member`, `chat_join_request`, `message_reaction`,
  `message_reaction_count`, `inline_query`, `chosen_inline_result`, `callback_query`,
  `shipping_query`, `pre_checkout_query`, `poll`, `poll_answer`, `chat_boost`,
  `chat_unpin`, `subscription` (Bot API 10.2, `BotSubscriptionUpdated`),
  `stopped_message_generation` (Bot API 10.3, `MessageGenerationStopped`).
- Only create handlers you need: each one is another update type that wakes your code,
  and the webhook's `allowed_updates` is derived from your deployed handlers.
- Scaffold with `npx tgcloud add handlers/callback_query` (validates the name; never
  overwrites) — the generated `export default` is live immediately.
- Handlers may return a value (shown by `tgcloud run`) and may be async (usually are).

### SDK surface (`import … from 'sdk'`)

| Import        | What it is                                                        |
|---------------|-------------------------------------------------------------------|
| `db`          | SQLite query builder + schema DSL (see above)                     |
| `api`         | The **entire** Telegram Bot API — `api.<method>({…})`. Proxy-based, so every current *and future* method works with no SDK update |
| `fetch`       | Outbound HTTP (web-fetch-like)                                    |
| `InputFile`   | File bytes + name + optional MIME `type` — works with `api` and `fetch` |
| `BotApiError` | Thrown by `api` on `{ ok: false }`                                |

- **`api`**: params use Bot API snake_case (`chat_id`, `message_id`, `reply_markup`).
  Result is unwrapped. On error it throws `BotApiError` with `.code` (400/403/429/…),
  `.description`, `.method`, `.parameters` (e.g. `retry_after` on 429,
  `migrate_to_chat_id`). Catch to handle expected failures:

  ```js
  try {
    await api.deleteMessage({ chat_id, message_id });
  } catch (e) {
    if (!(e instanceof BotApiError && e.code === 400)) throw e; // 400 = already gone
  }
  ```

- **Files** (available as of the 2026-09 docs; absent at launch):
  - Upload: `const doc = new InputFile(bytes, 'report.pdf', { type: 'application/pdf' });
    await api.sendDocument({ chat_id, document: doc });` — works anywhere a Bot API
    method expects a file, including nested in `sendMediaGroup`. Bytes are streamed as a
    real multipart upload. Bot API upload limits apply: **50 MB** per file, **10 MB** for
    photos.
  - Download: `const bytes = await api.getFileContent(file_id)` → `Uint8Array` (platform
    runs `getFile` + fetch for you — you never touch a download URL), or
    `const { file, body } = await api.getFileStream(file_id)` and
    `for await (const chunk of body)`. **20 MB** ceiling (Bot API `getFile` limit).

- **`fetch`** (outbound HTTP):

  ```js
  import { fetch, FormData, InputFile } from 'sdk';

  const res = await fetch('https://api.example.com/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name: 'Pavel' }),
  });
  if (!res.ok) throw new Error(res.statusText);
  const data = await res.json();
  ```

  - Response: `status`, `statusText`, `ok` (200–299), `url`, `headers`
    (`.get/.has/.keys/.entries`), `await res.json()` / `await res.text()`, or stream:
    `for await (const chunk of res.body)` — **use streaming for SSE / AI token streams**.
  - Body helpers set `Content-Type`: `fetch.body.json(…)`, `fetch.body.form(…)`,
    `fetch.body.text(…)`. Request bodies may also be `Uint8Array`, `InputFile`, or
    `FormData` (multipart) — streamed out.
  - A response body is readable **once** (second read throws; check `res.bodyUsed`).
    Redirects are followed automatically. HTTP errors resolve with `ok === false`;
    only real network errors reject.
  - **Constraints: text-only responses, 30 MB total response cap** (streaming doesn't
    raise it).

- **`console`**: `log/info/warn/error/trace` — captured and shown by `npx tgcloud run`
  (each line tagged `[file:line]`; `error`/`trace` append a stack). It is your primary
  debugging tool — there is no other logging surface documented.

## CLI cheat sheet (`npx tgcloud <command>`)

| Command | Purpose |
|---|---|
| `init` | Scaffold a project in the current folder (refuses to nest in another project) |
| `add <target>` | Scaffold one module: `add handlers/callback_query`, `add lib/cart`. Never overwrites |
| `login` | Link the project to a bot (saves token; the only command that prompts; needs a real TTY) |
| `status` | Per-file diff vs the deployed copy (offline); warns about stray root `.js` |
| `diff` | Line-by-line differences (offline) |
| `push [files…]` | Deploy in one atomic batch. No args = mirror your folder exactly (deletions included). File args narrow *which changes are sent* but still send the full manifest (targeted pushes never delete untouched modules). `--force` skips the concurrency check |
| `migrate` | Apply schema changes (interactive). Flags: `--dry-run`, `--safe` (auto-apply safe, skip warnings), `--yes` (auto-apply safe+warnings — careful), `--local` (diff against local schema.js) |
| `run <module> [args] [--ctx <json5>]` | Execute a handler on the platform using your **local** files, no deploy. `args` = the payload in JSON5; prints console output, return value, elapsed time |
| `fetch` | Refresh the local reference copy from the cloud (doesn't touch working files) |
| `pull` | Bring local files in line with the cloud |
| `reset` | Discard local changes; restore from cloud state |
| `webhook [sync]` | Show the platform-managed webhook (URL, allowed_updates, pending count, last error, in/out-of-sync). `sync [--drop-pending]` re-points it and rebuilds `allowed_updates` from deployed handlers |
| `completion <bash\|zsh\|fish>` | Shell tab-completion script (needs global install) |

Migration change statuses: **safe** (additive — applied together on confirmation),
**warning** (destructive/slow — each confirmed individually, no "apply all"),
**manual** (e.g. type change — shown with guidance, you do it), **undocumented**
(in DB but not in schema — shown for awareness).

Authentication in CI: set `TGCLOUD_TOKEN` env var (never written to disk). Resolution
order: env var → `.tgcloud/credentials` → error. The CLI never prompts mid-command;
commands exit non-zero on failure, so they compose in scripts.

## Day-to-day workflow

```bash
npx tgcloud run handlers/message '{ chat: { id: 1 }, text: "hi" }'   # 1. test locally
# edit schema.js if needed
npx tgcloud push      # 2. deploy code (atomic; reports pending DB changes, applies none)
npx tgcloud migrate   # 3. apply DB changes if schema changed
npx tgcloud status    # 4. sanity-check local vs cloud
```

- Tight loop: edit → `run` with a JSON5 payload → `push` when happy.
- After `push` of a changed `schema.js`, you'll be told the DB is out of sync — that's
  expected; run `migrate`.
- **Conflicts**: the cloud has a monotonically increasing revision; `push` sends yours
  and is rejected if the cloud moved ahead (another machine/teammate deployed). Then:
  `fetch` (re-check) / `pull` (adopt cloud state) / `push --force` (overwrite — dangerous).
- **Webhook drift**: deploying/removing handlers can leave the webhook out of sync
  (`status` flags it); someone calling `setWebhook` with the raw bot token breaks it too —
  repair with `npx tgcloud webhook sync`. "In sync" = webhook points at the platform and
  `allowed_updates` match your deployed handlers.
- **BotFather on the go**: the same project is fully manageable from BotFather's phone
  UI — create/edit/test-run handlers (with console output in chat), edit the `lib/`
  modules, edit `schema.js` (review + apply), grab the CLI token. Start on your phone,
  `npx tgcloud pull` on your laptop — it's one and the same cloud project.
- **Building with AI** is a first-class path: the scaffold ships `AGENTS.md` +
  `docs/tgcloud-sdk.md` so coding agents work in the project. If you generate a lot of
  code here, keep `AGENTS.md` updated as the bot grows.

## Key limits & constraints (as of 2026-09)

| Thing | Limit / fact |
|---|---|
| Language | JavaScript only (ES modules). No TypeScript build step, no npm, no native modules, no WASM, no filesystem |
| Runtime | One V8 isolate per invocation, close to Telegram's systems; code + DB per bot |
| HTTP response | Text-only, **30 MB** total per response (was 32 MB at launch) |
| File upload (Bot API) | **50 MB**/file, **10 MB** for photos |
| File download (`getFile`) | **20 MB** |
| SQLite | Foreign keys OFF (`PRAGMA foreign_keys` disabled); batch inserts bounded by SQLite variable limit |
| Scheduling | **No documented timers/cron** — the runtime is event-driven (woken by updates). For periodic work you need an external trigger (e.g. another service hitting a public endpoint, or a second bot pinging this one) |
| Secrets | **No documented secrets manager** — third-party API keys (e.g. an LLM provider) typically live in your code/modules, which is a real consideration for sensitive bots |
| DB export | **No documented mechanism** to export the bot's database — plan for lock-in if data portability matters |
| Pricing/quotas | Not documented on the platform page as of 2026-09 |
| Privacy | Bot conversations were never E2E-encrypted; with Serverless, app logic + stored data also live on Telegram's infrastructure |

## Gotchas & design notes

- **No cron ⇒ rethink "reminders/periodic jobs".** Serverless wakes on updates. Patterns:
  store `next_fire_at` in SQLite and act on the next relevant interaction; have a
  lightweight external scheduler POST a webhook/trigger; or keep a tiny always-on
  companion service.
- **Large backups/media**: you can *download* up to 20 MB per `file_id` and
  *re-upload* up to 50 MB, but there is **no persistent filesystem** — anything you save
  long-term must go back into Telegram (e.g. a private backup channel: `api.sendDocument`
  to your own channel chat) or to an external HTTP service. Storing `file_id`s in SQLite
  is the natural primary design; a `file_id` from your bot's own media stays resolvable.
- **No FKs ⇒ write delete order and orphan sweeps deliberately** (children first).
- **`await` everything DB** — the single most common bug when porting from Drizzle/Node.
- **Don't hardcode the bot API token** — you don't have it and don't need it; `api` is
  pre-authenticated. The CLI token (`app<id>:<secret>`) is for the CLI only and stays in
  `.tgcloud/`.
- **Idempotency**: updates can be retried; keep handler logic safe to re-run where it
  matters (use `update_id` from `ctx.update` for dedupe if needed).
- **Mini App backends** are the flagship use case: Mini Apps cannot hold the bot token
  client-side, so a serverless backend gives them a protected execution layer + a direct
  authenticated bridge. Bot API 10.2 (2026-07-20+) enforces Mini App origin
  protection — keep your Mini App domain stable.
- **Rich Messages + streaming** (Bot API 10.1–10.3) pair perfectly with `fetch` streaming:
  `sendRichMessage` / `sendRichMessageDraft` let you stream formatted AI answers —
  see `references/examples.md`.
- **Ephemeral messages** (Bot API 10.2): user-specific visible messages,
  `ephemeral_message_parameters` on send*, `Message.receiver_user`,
  `Message.ephemeral_message_id`, `editEphemeralMessage*`, `deleteEphemeralMessage`.
- The platform is new: **re-check the official page** (https://core.telegram.org/bots/serverless)
  for limits/fees before production commitments.

## References (read when you need depth)

- `references/sdk.md` — complete SDK API reference (db, api, files, fetch, console).
- `references/cli.md` — every `tgcloud` command with flags and CI notes.
- `references/limits-gotchas.md` — limits table, launch timeline, privacy/lock-in notes.
- `references/examples.md` — worked examples: counter bot, todo bot, streaming AI
  chatbot, backup bot skeleton (this repo's use case), Mini App backend.
