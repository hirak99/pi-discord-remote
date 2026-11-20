# pi-discord-remote

> **Superseded standalone repository:** This fork is now maintained in
> [`hirak99/pi-arnab-extensions`](https://github.com/hirak99/pi-arnab-extensions)
> under [`packages/discord-remote`](https://github.com/hirak99/pi-arnab-extensions/tree/main/packages/discord-remote).
> Use the monorepo for current development and installation. This repository is
> retained for its standalone history.
>
> Control your [Pi](https://pi.dev) coding-agent session from Discord.

![pi-discord-remote demo](demo.gif)

Each time you run `/pi-discord-remote start`, the extension automatically creates a **new Discord text channel** named after your current project + date (e.g. `kaleidoscope-may09`). Messages sent in that channel are injected into Pi as user prompts; Pi's responses are posted back. When you stop, the channel is deleted — keeping your server clean within Discord's channel limit.

## Fork additions

Everything in this section is **not in upstream** — it exists only in this fork.

### Installing this fork

The upstream `pi install npm:pi-discord-remote` command installs the *npm*
package, not this fork. To run this fork's code:

```bash
# 1. clone and build
git clone https://github.com/hirak99/pi-discord-remote ~/git/pi-discord-remote
cd ~/git/pi-discord-remote
npm install && npm run build

# 2. load it via pi's auto-discovered extensions dir (symlink keeps the
#    clone as the single source of truth)
mkdir -p ~/.pi/agent/extensions
ln -sfn ~/git/pi-discord-remote ~/.pi/agent/extensions/pi-discord-remote

# 3. configure and connect
#    edit ~/.pi/agent/pi-discord-remote/config.json (token, guildId, allowedUserIds)
#    in pi:
/pi-discord-remote start
```

Note: pi's `/reload` does **not** pick up rebuilt extension code (module
caching). After `npm run build`, restart pi, then `/pi-discord-remote start`.

Fork-only config keys used below: `displayNames`, `echo`, `steer`,
`autoStart`, `staticChannel`, `channelName`. For day-to-day operations — reconnecting after a reboot,
diagnosing failures, the fork dev loop — see [docs/notes.md](docs/notes.md).

### Message attribution

Prompts received from Discord are attributed to their author, so the agent
knows who is driving: messages arrive in Pi as `[Name (<@userID>)]: <text>`.
The name is resolved from the `displayNames` config map (keyed by user ID)
first, then the server nickname, then the global display name. If none is
available, the format is `[<@userID>]: <text>` without duplication. Reusing
the mention token in a Discord reply produces a real user ping. A
`displayNames` value of `null` marks the **default user** — their messages
are injected with no prefix at all.

| Config key | Type | Default | Description |
|------------|------|---------|-------------|
| `displayNames` | object | `{}` | Friendly-name overrides keyed by Discord user ID, shown with the `<@userID>` mention token. String wins over nickname / global name / username; `null` = default user, no prefix |

### Echo (two-way mirror)

By default the channel only shows turns that started in Discord. With
`"echo": true` in the config, everything is mirrored both ways:

- prompts typed in the Pi terminal appear in the channel as `🖥️ **Terminal**: …`,
  with a horizontal-rule divider appended so the boundary before the agent's reply is clear
- the agent's replies to those turns are posted as completed messages,
  each appended independently in transcript order when it finishes
- a standalone divider closes the run; Discord-originated runs include the
  sender's completion mention in that footer
- tool-call starts post as short, emoji-labeled summaries alongside the replies;
  tool results are never mirrored

Messages sent from Discord are not echoed back — the channel already shows
them.

| Config key | Type | Default | Description |
|------------|------|---------|-------------|
| `echo` | boolean | `false` | Mirror terminal-originated turns into the Discord channel |

### Steering (mid-turn messages)

By default, a message sent from Discord while a turn is running gets a
"⏳ please wait" reply and is **dropped**. With `"steer": true`, the message
is instead injected into the running turn with pi's `deliverAs: "steer"` —
the agent receives it at its next step, before its next LLM call, as
`[Name (<@userID>)]: <text>` — and the message gets a ⏩ reaction. Delivery is attributed
and deduped against echo exactly like idle-time messages.

Steering takes effect between model calls, so a long-running tool call
finishes before the agent reads your message. Aborting mid-tool (discord_agent's
`!`-style interrupt) is not implemented — pi's extension API only exposes
`ctx.abort()` inside event handlers.

The busy flag is released on `agent_settled` (not `agent_end`), so messages
arriving between sub-runs steer correctly instead of being rejected.

| Config key | Type | Default | Description |
|------------|------|---------|-------------|
| `steer` | boolean | `false` | Inject Discord messages received mid-turn into the running turn |

### Working / typing feedback

While Pi displays its built-in `⠏ Working` indicator (`turn_start` through
`agent_end`), the mirrored Discord channel shows the bot's native typing
indicator. It is refreshed every 8 seconds because Discord expires typing
automatically. Refreshing stops at `agent_end`, on disconnect, and during
shutdown. Discord has no explicit stop-typing API, so an already-sent indicator
can linger briefly after work finishes.

### Interruptions

When `agent_settled` reports `aborted: true`, the persistent session channel
receives `🛑 Agent interrupted`. Abortion can come from a human, an extension,
or session shutdown, so the notice deliberately does not claim a human cause.

### Provider errors

When a model request fails (e.g. `500 server_error`), Pi records an assistant
message with `stopReason: "error"`. The persistent session channel receives:

- `⚠️ Provider error (Pi may retry automatically)` for the first error of a
  streak (repeats during automatic retries are suppressed)
- `✅ Recovered after N provider errors` if a later request succeeds
- `<@user> ❌ Run failed after N provider errors` at `agent_settled` if the run
  ended on an error (not aborted), mentioning the Discord user who started it

Pi's `auto_retry_*` events are not delivered to extensions, so attempt counts
and delays are inferred, not read. Error text is code-fenced, whitespace-
collapsed and capped at 400 characters. Tool errors are deliberately not
mirrored.

### Subagent and background-job activity

When `pi-subagents` and/or `pi-background-tasks` is installed, their activity
is reported in the persistent session channel:

- subagent starts update internal state without posting an immediate message;
  running background jobs are fetched on demand through the plugin's EventBus
  status API
- every `agent_end` posts one combined snapshot below the turn divider, with a
  🤖 subagent section and/or an ⚙️ background-job section: 🆕 marks an activity's
  first snapshot and ⏳ marks activities reported by an earlier snapshot
- subagent completions/failures and background-job terminal events always post
  immediately, including short-lived activities and activities that finish
  while Pi is idle; completed jobs use ✅, failed jobs ❌, and killed jobs 🛑
- reports use trimmed names/descriptions (up to 80 characters), with type/ID
  fallbacks; immediate completion messages and running snapshots share one
  serialized delivery queue to preserve order
- duplicate background terminal events are ignored by task ID

Both plugins are optional and accessed only through `pi.events`. If either is
absent, its section is silently omitted; if neither has running activity, no
snapshot is posted and all other Discord features continue unchanged.

### Auto-start (per-project opt-in)

By default nothing connects until you run `/pi-discord-remote start`. A
project can opt in to connecting automatically on **new and resumed**
sessions by adding `.pi/discord-remote.json` to the project:

```json
{ "autoStart": true }
```

At session start the extension merges that file over the global config
(`~/.pi/agent/pi-discord-remote/config.json`) — any key can be overridden
per project (e.g. a different `categoryId`), but the bot `token` only needs
to live in the global config. Details:

- auto-connect fires only for `session_start` reasons `new` and `resume`;
  reloads keep the existing connection, and sessions without the project
  file stay manual
- an unparseable project file logs a warning and falls back to the global
  config — a broken opt-in never breaks the session
- `/pi-discord-remote start` also honors the project overrides

| Config key | Type | Default | Description |
|------------|------|---------|-------------|
| `autoStart` | boolean | `false` | Connect to Discord automatically when a new or resumed session starts (intended as a per-project opt-in via `.pi/discord-remote.json`) |

### Static channels

By default each session creates a fresh, date-suffixed channel and deletes it
on session end. With `"staticChannel": true` the channel becomes a permanent,
per-project archive:

- **Stable name** — `<project-dir>` (or `channelName`), no date suffix
- **Preserved** — the channel and its history are kept when the session ends;
  a 🔒 closing message is posted and `SendMessages` is denied to `@everyone`,
  making the channel read-only
- **Reused** — the next session in the same project adopts the channel: the
  read-only lock is lifted and a 🔓 "live again" message is posted. The
  project→channel binding is persisted in `~/.pi/agent/pi-discord-remote/channels.json`
  (with a name-based search as fallback for channels created before the
  binding existed)

Typical per-project opt-in (`.pi/discord-remote.json`):

```json
{ "autoStart": true, "staticChannel": true, "channelName": "process-graph" }
```

`channelName` is optional — omit it to use the slugified project directory
name. Set it when two different projects share a directory basename (a new
session would otherwise adopt the other's channel). It is honored
independently of `staticChannel`: without static mode there is no lock/reuse,
and concurrent sessions in the same project produce same-named channels
(Discord allows duplicates — each session only listens to its own).

Changing `channelName` needs no manual Discord rename: the binding identifies
a static channel by ID, and at adoption the bot renames the channel to the
configured name — history moves with it, nothing is orphaned. (Discord-side
renames without a config change are won back the same way, since an explicit
`channelName` is declared intent.)

Caveat: two *live* sessions in the same project share one channel — their
messages interleave and unlock/lock can race. Static channels assume one
session per project.

| Config key | Type | Default | Description |
|------------|------|---------|-------------|
| `staticChannel` | boolean | `false` | Keep the channel across sessions: lock read-only on session end, reuse (unlock) on the next session in the same project |
| `channelName` | string | — | Explicit channel name — no date suffix, honored with or without `staticChannel` (default: slugified project dir for static, date-suffixed per-session name otherwise) |

### Connection diagnostic

`test-connect.mjs` (repo root) attempts the same gateway login the extension
performs and reports exactly which of token / intents / guild / permissions
is failing — without printing the token:

```bash
node test-connect.mjs
```

## Install

```bash
pi install npm:pi-discord-remote
```

## Bot setup

1. Create a bot at [discord.com/developers/applications](https://discord.com/developers/applications)
2. Under **Bot → Privileged Gateway Intents**, enable **Message Content**
3. Invite the bot to your server with these permissions:
   - Read Messages / View Channels
   - Send Messages
   - Add Reactions
   - **Manage Channels** ← required for auto-create/delete
   - **Manage Roles** ← required for static-channel lock/unlock (editing
     channel permission overwrites is gated on this, not Manage Channels)

## Usage

```
/pi-discord-remote setup        — configure token, server ID, optional category
/pi-discord-remote start        — create channel + connect
/pi-discord-remote stop         — delete channel + disconnect
/pi-discord-remote status       — show connection state
/pi-discord-remote open-config  — edit config JSON in Pi's editor
```

### Setup prompts

| Field | Where to find it |
|-------|-----------------|
| Bot token | Discord Developer Portal → Bot → Token |
| Guild (Server) ID | Right-click server → Copy Server ID (needs Developer Mode) |
| Category ID | Right-click a category → Copy Category ID (optional — channels go to server root otherwise) |
| Allowed user IDs | Right-click a user → Copy User ID (leave empty to allow everyone) |

Config is stored at `~/.pi/agent/pi-discord-remote/config.json`.

### Environment variables (alternative to config file)

For CI or headless setups, you can skip `setup` entirely and set these env vars:

| Variable | Description |
|----------|-------------|
| `DISCORD_TOKEN` | Bot token (overrides config file) |
| `DISCORD_GUILD_ID` | Guild (server) ID |
| `DISCORD_CATEGORY_ID` | Optional category ID |

## Config reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `token` | string | — | Discord bot token |
| `guildId` | string | — | Discord guild (server) ID |
| `categoryId` | string | — | Optional category for auto-created channels |
| `allowedUserIds` | string[] | `[]` | Allow-list of Discord user IDs (empty = everyone) |
| `reactions` | boolean | `true` | React with ⏳ while processing |
| `autoStart` | boolean | `false` | Connect automatically on new/resumed session start — intended as a per-project opt-in via `.pi/discord-remote.json` (see above) |
| `staticChannel` | boolean | `false` | Keep the channel across sessions (lock read-only on end, reuse on next session) — see above |
| `channelName` | string | — | Explicit channel name, no date suffix, honored with or without `staticChannel` — see above |

Edit config with `/pi-discord-remote open-config`.

## How it works

The extension loads silently on Pi startup — no channel is created until you explicitly run the command.

- **`/pi-discord-remote start`** — bot logs in, creates a text channel named `<project>-<mon><dd>-<HHMM>`, and starts listening there only
- **Incoming message** — injected as a user prompt into the active Pi session; bot reacts ⏳ while Pi works, then posts the completed response back. Each assistant reply is appended as its own message, in transcript order, when it completes (no live editing)
- **Tool calls** — each invocation posts a short summary with an emoji and a path when relevant (🔧 running a command, 📄 reading a file, 📝 writing to a file); commands, search patterns, and tool results are not mirrored
- **`/pi-discord-remote stop`** (or Pi exit) — channel is deleted, bot disconnects

### Optional image skill tool

Image forwarding is **not automatic** in `pi-discord-remote`.

If you want to send an image, use the explicit `discord_send_image` tool (opt-in):
- optional `channelId` (recommended for deterministic targeting)
- source by local `path`
- or source by `url`
- or source by `base64` (+ optional `mediaType`)

Exactly one source should be provided per call. If omitted, the tool falls back to the latest `agent_browser` image artifact path.

`/pi-discord-remote status` now shows the active **Channel ID** so you can pass it directly to `discord_send_image`.

The tool returns explicit send errors (for example `unknown_channel:<id>` or timeout/HTTP errors) instead of silently forwarding images.

### `ask_user_question` → Discord

Pi's TUI-only `ask_user_question` dialog (from `@juicesharp/rpiv-ask-user-question`) is invisible over Discord remote. When Discord is connected, pi-discord-remote:

1. **Blocks** the original `ask_user_question` via `tool_call` event interception — the TUI dialog never appears
2. **Registers** `discord_ask_user_question` — a drop-in replacement that formats questions as Discord messages and collects answers from the channel
3. **Hints** the LLM via `before_agent_start` to prefer `discord_ask_user_question` when Discord is connected

When Discord **is not** connected, the original `ask_user_question` (TUI dialog) works normally as a fallback. No need to uninstall `@juicesharp/rpiv-ask-user-question` — the two extensions coexist gracefully.

## License

MIT
