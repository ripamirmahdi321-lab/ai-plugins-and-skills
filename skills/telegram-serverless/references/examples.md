# Worked examples

Complete, deployable examples for tgcloud projects. All follow the project layout:
`schema.js` (root) + `handlers/<update_type>.js` (flat) + `lib/` (shared). Remember:
bare-name imports, `await` every DB call, `api` throws `BotApiError`, drops via
`.deprecated()`.

Test a handler before deploying:
```bash
npx tgcloud run handlers/message '{ chat: { id: 1 }, text: "hello" }'
```
Deploy: `npx tgcloud push` (then `npx tgcloud migrate` if the schema changed).

---

## 1. Counter bot (official quick demo)

Replies to every message and remembers how many it has seen from each chat.

`schema.js`
```js
import { table, integer } from 'sdk/db';

export const counters = table('counters', {
  chatId: integer('chat_id').primaryKey(),
  seen:   integer('seen').notNull().default(0),
});
```

`handlers/message.js`
```js
import { api, db } from 'sdk';
import { counters } from 'schema';
import { sql } from 'sdk/db';

export default async function (message) {
  const chatId = message.chat.id;

  // Insert, or bump if this chat already has one — row back via .returning().
  const [row] = await db.insert(counters)
    .values({ chatId, seen: 1 })
    .onConflictDoUpdate({
      target: counters.chatId,
      set: { seen: sql`${counters.seen} + 1` },
    })
    .returning()
    .run();

  await api.sendMessage({
    chat_id: chatId,
    text: `Hello! I've seen ${row.seen} message(s) from you.`,
  });
}
```

```bash
npx tgcloud push      # upload modules
npx tgcloud migrate   # create the counters table
```

---

## 2. To-do list bot with inline buttons

Adds an item when the user sends text, toggles with buttons, lists with `/list`.

`schema.js` — note that DDL defaults (like `DEFAULT (unixepoch())`) must be literal SQL via the `sql` tag, not JS values:
```js
import { table, integer, text, boolean, index, sql } from 'sdk/db';

