# tgcloud SDK — complete reference

Everything a serverless module can import from `sdk`: the database (`db`), the Telegram
Bot API (`api`), outbound HTTP (`fetch`), file handling (`InputFile`,
`api.getFileContent`, `api.getFileStream`), and `console` logging.

> Source: official scaffold template `docs/tgcloud-sdk.md` (@tgcloud/cli 0.1.2) +
> official docs page (snapshot 2026-09-05). The files section came from the latter.

## `sdk` at a glance

```js
import { db, api, fetch, InputFile, BotApiError } from 'sdk'
// or from submodules:
import { table, integer, text, eq, sql } from 'sdk/db'
import { api } from 'sdk/api'
import { fetch } from 'sdk/fetch'
```

- **`db`** — SQLite query builder + schema DSL (Drizzle-like).
- **`api`** — the whole Telegram Bot API: `api.sendMessage({…})`.
- **`fetch`** — outbound HTTP, web-`fetch`-like.
- **`InputFile`** — file bytes + name + optional MIME type, for uploads to `api` or `fetch`.
- **`console`** — global, no import; output surfaces in `npx tgcloud run`.

## Rules that bite (runtime is not stock Node)

- **Import by bare module name** — `from 'schema'`, `from 'lib/cart'`, `from 'sdk/db'`.
  Never a relative path or `.js` extension.
- **Every DB call is async — always `await`.** `.all()/.get()/.values()/.run()`,
  `db.$count()`, raw `db.run/all/get/values` all return Promises.
- **No foreign keys** — `.references()` / `foreignKey()` throw at declaration.
- **Drops only via `.deprecated('reason')`**; type changes are manual (`db.run(...)`).
- **Handler `export default (payload, ctx)`** — payload = the update-type object
  (`Message` for `handlers/message`); full `Update` on `ctx.update`.
- No npm packages, no filesystem, no Node `Buffer` (use `Uint8Array`).

# Database (`db`)

## Importing

```js
import { table, integer, text, eq, sql } from 'sdk/db'   // named imports
import db, { sql } from 'sdk/db'                          // or namespace + names
```

## Schema definition (`schema.js`)

Tables are **named exports**. `table(name, columns, extras?)` builds a descriptor at
load time (no DB access); the platform discovers exported tables when `schema.js` is
deployed and migrates on `npx tgcloud migrate`.

```js
import { table, integer, text, boolean, json, index, check, sql } from 'sdk/db'

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
}))

export const todos = table('todos', {
  id:       integer('id').primaryKey({ autoIncrement: true }),
  userId:   integer('user_id').notNull(),   // logical link to users.id — NOT enforced
  text:     text('text').notNull(),
  done:     boolean('done').default(false),
  priority: integer('priority').default(0),
}, (t) => ({
  userDoneIdx:   index('idx_todos_user_done').on(t.userId, t.done),
  priorityCheck: check('priority_check', sql`${t.priority} >= 0`),
}))
```

`extras` callback: `(t) => ({…})` where `t` exposes column refs (`t.userId`).

### Column types

| factory     | SQLite  | notes                                    |
|-------------|---------|------------------------------------------|
| `text()`    | TEXT    |                                          |
| `integer()` | INTEGER |                                          |
| `real()`    | REAL    | alias `float()`                          |
| `numeric()` | NUMERIC |                                          |
| `blob()`    | BLOB    | takes/returns `Uint8Array`               |
| `boolean()` | INTEGER | stored 0/1, read as `true`/`false`       |
| `json()`    | TEXT    | auto `JSON.stringify` / `JSON.parse`     |

Signature: `factory(name?, opts?)` — name optional (defaults to the JS key).
`opts.mode` sets runtime value conversion:

| mode           | stored as          | JS value              |
|----------------|--------------------|-----------------------|
| `boolean`      | INTEGER 0/1        | `boolean`             |
| `json`         | TEXT (JSON)        | any object/array      |
| `timestamp`    | INTEGER (unix s)   | `Date`                |
| `timestamp_ms` | INTEGER (unix ms)  | `Date`                |
| `bytes`        | BLOB               | `Uint8Array`          |

`boolean()` / `json()` are sugar over `integer(name, { mode: 'boolean' })` /
`text(name, { mode: 'json' })`. Mode controls **encoding, not storage**:
`blob('col', { mode: 'json' })` stores JSON in a BLOB column.

