# omosense: install, config, subscriptions

omosense is the resident daemon this skill depends on. It streams inbound messages, calendar/mail, reminders, session state and memory-tidy events to the agent, and sends outbound bot messages. Every command below is run as `bunx omosense ...`.

> **Status (2026-10-06).** The profile-based config, the `OMOSENSE_DIR` default and the commands on this page are merged in omosense and released on npm as `omosense` (latest `0.0.2`). `bunx omosense` (or `npx omosense`) runs it with no other install step. Nothing on this page is pending.

## Install

omosense is the npm package `omosense`, with prebuilt binaries for macOS and Linux on arm64 and x64. `bunx omosense ...` downloads and runs it; there is no separate installer.

1. Check that `bunx omosense --help` runs and prints `Usage: omosense <subcommand> ...`. To pin a version, write `bunx omosense@<version>` (current release `0.0.2`) and use that same form in every command and monitor below. If it does not run, stop and show the user the error output. Do not invent another install path.
2. Choose the data directory: `$OMOSENSE_DIR`, default `~/.omosense`. State lives in `$OMOSENSE_STATE`, default `<dir>/state`. The socket is `$OMOSENSE_SOCK`, default `<dir>/omosense.sock`.
3. Write `<dir>/config.json` (shape below), then check it with `bunx omosense listen --profile <PROFILE> --dry-run`. The `PLAN` line must list the bots you expect. A config in an older shape is rejected with an error naming the offending key. Fix the key; never work around the error.

Bot credentials are kept by agent-messenger (`~/.config/agent-messenger/`). Never print tokens or put them in logs, briefs or public artifacts.

`bunx` and `npx` each run omosense from an install cache they manage, and an auto-spawned daemon keeps running that cached binary. The omosense README warns that clearing the cache under a running daemon deletes its binary, and recommends `npm i -g omosense` for long-running daemons; after a global install, call `omosense ...` in place of `bunx omosense ...`.

## Config shape

Each profile is self-contained. There are no top-level `telegram`, `discord`, `owner` or `rpc` keys.

```json
{
  "profiles": {
    "<PROFILE>": {
      "telegram": { "bots": ["<bot-name>"], "roles": { "<user-id>": "owner" } },
      "discord":  { "bots": [], "roles": {} },
      "rpc":      { "enabled": true, "session": null, "all": false },
      "tidy":     { "enabled": true, "learnOthers": true, "exclude": [] },
      "memory":   "<this agent's memory repo id>",
      "calendars": [],
      "mail": false
    }
  }
}
```

- `bots`: bot names as registered in agent-messenger. An empty list means no source for that platform. `say` uses the first entry by default.
- `roles`: account id to role name. Any role name is allowed. `owner` is the privileged one, and anyone not listed is `other` (see SKILL.md I2).
- `rpc`: lets the agent watch the work sessions it started. `session` pins the delivery target. When it is null, the registered session is used.
- `tidy`: settings for the memory-tidy companion skill. See [memory-tidy](../../memory-tidy/SKILL.md).
- `memory`: this agent's own memory repo id under `~/.omo/memory/agents/`.

## Subscriptions

Arm one persistent monitor per source the profile uses, with the command `exec bunx omosense attach <source> --profile <PROFILE>` and the filter below. Keep each handle so you can detach it later. Do not arm a source twice: duplicate subscribers cause duplicate replies.

| Source | Filter | Events |
| --- | --- | --- |
| `listen` | `^(EVENT\|LOG) ` | Inbound messages: `platform`, `bot`, chat/channel id, optional `thread_id`, `message_id`, `role`, `text`, `reply_to`, `forwarded`, `quote`, `attachments`, and `transcribed` or `transcribe_error` for voice |
| `google` | `^(CAL\|SOON\|MAIL\|LOG) ` | New calendar events, starts within 30 minutes, unread mail |
| `remind` | `^(REMIND\|LOG) ` | Reminder delivery results (`sent`, `failed`, `skipped-late`) |
| `herdr` | `^(HERDR\|LOG) ` | Herdr panes turning blocked, or a registered pane going working to idle |
| `rpc` | `^(RPC\|LOG) ` | Session transitions for registered work sessions (blocked, done, opened, closed). Only when `rpc.enabled` |
| `tidy` | `^(TIDY\|LOG) ` | `TIDY {"changed":[{repo,from,to}]}`. Only when `tidy.enabled`. Hand it to the memory-tidy skill in a background write-capable worker |

- `attach` starts the daemon itself when its socket is missing or refuses connections, and reconnects on its own after an unexpected disconnect or a daemon upgrade. It exits 0 when the daemon or that profile is stopped or the monitor is killed, and non-zero when the daemon rejects the attach (unknown profile, invalid source, profile still stopping) or cannot be reached or spawned.
- `--only <PREFIX,...>` limits a client to those line prefixes (the daemon's own `LOG omosense` notices always pass). `--name <N>` names the client in `daemon status` (default `<source>-<PROFILE>`).
- `remind` and Discord `listen` are always-on: they keep running with no client attached, and their lines are recorded in `<state>/omosense-journal-<PROFILE>.jsonl` for replay. Other sources, Telegram `listen` included, pause when no client is attached. Detaching does not silence reminders or Discord; only a daemon stop does.

After arming, `bunx omosense daemon status` must show one client per armed source and each expected source `running` (not `paused`, `error` or `locked-by-other`). An armed subscription is not proof that the platform is connected. Watch the `LOG` lines for connection errors.

## Outbound

`bunx omosense say <platform> <action> '<json>' --profile <PROFILE>`. Add `{"bot":"<name>"}` to override the default bot. Replies, edits and reactions use the inbound event's `bot`. Run `bunx omosense say --help` for the current actions. Anything `say` and agent-messenger do not cover goes to the platform bot API directly, checked against its current docs (SKILL.md B3).

## Stop

Detach every monitor with its saved handle first. A connected `attach` exits when the daemon or its profile is stopped, but any later `attach` (for example a monitor that gets re-armed) spawns the daemon again and re-enables a stopped profile. Then pick one:

- **Pause:** `bunx omosense daemon stop`. Stops the whole daemon, every profile on it included. It cancels nothing, discards nothing and writes no stop marker: reminders, replay journals and all other state stay in place, and the next `attach` starts it again.
- **Permanent profile stop:** `bunx omosense daemon stop --profile <PROFILE>`. Stops only that profile's sources; the daemon and other profiles keep running. It marks every pending reminder of the profile `cancelled` (terminal, never sent later), discards the undelivered entries of the profile's replay journal, and writes `<state>/omosense-profile-<PROFILE>.stopped`, so the profile stays stopped across daemon restarts. The next `attach` of that profile re-enables it, but the cancelled reminders and discarded replay do not come back. It also works while the daemon is offline and prints one JSON result line. Use it only when the user wants that profile's pending work dropped.

Check `daemon status` afterwards. Bots, credentials and config are kept either way, so "`<NAME>` mode" turns it back on.