export const todos = table('todos', {
  id:      integer('id').primaryKey({ autoIncrement: true }),
  tgId:    integer('tg_id').notNull(),
  text:    text('text').notNull(),
  done:    boolean('done').default(false),
  created: integer('created_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
}, (t) => ({
  userDoneIdx: index('idx_todos_user_done').on(t.tgId, t.done),
}));
```

`handlers/message.js`
```js
import { api, db } from 'sdk';
import { todos } from 'schema';
import { eq } from 'sdk/db';

export default async function (message) {
  const text = message.text;
  if (text == null) return;                       // media etc. — ignored
  const tgId = message.from?.id ?? message.chat.id;
  const chatId = message.chat.id;

  if (text.startsWith('/list')) {
    const rows = await db.select()
      .from(todos)
      .where(eq(todos.tgId, tgId))
      .orderBy(todos.id)
      .all();
    const lines = rows.length
      ? rows.map(r => `${r.done ? '✅' : '⬜'} ${r.text}`).join('\n')
      : '(nothing yet)';
    await api.sendMessage({ chat_id: chatId, text: `Your to-dos:\n${lines}` });
    return;
  }

  if (text.startsWith('/')) return;               // unknown command — ignore
  const [row] = await db.insert(todos)
    .values({ tgId, text })
    .returning()
    .run();
  const kb = {
    inline_keyboard: [[
      { text: '⬜ Mark done', callback_data: `todo:toggle:${row.id}` },
    ]],
  };
  await api.sendMessage({
    chat_id: chatId,
    text: `Added #${row.id}: ${row.text}`,
    reply_markup: kb,
  });
}
```

`handlers/callback_query.js`
```js
import { api } from 'sdk';
import { todos } from 'schema';
import { db } from 'sdk';
import { eq } from 'sdk/db';

export default async function (callbackQuery) {
  const [ , action, idStr ] = callbackQuery.data.split(':');
  const chatId = callbackQuery.message.chat.id;

  if (action === 'toggle') {
    const id = Number(idStr);
    // Update and get the changed row back in the same statement via .returning().
    const [row] = await db.update(todos)
      .set({ done: true })
      .where(eq(todos.id, id))
      .returning()
      .run();
    if (row) {
      await api.editMessageText({
        chat_id: chatId,
        message_id: callbackQuery.message.message_id,
        text: `#${row.id}: ${row.text} — done ✅`,
        reply_markup: { inline_keyboard: [[
          { text: '↩️ Reopen', callback_data: `todo:reopen:${row.id}` },
        ]] },
      });
    }
  }
  await api.answerCallbackQuery({ callback_query_id: callbackQuery.id }); // always answer
}
```
(Keep a `todo:reopen` branch symmetric to `todo:toggle` in real code.)

Notes:
- `answerCallbackQuery` should always be called, otherwise the client spinner hangs.
- Ownership check: in production verify `row.tgId` matches the caller before
  toggling (app-level integrity — there are no FKs or row-level protection).
- `/list` output can exceed 4096 chars — paginate (`limit`/`offset`) for real use.

---

## 3. Streaming AI chatbot (the flagship pattern)

Bot API 10.1+ added draft streaming precisely for LLM answers. In serverless this is
`fetch` (streaming) + `api.sendRichMessageDraft` (animated partial updates) + final
`api.sendRichMessage`.

Key API facts (Bot API 10.1–10.3):
- `sendRichMessageDraft`: **private chats only** (Integer `chat_id`); params:
  `chat_id`, `draft_id` (Integer, **non-zero**; same id per turn → animated updates),
  `rich_message` (InputRichMessage — exactly one of `html` / `markdown` / `blocks`;
  new-file upload not supported in drafts), `can_stop` (shows a stop button → you
  receive a `stopped_message_generation` update), `keep_on_stop`.
- Drafts are **ephemeral (~30 s preview, not persisted)** — you **must finalize** by
  sending the full message with `sendRichMessage`.
- Fallback for older clients: `sendMessage` + `editMessageText` loop.

`schema.js`
```js
import { table, integer, text } from 'sdk/db';

export const convo = table('convo', {
  chatId:  integer('chat_id').primaryKey(),
  history: text('history').notNull().default('[]'),   // JSON array of {role, content}
});
```

`handlers/message.js`
```js
import { api, fetch, db, BotApiError } from 'sdk';
import { convo } from 'schema';
import { eq, sql } from 'sdk/db';
import { LLM_URL, LLM_KEY } from 'lib/llm';           // ⚠️ keys live in code — see limits doc

const DRAFT_ID = 1;                                    // fixed per chat+turn here; vary for parallel turns

export default async function (message) {
  if (!message.text || message.text.startsWith('/')) return;
  const chatId = message.chat.id;

  // Load history for this chat, seeding an empty one on first contact.
  let row = await db.select().from(convo).where(eq(convo.chatId, chatId)).get();
  if (!row) {
    [row] = await db.insert(convo).values({ chatId, history: '[]' }).returning().run();
  }
  let history = JSON.parse(row.history);
  history.push({ role: 'user', content: message.text });
  if (history.length > 20) history = history.slice(-20);

  // Open the draft with a "thinking" state.
  await api.sendRichMessageDraft({
    chat_id: chatId,
    draft_id: DRAFT_ID,
    rich_message: { html: '' },          // empty html → "Thinking…" placeholder
    can_stop: true,
  }).catch(e => { if (!(e instanceof BotApiError)) throw e; });

  let answer = '';
  const res = await fetch(LLM_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${LLM_KEY}` },
    body: JSON.stringify({ messages: history, stream: true }),
  });
  if (!res.ok) throw new Error(`LLM ${res.status} ${res.statusText}`);

  // Stream the SSE body — text-only, ≤30 MB total (fine for a normal answer).
  let buf = '';
  for await (const chunk of res.body) {
    buf += typeof chunk === 'string' ? chunk : new TextDecoder().decode(chunk);
    let i;
    while ((i = buf.indexOf('\n')) >= 0) {
      const line = buf.slice(0, i).trim();
      buf = buf.slice(i + 1);
      if (!line.startsWith('data:')) continue;
      const data = line.slice(5).trim();
      if (data === '[DONE]') break;
      try {
        const delta = JSON.parse(data).choices?.[0]?.delta?.content ?? '';
        if (!delta) continue;
        answer += delta;
        await api.sendRichMessageDraft({
          chat_id: chatId,
          draft_id: DRAFT_ID,
          rich_message: { html: answer },   // or markdown / blocks
        }).catch(() => {});                  // animated update; drop errors between chunks
      } catch { /* partial JSON line — ignore */ }
    }
  }

  // FINALIZE — drafts don't persist.
  history.push({ role: 'assistant', content: answer });
  await api.sendRichMessage({ chat_id: chatId, rich_message: { html: answer } });
  await db.update(convo).set({ history: JSON.stringify(history) }).where(eq(convo.chatId, chatId)).run();
}
```

`handlers/stopped_message_generation.js` (scaffold with `npx tgcloud add handlers/stopped_message_generation` if the platform lists the type)
```js
// User pressed the stop button during a draft.
export default async function (stopped) {
  // e.g. notify, cancel in-flight work (no timers — the in-flight stream in the
  // previous invocation simply ends), or persist a partial answer.
}
```

Real-world notes:
- One streaming turn per chat at a time (fixed `draft_id` collides otherwise); vary
  `draft_id` or guard with a per-chat state row if you support parallel turns.
- The 30 MB response cap covers the whole LLM response body — normal for text answers.
- If your LLM endpoint returns binary or >30 MB, you need an intermediate relay.
- Keep `lib/llm.js` as the single place for endpoint + key (rotate = re-deploy).

---

## 4. Chat-backup bot skeleton (this repo's use case)

Design constraints that shape the whole thing: **no filesystem** (metadata + `file_id`s
in SQLite; media mirrored into a private channel), **20 MB download / 50 MB upload**
caps, **no cron** (backup runs while the bot is active; finalization is event-triggered).

`schema.js`
```js
import { table, integer, text, boolean, index, sql } from 'sdk/db';

export const backups = table('backups', {
  id:          integer('id').primaryKey({ autoIncrement: true }),
  chatId:      integer('chat_id').notNull(),
  ownerId:     integer('owner_id').notNull(),
  status:      text('status').notNull().default('running'),   // running|paused|done
  lastMsgId:   integer('last_message_id').default(0),
  startedAt:   integer('started_at', { mode: 'timestamp' }).default(sql`(unixepoch())`),
}, (t) => ({
  chatIdx: index('idx_backups_chat').on(t.chatId).where(sql`status = 'running'`),
}));

export const savedMessages = table('saved_messages', {
  id:        integer('id').primaryKey({ autoIncrement: true }),
  backupId:  integer('backup_id').notNull(),
  msgId:     integer('message_id').notNull(),
  chatId:    integer('chat_id').notNull(),
  date:      integer('date', { mode: 'timestamp' }),
  senderId:  integer('sender_id'),
  text:      text('text'),
  mediaType: text('media_type'),     // photo|video|document|audio|voice|...
  fileId:    text('file_id'),        // resolvable by this bot later
  mirrored:  boolean('mirrored').default(false),
}, (t) => ({
  chatMsgIdx: index('idx_saved_chat_msg').on(t.chatId, t.msgId),
}));
```

`handlers/message.js`
```js
import { api, db } from 'sdk';
import { eq } from 'sdk/db';
import { backups, savedMessages } from 'schema';
import { startKeyboard } from 'lib/reply';

export default async function (message, ctx) {
  // 1) Interactive: /backup in a chat the bot can read.
  const text = message.text;
  if (text === '/backup') {
    const [b] = await db.insert(backups)
      .values({ chatId: message.chat.id, ownerId: message.from?.id ?? 0 })
      .returning().run();
    await api.sendMessage({
      chat_id: message.chat.id,
      text: `Backup started for this chat (#${b.id}). I'll store every message that passes through me.`,
      reply_markup: startKeyboard(b.id),
    });
    return;
  }

  // 2) Passive: if this chat has a running backup, record the message.
  const [b] = await db.select()
    .from(backups)
    .where(eq(backups.chatId, message.chat.id))
    .get();
  if (!b || b.status !== 'running' || message.message_id <= b.lastMsgId) return;

  const mediaType = message.photo ? 'photo'
    : message.video ? 'video'
    : message.audio ? 'audio'
    : message.voice ? 'voice'
    : message.document ? 'document'
    : null;
  const fileId = message.photo?.[message.photo.length - 1]?.file_id
    ?? message.video?.file_id ?? message.audio?.file_id
    ?? message.voice?.file_id ?? message.document?.file_id ?? null;

  await db.insert(savedMessages).values({
    backupId: b.id,
    msgId: message.message_id,
    chatId: message.chat.id,
    date: new Date(message.date * 1000),
    senderId: message.from?.id ?? null,
    text: message.text ?? message.caption ?? null,
    mediaType,
    fileId,
  }).run();

  await db.update(backups).set({ lastMsgId: message.message_id }).where(eq(backups.id, b.id)).run();
  // ⚠️ Mirror media into your private backup channel here (see lib/backup.js sketch below).
}
```

`handlers/callback_query.js`
```js
import { api, db } from 'sdk';
import { eq } from 'sdk/db';
import { backups } from 'schema';

