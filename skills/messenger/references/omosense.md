# omosense: install, config, the session host

omosense is the session host this skill depends on. It's one foreground process per session, run in the session folder. It hosts every source that folder's config enables (inbound messages, calendar and mail, reminders, herdr panes, work-session state, memory-tidy) and prints one line per event on stdout. It also sends outbound bot messages with `say`.

In this page `<session folder>` is the messenger agent session's working directory: the bot folder that holds the omo-mate clone (see README Install). Every command is written `bunx omosense@latest ...` and is run from that folder (see [Install](#install) for why).

## Install

omosense is the npm package `omosense`. `bunx omosense@latest ...` downloads and runs it; there's no separate installer.

1. Always use the `@latest` tag, never a bare `bunx omosense`. A bare name can keep running an older cached copy; `@latest` checks npm on every run.
2. Run `bunx omosense@latest --help`. It must print `Usage: omosense [<subcommand> [flags]]` followed by a line starting `Bare omosense runs the session host`. If it prints anything else or doesn't run, stop and show the user the output. Don't invent another install path.

`@latest` needs network on every start, and a host started this way keeps the version it started with until it's restarted.

bunx and npx run the binary from a cache they manage. Clearing that cache while omosense runs deletes the binary under it. For a session that stays up for days, install it globally with `npm i -g omosense@latest` and call `omosense ...` in place of `bunx omosense@latest ...` everywhere below.

Bot credentials are kept by agent-messenger in `~/.config/agent-messenger/`. Never print tokens or put them in logs, briefs or public artifacts.

## Config