### Column modifiers (chainable)

```js
integer('id').primaryKey()
integer('id').primaryKey({ autoIncrement: true })
text('name').notNull()
text('tg').unique()
text('lang').default('en')
integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`)
text('slug').generatedAlwaysAs(sql`lower(name)`, { mode: 'stored' })  // or 'virtual'
text('name').constraint('COLLATE NOCASE')        // arbitrary column-level DDL
text('email').deprecated('replaced by login')    // marks for drop; terminal
```

- `.default()` on a `json()` column encodes the value automatically; a `sql` default is
  passed through verbatim.
- `.deprecated()` is **terminal** — nothing after it.

### Table-level constraints & indexes (in the extras callback)

```js
table('t', { /* columns */ }, (t) => ({
  pk:    primaryKey({ columns: [t.a, t.b] }),
  uq:    unique('uq_email').on(t.email),
  chk:   check('chk_done', sql`${t.done} in (0, 1)`),
  idx:   index('idx_name').on(t.col),
  uidx:  uniqueIndex('uidx_email').on(t.email),
  lower: index('idx_lower').on(sql`lower(${t.email})`),          // expression index
  active: index('idx_active').on(t.userId).where(sql`done = 0`), // partial index
}))   // foreignKey(...) is NOT supported — it throws
```

Table modifiers (chained after `table(...)`): `.strict()`, `.withoutRowid()`,
`.constraint('CHECK (x > 0)', 'chk_x')`, `.deprecated('reason')`.

### No foreign keys — details

The runtime runs with `PRAGMA foreign_keys` **off**. Declared FKs would be silently
inert (no cascades, no orphan protection), so `.references()` and table-level
`foreignKey()` **throw when declared** and a schema using them won't deploy.
`REFERENCES` / `FOREIGN KEY` smuggled via `.constraint()` or raw `db.run('CREATE TABLE …')`
also stay inert. Enforce integrity in application code:
insert parents before children, delete children before parents, sweep orphans with
`LEFT JOIN … WHERE parent.id IS NULL`.

## Query builder

### select

`select(projection?)` → `.from(table)` → builder. No projection = `SELECT *`.

```js
await db.select().from(todos).all()                          // all rows
await db.select().from(todos).where(eq(todos.id, 1)).get()   // first row or null
await db.select().from(todos).values()                       // rows as value arrays
await db.$count(todos)                                       // COUNT(*)
await db.$count(todos, eq(todos.done, false))

await db.select().from(todos)
  .where(and(eq(todos.userId, uid), eq(todos.done, false)))
  .orderBy(desc(todos.priority), asc(todos.id))
  .limit(10).offset(20)
  .all()

// custom projection: { alias: colRef | sqlExpr | aggregate }
await db.select({ id: todos.id, title: todos.text, n: count() })
  .from(todos).groupBy(todos.userId).having(sql`count(*) > ${1}`).all()
```

Chain: `.where()` `.orderBy()` `.limit()` `.offset()` `.groupBy()` `.having()`
`.distinct()`. Terminals: `.all()` `.get()` `.values()`. The builder is **awaitable** —
`await db.select().from(todos)` runs `.all()` for you, so `.all()` is optional on
select/insert/update/delete.

### insert / update / delete

```js
await db.insert(todos).values({ userId: 1, text: 'Buy milk' }).run()
await db.insert(todos).values([{ text: 'A' }, { text: 'B' }]).run()   // batch
await db.insert(todos).values({ text: 'X' }).returning().run()        // RETURNING *
await db.insert(users).values({ tgId: 42, name: 'Ann' })
  .onConflictDoNothing({ target: users.tgId }).run()
await db.insert(users).values({ tgId: 42, name: 'Ann' })
  .onConflictDoUpdate({ target: users.tgId, set: { name: 'Ann' } }).run()