export default async function (q) {
  const [ , action, idStr ] = q.data.split(':');
  const id = Number(idStr);
  const [b] = await db.select().from(backups).where(eq(backups.id, id)).get();
  if (!b) return api.answerCallbackQuery({ callback_query_id: q.id, text: 'Unknown backup' });

  if (action === 'stop') {
    await db.update(backups).set({ status: 'done' }).where(eq(backups.id, id)).run();
    await api.editMessageText({
      chat_id: q.message.chat.id,
      message_id: q.message.message_id,
      text: `Backup #${id} stopped.`,
      reply_markup: null,
    });
  }
  await api.answerCallbackQuery({ callback_query_id: q.id });
}
```

`lib/reply.js`
```js
// Shared keyboard builders — imported by bare name from handlers.
export function startKeyboard(backupId) {
  return {
    inline_keyboard: [[
      { text: '⏹ Stop backup', callback_data: `backup:stop:${backupId}` },
    ]],
  };
}
```

`lib/backup.js` (sketch — the media-mirroring piece)
```js
import { api, InputFile } from 'sdk';

const BACKUP_CHANNEL = '@your_private_backup_channel';   // bot must be admin

// Copy a message's media from the source chat into the backup channel.
export async function mirrorMedia(message, meta) {
  // Two ways to move media:
  //  (a) Re-send by file_id (no download needed!) — cheapest, no size cap beyond
  //      what the bot can send (50 MB upload / 10 MB photo Bot API limits):
  if (message.photo) {
    const f = message.photo[message.photo.length - 1];
    await api.sendPhoto({ chat_id: BACKUP_CHANNEL, photo: f.file_id, caption: meta.caption });
    return true;
  }
  if (message.document) {
    await api.sendDocument({ chat_id: BACKUP_CHANNEL, document: message.document.file_id });
    return true;
  }
  //  (b) Download (≤20 MB) and re-upload — needed when you must transform bytes:
  // const bytes = await api.getFileContent(message.video.file_id);
  // await api.sendVideo({ chat_id: BACKUP_CHANNEL, video: new InputFile(bytes, 'clip.mp4', { type: 'video/mp4' }) });
  // catch BotApiError 400 (file too big for getFile) → store file_id only and mark mirrored=false.
  return false;
}
```

Operational notes (be honest about these with the user):
- **A bot only sees messages it is a member of / that are forwarded or sent to it.**
  Backing up an arbitrary group's history requires the bot to be added to that group;
  full history of a private chat is not obtainable through the Bot API.
- **No timers**: "finalize the backup after N minutes" needs an external trigger or a
  user action (`/backup stop`); there's no documented cron.
- Big files: `getFile` caps at 20 MB; re-send-by-`file_id` avoids the download entirely
  whenever possible.
- Export path (counter the platform's lack of DB export): a `/export` command that
  streams `saved_messages` as a JSON document to the owner chat (and `file_id`s are
  the durable handle on the media itself).

---

## 5. Mini App backend (pattern notes)

Mini Apps run client-side inside the Telegram app and **cannot hold the bot token** —
users can inspect downloaded code. A serverless backend gives them a protected,
pre-authenticated execution layer sitting next to the platform.

Pattern:
- `handlers/` for the update types that drive your Mini App (e.g. `message`,
  `callback_query`, `pre_checkout_query` for payments).
- All state in `db`; serve dynamic data through the platform's authenticated bridge
  (Mini App init data verifies the user) — check the current official docs for the
  exact Mini App ↔ serverless request surface; the platform docs describe it as a
  "direct bridge between authenticated Telegram activity and application logic".
- Bot API 10.2 (2026-07-20) enforces Mini App **origin protection**: calls from
  origins other than your Mini App domain are rejected (opt-out via BotFather is
  discouraged). Keep your Mini App domain stable; don't build your app assuming
  cross-origin backend calls.
- Good fits: leaderboards, quizzes, stores, per-user dashboards, subscription gates
  (`Update.subscription` / `BotSubscriptionUpdated` → `handlers/subscription.js`).

---

## Testing & deploy checklist (per feature)

1. `npx tgcloud run handlers/<type> '<json5 payload>'` — unit-test the logic.
2. Add/adjust `schema.js` → `npx tgcloud push` → `npx tgcloud migrate`
   (`--dry-run` first for anything non-additive).
3. `npx tgcloud status` — expect "in sync" (code) and webhook in sync.
4. If you added/removed handler files: `npx tgcloud webhook sync` if it's flagged.
5. Real-message test in Telegram; check behavior for updates you DON'T handle
   (should be ignored, no wake-ups).
