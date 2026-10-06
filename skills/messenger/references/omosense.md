# omosense: install, config, subscriptions

omosense is the resident daemon this skill depends on. It streams inbound messages, calendar/mail, reminders, session state and memory-tidy events to the agent, and sends outbound bot messages. Every command below is run as `bunx omosense ...`.

> **Status (2026-10-06).** Confirmed: the profile-based config, the `OMOSENSE_DIR` default and the commands on this page come from the omosense generalization work (branch `wish/omosense-generalize`, not yet merged or released).
> **Still to do:** the npm package that makes `bunx omosense` work, an install script and a release. Until they exist, stop at R3 and tell the user that omosense is not installable yet. Do not invent another install path.

## Install

1. Check that `bunx omosense --help` runs. If it does not, stop (see Status).
2. Choose the data directory: `$OMOSENSE_DIR`, default `~/.omosense`. State lives in `$OMOSENSE_STATE`, default `<dir>/state`. The socket is `$OMOSENSE_SOCK`, default `<dir>/omosense.sock`.
3. Write `<dir>/config.json` (shape below), then check it with `bunx omosense listen --profile <PROFILE> --dry-run`. The `PLAN` line must list the bots you expect. A config in an older shape is rejected with an error naming the offending key. Fix the key; never work around the error.

Bot credentials are kept by agent-messenger (`~/.config/agent-messenger/`). Never print tokens or put them in logs, briefs or public artifacts.

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

After arming, `bunx omosense daemon status` must show one client per armed source and each expected source `running` (not `paused`, `error` or `locked-by-other`). An armed subscription is not proof that the platform is connected. Watch the `LOG` lines for connection errors.

## Outbound

`bunx omosense say <platform> <action> '<json>' --profile <PROFILE>`. Add `{"bot":"<name>"}` to override the default bot. Replies, edits and reactions use the inbound event's `bot`. Run `bunx omosense say --help` for the current actions. Anything `say` and agent-messenger do not cover goes to the platform bot API directly, checked against its current docs (SKILL.md B3).

## Stop

Detach every monitor with its saved handle, then `bunx omosense daemon stop --profile <PROFILE>` (that profile only) or `bunx omosense daemon stop` (the whole daemon). Check `daemon status` afterwards. Bots, credentials, config and reminders are kept, so "`<NAME>` mode" turns it back on.