await db.update(todos).set({ done: true }).where(eq(todos.id, 1)).run()   // .set() required
await db.delete(todos).where(eq(todos.id, 1)).run()
```

- A plain insert/update/delete resolves to `[]` — no insert id or row count. Add
  `.returning()` (→ `RETURNING *`) or `.returning({ id: todos.id })` to get rows back
  (converted, since bound to the table).
- Batch insert is one statement — capped by SQLite's variable limit (`rows × columns`);
  chunk large batches yourself.
- Writes resolve to a run result when using raw `db.run`: `{ rowsAffected, lastInsertRowid, rows }`.

### Raw SQL — `db.run` / `db.all` / `db.get` / `db.values`

Mode is chosen by method: `run` = write/exec, `all` = all rows (keyed by column name),
`get` = first row or `null`, `values` = rows as positional arrays (use it when a join
projects two columns with the same name — `db.all` would let the second overwrite the
first).

```js
await db.run('UPDATE todos SET done = 1 WHERE id = :id', { ':id': 5 })
// → { rowsAffected: 3, lastInsertRowid: 0, rows: [] }
await db.all(sql`SELECT * FROM todos WHERE done = ${false}`)
await db.get(sql`SELECT count(*) AS c FROM todos`)
await db.values(sql`SELECT id, text FROM todos`)   // → [[1, 'buy milk']]
```

Each takes a ``sql`…``` `` object or a `(queryString, params)` pair. Raw rows come back
**without mode conversion** (boolean → 0/1, json → string, timestamp → number).
`db.raw.read(sql, params)` / `db.raw.write(sql, params)` are the low-level hatch;
prefer `run`/`all`/`get`.

## `sql` tagged template

```js
sql`count = ${n}`                 // count = :p1        (value → bound parameter)
sql`${todos.priority} > ${min}`   // priority > :p2     (column → identifier)
sql`WHERE ${cond}`                // nested sql spliced in
sql.raw('datetime("now")')        // literal, no parameters
```

In DDL contexts (DEFAULT / CHECK / GENERATED) parameters don't work — use literal SQL.

## Operators

```js
import {
  eq, ne, gt, gte, lt, lte,
  like, notLike,
  isNull, isNotNull, and, or, not,
  between, notBetween, inArray, notInArray,
  count, sum, avg, min, max,
  asc, desc,
} from 'sdk/db'
```

`.where(e1, e2)` with multiple args ≡ `and(e1, e2)`. A comparison's second argument may
be a value (default), another column, or a `sql` fragment — `eq(a.x, b.y)` works.
Aggregates are `sql` fragments for `.select({ … })` projections.

## Migrations (summary)

- **Adding** tables/columns/indexes — **safe**: applied together on `migrate` confirmation.
- **Dropping** — only via `.deprecated()`; shown as **warning**, confirmed individually.
- **Changing a column's type** — **manual**: `migrate` shows guidance; do it with
  `db.run(...)` (new column → copy → deprecate old).
- Objects in the DB but not in the schema — **undocumented**: shown for awareness only.
- `npx tgcloud push` reports pending DB changes but **never applies them**.
- Flags: `--dry-run`, `--safe`, `--yes`, `--local` (see `cli.md`).

# Telegram Bot API (`api`)

`import { api, BotApiError } from 'sdk'`. Call any method as `api.<method>(params)` —
a Proxy dispatches the name, so every current and future Bot API method works with no
SDK update.

```js
const me = await api.getMe()                            // → unwrapped `result`
await api.sendMessage({ chat_id: id, text: 'Hello!' })
await api.editMessageText({ chat_id, message_id, text: 'Updated' })
await api.answerCallbackQuery({ callback_query_id, text: 'Done' })
```

- **Envelope unwrapped**: on `{ ok: true }` resolves to `result` directly.
- **Params**: one object, Bot API snake_case names (`chat_id`, `message_id`, …).
- **Failures throw `BotApiError`** with `.code` (Bot API `error_code`: 400/403/429/…),
  `.description` (human-readable), `.method`, `.parameters` (e.g. `retry_after` on 429,
  `migrate_to_chat_id`).

```js
try {
  await api.deleteMessage({ chat_id, message_id });
} catch (e) {
  if (e.code !== 400) throw e;   // 400 = already gone; anything else is a real error
}
```

Notable Bot API 10.x additions usable via `api` (all params per official API docs):

- **Rich Messages** (10.1, extended 10.2/10.3): `sendRichMessage`,
  `sendRichMessageDraft` (stream partial rich messages — the AI-streaming pattern),
  `rich_message` on `editMessageText`, `InputRichMessage` (exactly one of `html` /
  `markdown` / `blocks`), block builders (lists, tables, details, collages, slideshows,
  media, thinking blocks, …), `RichBlockDocument`, `tg://document?id=` links.