The config lives in `<session folder>/.omosense/config.json`. State lives next to it in `<session folder>/.omosense/state/` (reminders, registered threads, pending work-session completions, the memory-tidy watermark, Telegram attachments, lock files). One of those files is written by the agent, not by omosense: `threads.json` holds the work sessions this agent started (see [Registry](sessions.md#registry)). The config is flat:

```json
{
  "telegram": { "bot": "<bot-name>", "roles": { "<user-id>": "owner" } },
  "discord":  { "roles": {} },
  "rpc":      { "enabled": true, "all": false },
  "tidy":     { "enabled": true, "learnOthers": true, "exclude": [], "checkMin": 10, "quietMin": 60 },
  "herdr":    { "enabled": true, "blockedAll": false },
  "memory":   "<this agent's memory repo id>",
  "mail": false
}
```

| Key | Meaning |
| --- | --- |
| `telegram.bot`, `discord.bot` | One bot name (a string) per platform, as registered in agent-messenger. Unset means that platform's listener doesn't run. |
| `telegram.roles`, `discord.roles` | User id to role name. The key is the sender's user id, the `from_id` of their `EVENT` (on Telegram the numeric `from.id`, not an @username and not a group `chat_id`). SKILL.md B4 shows how to get it on first run. Any role name is allowed. `owner` is the privileged one, and anyone not listed is `other` (see SKILL.md I2). |
| `rpc.enabled` | Watch the work sessions this agent started. |
| `rpc.all` | Watch every session, not only those registered in `threads.json`. |
| `tidy.enabled`, `tidy.learnOthers`, `tidy.exclude` | Settings for the memory-tidy companion skill. See [memory-tidy](../../memory-tidy/SKILL.md). |
| `tidy.checkMin` | How often tidy checks the memory repos, in minutes. Default `10`. Any number greater than 0, fractions included. Needs omosense 0.2.0 or newer; older versions ignore it. |
| `tidy.quietMin` | How long a changed repo's HEAD commit must be quiet before a `TIDY` line reports it, in minutes. Default `60`. Any number greater than 0, fractions included. Needs omosense 0.2.0 or newer; older versions ignore it. |
| `herdr.enabled` | Absent means on. Only an explicit `false` turns herdr off. |
| `herdr.blockedAll` | Default `false`: a `blocked` line prints only for registered job panes. `true` prints it for every pane except omosense's own. |
| `memory` | This agent's own memory repo id. It's the `AGENT_ID: <id>` line under `<memory_metadata>` in this agent's system prompt, and the repo is `~/.omo/memory/agents/<id>/repo`. |
| `calendars` | Which calendars google watches. Absent means all of them. |
| `mail` | Whether google also watches mail. |

One bot belongs to one session.

## Check the config

```sh
cd <session folder> && bunx omosense@latest listen --dry-run
```

It exits 0 and prints one `PLAN` line, for example `PLAN {"dir":"<session folder>/.omosense","discord":[],"lock":"listen","state":"<session folder>/.omosense/state","telegram":["<bot-name>"]}`. Check that each platform lists the bot you expect; a platform without a bot shows `[]`. The dry run creates no state.

A config in an older shape, or with a value of the wrong type, exits 1 with a message naming the key, for example `omosense: config.json: telegram.bot must be a string`. Fix the named key. Never work around the error.

## Subscriptions

Arm exactly one persistent monitor per session. It runs the host, and the host runs every enabled source:

```
command: cd <session folder> && exec bunx omosense@latest
filter:  ^(EVENT|CAL|SOON|MAIL|REMIND|HERDR|RPC|TIDY) 
persistent: true
```

`exec` makes the stop signal reach the binary. `LOG` is left out of the filter so housekeeping lines don't spend the monitor's event budget. Keep the handle; you need it to stop.

Never arm two. A second omosense in the same folder can't take the sources: each one logs `ALREADY_RUNNING` and retries every 30 seconds.

| Source | Runs when | Prints |
| --- | --- | --- |
| listen Telegram | `telegram.bot` is set | `EVENT` |
| listen Discord | `discord.bot` is set | `EVENT` |
| remind | always | `REMIND` |
| google | always; needs `zele` on PATH, otherwise it only logs an error and reports nothing (`zele` is optional) | `CAL`, `SOON`, `MAIL` |
| herdr | always, unless `herdr.enabled` is `false` | `HERDR` |
| rpc | `rpc.enabled` is `true` | `RPC` |
| tidy | `tidy.enabled` is `true` | `TIDY` |

Every source can also print `LOG`. A crashed source logs `LOG omosense source <name> crashed: <err>` and restarts with a backoff of 5 seconds, doubling up to 5 minutes.

An `EVENT` line carries `platform`, `bot`, `kind`, `chat_id` and `chat_type` (Telegram) or `channel_id` and `guild_id` (Discord), and optionally `thread_id`, `message_id`, `role`, `from`, `from_id`, `text`, `reply_to`, `forwarded`, `quote`, `attachments` (each with `path` once downloaded, or `error`), and `transcribed` or `transcribe_error` for voice.

`reply_to` is `null` unless the message replies to another one. Then it's `{"message_id": <id>, "text": "<replied text>"}`, plus `from` (the replied author's username) when known. On Discord `text` is present only when the replied content is available. Telegram bots can't read chat history, so `reply_to`, `quote` and your own records are the Telegram history.

herdr prints `blocked` at once, but a registered job pane going from `working` to `idle` or `done` is kept in `state/herdr-pending.json`; once 5 minutes pass with no newer completion, all of them come out as one `HERDR {"event":"done-batch","entries":[...]}` line, one entry per pane. A remote herdr machine that can't be reached is asked again after 10, 20, 40, 80, 160, then 300 seconds; only its first failure is logged, and recovery logs `herdr <label> recovered`.

A `TIDY {"changed":[{repo,from,to}]}` line goes to the memory-tidy skill, run in a background write-capable worker. One line carries at most 10 repos; more come as further lines. The same HEAD isn't reported again within 6 hours, across restarts too (`state/tidy-announced.json`). Spawn the worker with `load_skills: ["memory-tidy"]` (this session has the skill from `--skill`). If the task tool reports it missing, the worker reads `<session folder>/omo-mate/skills/memory-tidy/SKILL.md`.

After arming, read the monitor's own output. Look for the `LOG omosense host starting dir=<session folder>/.omosense sources=...` line (bunx may print its own resolve lines first), and the list must name the sources you expect. An armed host isn't proof the platform is connected; watch for connection errors in its `LOG` lines. Even a start line doesn't prove the platform delivers messages: to prove it, send the bot a message and watch the host output for its `EVENT` line. Messages that arrive while the host is off aren't received. There's no queue.

## Health and recovery

A dead host is silent. No lines doesn't mean healthy, and messages that arrive while it's down are lost.

Run this check on every "`<NAME>` mode" re-entry, and whenever a watch you expect stays silent:

1. Read the host monitor's exit summary, if it has one. An exited monitor means the host is gone.
2. Read the pids in `<session folder>/.omosense/state/*.lock.json` and run `kill -0 <pid>` on each. If it fails, that holder is dead. A dead holder's lock is taken over by the next host, so it never blocks a restart.
3. After re-entry, look for `LOG omosense host starting` in the monitor output. If it's missing, the host didn't start.

To recover:

1. Kill the old monitor handle if there is one.
2. Arm exactly one host monitor, as in [Subscriptions](#subscriptions). Never a second: a second host logs `ALREADY_RUNNING` and retries every 30 seconds.
3. Find its `LOG omosense host starting` line (bunx may print its own resolve lines first) and check that `sources=` names the sources you expect.
4. Run `rpc pending`, then `rpc subscribe <session-id>` (see [Work-session completions](#work-session-completions)).

A long-lived session should prefer the global install (see [Install](#install)). Clearing the bunx cache deletes the binary a bunx host runs from.

## Work-session completions

With `rpc.enabled`, work sessions that turn `blocked`, `opened` or `closed` come out as `RPC` lines. A finished session (`done`) doesn't. Completions are kept in the state folder and sent as one batched message to the subscribed session once 5 minutes pass with no new completion. Nothing is re-sent until it's acked.

```sh
cd <session folder> && bunx omosense@latest rpc subscribe <session-id>   # send done batches here
cd <session folder> && bunx omosense@latest rpc subscription             # show the subscriber
cd <session folder> && bunx omosense@latest rpc pending                  # list un-acked completions
cd <session folder> && bunx omosense@latest rpc ack <id> [<seq>]         # clear one
```

`<session-id>` is this messenger session's own `durableSessionId` (the same value as its `thread_id`). To find it, run `omo thread list --all-scope --json`. It prints an array of objects with, among others, the fields `sessionId`, `durableSessionId`, `status`, `cwd` and `name`; pick this session's entry by `cwd` and `name`. `sessionId` is the routing handle, not the subscribe id. After `rpc subscribe`, run `rpc subscription` and confirm it shows the same id. Each entry in a batch carries its own ack command; run it as written.

`subscribe` checks the id against `omo thread list` first. An id that isn't a live thread is refused: it exits 1 and saves nothing. If `omo` can't be queried, it warns and saves the id anyway. Liveness is checked again when a batch is due. With no subscriber the batch is dropped (`LOG rpc batch dropped: no subscriber`). With a subscriber that isn't alive the batch is dropped and the subscription removed (`LOG rpc batch dropped: subscriber <id> not alive; unsubscribed`). So on every "`<NAME>` mode" re-entry, run `rpc pending` to catch up, then `rpc subscribe <session-id>` again.

## Reminders

Add a reminder with `remind add`:

```sh
cd <session folder> && bunx omosense@latest remind add --in 30m --platform telegram --target '{"chat_id":123456789}' --text "<text>"
```

The full form is `remind add (--at TIME | --in DURATION) --platform telegram|discord --target JSON --text TEXT [--id ID]`. It prints `REMIND added <entry>`. A time more than 6 hours in the past is refused with exit 2, and a failed read or write exits 1. It holds `state/reminders.lock`, the same lock the remind tick takes, so an add and a tick never overwrite each other.

Reminders live in `<session folder>/.omosense/state/reminders.json`, a JSON array. An entry:

```json
{"id":"<id>","at":"2026-10-08T09:00:00+09:00","platform":"telegram","target":{"chat_id":123456789},"text":"<text>"}
```

`target` is the `say` JSON without `text`. `at` is an ISO datetime with `Z` or an offset (preferred), a naive datetime (host-local time) or a date only. An unparseable `at` is sent immediately.

The remind source always runs. It checks at start and every 20 seconds and sends a due entry with `say <platform> send`. Then it marks the entry `sent`, or `failed` plus `error` (never retried), or `skipped` when it's more than 6 hours late. `cancelled` is a marker you set; the host never cancels. Entries with any of these markers are left alone. After writing the file it prints `REMIND sent <entry json>`, `REMIND failed <entry json>` or `REMIND skipped-late <entry json>`.

The host rewrites the whole file (a 2-space indented array) when it marks entries. Add entries with `remind add`, not by editing the file.

## Outbound

```sh
cd <session folder> && bunx omosense@latest say <platform> <action> '<json>'
```

Telegram actions: `send`, `edit`, `draft`, `typing`, `react`, `unreact`, `topic`, `topic-edit`, `photo`, `doc`. Discord actions: `send`, `edit`, `typing`, `react`, `unreact`, `thread`, `thread-edit`, `file`.

Fields per action, as of omosense origin/main e00e8f0. `omosense say --help` lists each action with its fields. `say` sends only these keys; any other key is dropped without an error, so a misspelled key (for example `reply_to_message_id`) just goes missing.

| Platform | Action | Fields | Optional |
| --- | --- | --- | --- |
| Telegram | `send` | `chat_id`, `text` | `reply_to` (the message id to reply to), `parse_mode`, `thread_id` |
| Telegram | `edit` | `chat_id`, `message_id`, `text` | `parse_mode` |
| Telegram | `draft` | `chat_id`, `draft_id`, `text` | `thread_id` |
| Telegram | `typing` | `chat_id` | `thread_id` |
| Telegram | `react` | `chat_id`, `message_id` | `emoji` (default 👀) |
| Telegram | `unreact` | `chat_id`, `message_id` | |
| Telegram | `topic` | `chat_id`, `name` | |
| Telegram | `topic-edit` | `chat_id`, `thread_id`, `name` | |
| Telegram | `photo`, `doc` | `chat_id`, `path` (a local file) | `caption`, `thread_id` |
| Discord | `send` | `channel_id`, `text` | `reply_to` (the message id to reply to) |
| Discord | `edit` | `channel_id`, `message_id`, `text` | |
| Discord | `typing` | `channel_id` | |
| Discord | `react`, `unreact` | `channel_id`, `message_id` | `emoji` (default 👀) |
| Discord | `thread` | `channel_id`, `name` | `message_id` (start the thread from that message) |
| Discord | `thread-edit` | `thread_id` | `name`, `archived` |
| Discord | `file` | `channel_id`, `path` (a local file) | `text` |

Discord `send`, `edit` and `file` also take `content` as an alias of `text`.

On Telegram, `thread_id` is the topic (the `EVENT` `thread_id`).

`say` uses the platform's configured bot. Add `"bot":"<name>"` to the JSON to override it; the key is stripped before the request. With no bot configured and no override, `say` exits 2. Replies, edits and reactions use the inbound event's `bot`. Anything `say` and agent-messenger don't cover goes to the platform bot API directly, checked against its current docs (SKILL.md B3).

Telegram specifics:

- `draft` calls `sendMessageDraft`. It works in private chats only and shows an ephemeral preview. Finalize it with `send`.
- `sendRichMessage` isn't a `say` action. Call the bot API directly, per SKILL.md B3.
- Topics in the private chat with the bot need Threaded Mode, switched on in BotFather's Mini App. No Bot API method toggles it.

Reading history:

- Discord: `GET /channels/{channel.id}/messages` with the header `Authorization: Bot <token>`. The token is in `~/.config/agent-messenger/discordbot-credentials.json`. Never print it.
- Telegram: none. Use `reply_to`, `quote` and your own records.

## Stop

Kill the host monitor with its saved handle (`kill_bash`). The host gets the signal, stops every source, releases its locks and exits 0. There's nothing else to stop. Reminders and the rest of the state stay in `.omosense/state/`, and re-arming the monitor ("`<NAME>` mode") starts it again. While it's off, nothing is received.
