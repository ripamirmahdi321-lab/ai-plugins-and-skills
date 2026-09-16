# `tgcloud` CLI — complete reference

The bridge between your project folder and the cloud. Requires **Node.js 18+**.
npm package: `@tgcloud/cli` (command `tgcloud`, MIT). As of 2026-09 the latest version
is 0.1.2 (published 2026-07-13).

Run it project-locally with `npx tgcloud <command>` (the scaffold installs it as a
dev-dependency and adds `npm run` shortcuts like `npm run deploy`, `npm run status`).
Optional global install: `npm install -g @tgcloud/cli` (needed for bare `tgcloud` and
for tab-completion, which can't hook into `npx`).

The CLI finds your project by walking up from the cwd to the nearest `.tgcloud/`, so
every command works from any subfolder. All commands exit non-zero on failure (rejected
deploy, failed migration, auth error, module that threw during `run`) — they compose in
scripts and CI.

## Authentication

A project is tied to **one bot** by its **CLI access token**, form `app<id>:<secret>`
(from @BotFather → your bot → **Serverless** → **CLI Access**). It is a *separate* token
from the bot's API token. The `app<id>` part is public and may be printed; the secret
never appears in logs or errors.

Resolution order:

1. `TGCLOUD_TOKEN` environment variable — for CI; never written to disk.
2. `.tgcloud/credentials` — written by `npx tgcloud login`.
3. Neither → error pointing at `npx tgcloud login`.

The CLI **never prompts for a token mid-command** (a surprise prompt would hang scripts
and CI). `login` is the only command that asks, and it requires a real TTY. If a saved
token becomes invalid (401/403) the CLI clears it and asks you to `login` again.

## Commands

### `init`

```bash
npx tgcloud init
```

Scaffolds a project in the current directory: `schema.js`, `lib/`, `handlers/`, a
starter `message` handler, `AGENTS.md`, `docs/` (with `tgcloud-sdk.md`),
`package.json`, and the `.tgcloud/` state folder. The starter file set is provided by
the platform (new files can appear without upgrading the CLI); offline it falls back to
a built-in copy. Refuses to nest inside another project (an ancestor with `.tgcloud/`);
re-running in your own project root just fills in what's missing.

Project creator alternative: `npm create @tgcloud/bot <folder>` (pass `.` for the
current folder; works in an existing folder; never overwrites existing files; also
installs the CLI into the project).

### `add <target>`

```bash
npx tgcloud add handlers/callback_query   # a new update handler
npx tgcloud add lib/cart                  # a new shared module
```

Scaffolds one module, wired up and ready to edit. `<target>` is the module path
(trailing `.js` optional). For `handlers/` the name must be a Telegram update type —
the platform advertises the valid set and invalid names are rejected up front.
`handlers/` is flat; `lib/` may nest (`lib/payments/stripe`). Never overwrites.
Giving just a directory is a helpful error: for `handlers/` it lists the update types
you don't have yet. The generated handler has a live `export default` — active as soon
you `push`.

### `login`

```bash
npx tgcloud login
```

Prompts for the CLI access token, validates it against the platform, saves it to
`.tgcloud/credentials`. Requires a real terminal; never runs (and never hangs) in CI.

### `status`

```bash
npx tgcloud status
```

Per file, what changed between the working directory and the deployed copy: modified,
new, deleted, unchanged. Fully offline (compares against the local reference copy in
`.tgcloud/`). A full run also warns about stray `.js` files at the project root and
flags webhook drift.

### `diff`

```bash
npx tgcloud diff
```

Like `status`, but shows actual line-by-line differences for changed modules. Offline.

### `push [files...]`

```bash
npx tgcloud push                       # deploy the whole project
npx tgcloud push handlers/message.js   # narrow which changes are sent
npx tgcloud push handlers/             # a directory
```

Deploys your project to the cloud in **one atomic batch** and updates the local record
of what the cloud holds.

- **No arguments**: deployed state mirrors your folder exactly — modules deleted
  locally are removed in the cloud.
- **File/dir arguments**: narrow which *changes* are sent, but still send the full
  manifest, so a targeted push never deletes untouched modules.
- `--force`: skip the concurrency check and overwrite whatever is in the cloud. Only
  when you're sure.
- After a deploy, if `schema.js` changed and the database is out of sync, `push` prints
  a summary of the pending changes and suggests `npx tgcloud migrate`. **It never
  applies them itself.**
- Deploying can leave the webhook temporarily out of sync; `status` flags it,
  `webhook sync` fixes it.

### `migrate`

```bash
npx tgcloud migrate
```

Applies schema changes to the database. Computes the difference between your deployed
`schema.js` and the live database, then walks through it with a running `[N/M]` counter:

1. A brief summary of all pending changes.
2. **Safe** changes (additive) — applied together in one step on confirmation.
3. **Warnings** (drops via `.deprecated()`, slow operations) — one at a time, each
   confirmed separately. There is no "apply all" for destructive changes.
4. **Manual** changes (e.g. column type change) — shown with a reason and suggested
   action; not applied.
5. **Undocumented** objects (in the DB, not in your schema) — shown for awareness.

Ends with a summary: applied, skipped, awaiting manual fix, not in schema.

Options:

| Flag | Effect |
|---|---|
| `--dry-run` | Print everything, apply nothing |
| `--safe` | Auto-apply safe changes, skip warnings |
| `--yes` | Auto-apply safe changes **and every warning**, skip manual — use with care |
| `--local` | Diff against your local `schema.js` instead of the deployed one |

Without a flag, `migrate` requires a terminal and errors in a non-interactive
environment rather than guessing (so CI must pass flags explicitly).

### `run <module> [args] [--ctx <json5>]`

```bash
npx tgcloud run handlers/message '{ chat: { id: 1 }, text: "hi" }'
```

Executes a handler on the platform **without deploying**, using your current **local**
files (the module space is assembled from your local project, so locally-changed
`lib/` code is used too). Prints everything the handler logged with `console.*`
(colored, `[file:line]`-tagged, with elapsed-time prefix), its return value, and how
long it took. The tightest iteration loop — no deploy, no real message needed.

- `<module>`: a bare name (searched under `handlers/`) or a path like `handlers/message`.
- `[args]`: the payload your handler receives — the update-type object — in **JSON5**
  (unquoted keys OK). For `handlers/message` that's a `Message`.
- `--ctx <json5>`: the handler's second argument (per-invocation context), e.g.
  `--ctx '{ update: { update_id: 1 } }'`.
- Big payloads: `npx tgcloud run handlers/message "$(cat message.json5)"`.

### `fetch`

```bash
npx tgcloud fetch
```

Refreshes the local reference copy of the deployed state without touching your working
files. Useful to re-check a conflict before deciding how to resolve it.

### `pull`

```bash
npx tgcloud pull
```

Brings your local project in line with the cloud — updates both the reference copy and
your working files to the deployed state.

### `reset`

```bash
npx tgcloud reset
```

Discards local changes and restores the working directory from the last known cloud
state. For throwing away an experiment.

### `webhook [sync]`

```bash
npx tgcloud webhook
npx tgcloud webhook sync [--drop-pending]
```

Telegram delivers updates to your bot through a webhook that **the platform manages for
you** — you never point it anywhere by hand. `webhook` shows: the URL, the
`allowed_updates` list, how many updates are pending, the last delivery error (if any),
and whether it is **in sync** with your deployed handlers.

"In sync" = the webhook points at the platform and `allowed_updates` match your
deployed handlers — Telegram delivers exactly the update types you handle, nothing
else. Deploying/removing a handler can leave it out of sync until refreshed;
`status` flags this too. `webhook sync` re-points the webhook and rebuilds
`allowed_updates` from your deployed handlers; `--drop-pending` discards updates
Telegram had already queued before the sync (otherwise they're delivered once the
webhook is healthy again).

### `completion <bash|zsh|fish>`

```bash
tgcloud completion bash     # needs the bash-completion package
tgcloud completion zsh      # ensure compinit in ~/.zshrc
tgcloud completion fish
```

Prints a shell completion script to stdout. `<Tab>` completes commands, flags, module
directories, the handler update-types you don't have yet, and local runnable modules —
computed live. Needs a global install (bare `tgcloud` on PATH).

## Staying in sync (team/CI)

Every project has a monotonically increasing **revision** in the cloud, bumped on each
deploy. The CLI remembers the revision it last synced with and sends it on each `push`.
If the cloud moved ahead (another machine or teammate deployed), the push is
**rejected** instead of silently overwriting their work. Three ways forward:

```bash
npx tgcloud fetch           # pull the latest into the reference copy, then re-check
npx tgcloud pull            # pull the latest into both reference and working files
npx tgcloud push --force    # overwrite the cloud state (dangerous)
```

CI pattern:

```bash
# CI: no TTY, no interactive prompts
export TGCLOUD_TOKEN="app123:…"      # secret env var
npx tgcloud push
npx tgcloud migrate --dry-run        # or --safe for additive-only deploys
```