- **Ephemeral messages** (10.2): `ephemeral_message_parameters` on send* methods,
  `Message.receiver_user`, `Message.ephemeral_message_id`,
  `ReplyParameters.ephemeral_message_id`, `editEphemeralMessageText/Media/Caption/
  ReplyMarkup`, `deleteEphemeralMessage`, `BotCommand.is_ephemeral`.
- **Communities** (10.2): `Community`, `Message.community_chat_added /
  community_chat_removed`, `ChatFullInfo.community`.
- **Subscriptions** (10.2): `Update.subscription` / `BotSubscriptionUpdated` — a
  serverless handler `handlers/subscription.js` can react to payment-subscription
  changes.
- **Stopped generation** (10.3): `Update.stopped_message_generation` /
  `MessageGenerationStopped`.

# Files

Work with file **bytes**, not just `file_id`s (added after launch; present in the
2026-09 docs).

## Upload — `InputFile`

A plain holder: bytes (`Uint8Array`), filename, optional MIME `type`. Pass wherever a
Bot API method expects a file — including nested inside params (albums).

```js
import { api, InputFile } from 'sdk'

const doc = new InputFile(bytes, 'report.pdf', { type: 'application/pdf' })
await api.sendDocument({ chat_id, document: doc })

await api.sendMediaGroup({ chat_id, media: [
  { type: 'photo', media: new InputFile(a, 'a.jpg', { type: 'image/jpeg' }) },
  { type: 'photo', media: new InputFile(b, 'b.jpg', { type: 'image/jpeg' }) },
]})
```

Bytes are streamed to Telegram as a real multipart upload (not buffered whole).
Bot API upload limits apply: **50 MB** per file, **10 MB** for a photo.

## Download — by `file_id`

Two helpers; the platform resolves the `file_id` for you (runs `getFile` and fetches
the content behind the scenes) — you never touch the bot token or a download URL.

```js
const bytes = await api.getFileContent(file_id)          // → Uint8Array (whole file)

const { file, body } = await api.getFileStream(file_id)  // stream instead
file.file_unique_id                                      // the getFile info
for await (const chunk of body) { /* Uint8Array */ }     // or buffer: await body.bytes()
```

`getFile` downloads are capped at **20 MB** — the ceiling for both helpers.

# HTTP (`fetch`)

```js
import { fetch } from 'sdk'

const res = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Pavel' }),
})
if (!res.ok) throw new Error(res.statusText)
const data = await res.json()
```

Response: `res.status`, `res.statusText`, `res.ok` (200–299), `res.url` (final URL
after redirects), `res.headers` (`.get()/.has()/.keys()/.entries()`), body readers
`await res.json()` / `await res.text()`, or stream: `for await (const chunk of res.body)`
— **this is how you consume SSE / token-by-token AI output**.

Body helpers set the matching `Content-Type`:

```js
await fetch(url, { method: 'POST', body: fetch.body.json({ a: 1 }) })  // application/json
await fetch(url, { method: 'POST', body: fetch.body.form({ a: 1 }) })  // x-www-form-urlencoded
await fetch(url, { method: 'POST', body: fetch.body.text('hi') })      // text/plain
```

Request bodies may also carry **files and raw bytes**: pass a `Uint8Array`, an
`InputFile`, or a `FormData` (`multipart/form-data`) as `body` — streamed out:

```js
import { fetch, FormData, InputFile } from 'sdk'

const form = new FormData()
form.append('file', new InputFile(bytes, 'photo.jpg', { type: 'image/jpeg' }))
form.append('caption', 'from my bot')
const res = await fetch('https://api.example.com/upload', { method: 'POST', body: form })
```

Behavior: a response body can be read **once** (second read throws
`TypeError: body used already`; check `res.bodyUsed`). Redirects are followed
automatically. A 404 (any HTTP status) resolves normally with `res.ok === false`; only
real network errors (bad host, invalid URL) reject.

**Constraints: the response is text-only (binary payloads not supported) and the total
response is capped at 30 MB** — streaming with `res.body` processes a large body
incrementally but does not raise the limit.

# Logging (`console`)

Standard global `console` — nothing to import. Output is captured and shown by
`npx tgcloud run`, making it the primary debugging tool.

```js
console.log('processing', { chatId: id })   // log / debug — plain
console.info('started')                     // info — blue
console.warn('rate limited')                // warn — yellow
console.error(err)                          // error — red, with stack trace
```

Each line is tagged with its `[file:line]` origin. `console.error` and
`console.trace` append a full stack; `console.warn` does not.
