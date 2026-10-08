# omo-mate

Two OmO skills that turn your agent into an always-on messenger friend: it lives in a chat bot, listens only to the people you register, does the work itself, and keeps its memory tidy.

| Skill | What it does |
| --- | --- |
| [`messenger`](skills/messenger/SKILL.md) | One-time setup (name, platform, avatar), then runs `<NAME> mode`: inbound messages, replies, work threads, work sessions, monitors. |
| [`memory-tidy`](skills/memory-tidy/SKILL.md) | Folds the memories of your other OmO agents into the messenger agent's own memory, deduplicated, with source pointers. |

omosense connects the two: one foreground process per agent session, run in that session's folder, that streams messages, calendar, session and `TIDY` events to the agent (one bot, one omosense).

## Install

### Why one folder per bot

A messenger skill means one bot, one folder. A global install would put the skill descriptions into every OmO session you open, which wastes tokens, and it leaves room for a session that never meant to make a bot to try making one. With a folder install, only the session opened in that folder sees the skills. Removing a bot means deleting its folder, and updating means a `git pull`.

### Layout

```text
~/bots/<bot-name>/          bot folder = the agent session's working folder
├── omo-mate/               this repository, cloned
└── .omosense/
    ├── config.json         omosense config
    └── state/              omosense state (reminders.json and the rest)
```

### Set up a bot folder

```sh
mkdir -p ~/bots/<bot-name> && cd ~/bots/<bot-name>
git clone https://github.com/DevNewbie1826/omo-mate.git
```

Then, inside a [Herdr](https://herdr.dev) pane, open a plain OmO session from the bot folder:

```sh
omo
```

and hand it the repository:

```text
Read https://github.com/DevNewbie1826/omo-mate — clone it here as omo-mate/,
read skills/messenger/SKILL.md, and set up the bot following it.
```

The session clones the repository itself, reads the skill documents, and runs the setup — no skill registration is needed. The documents are self-contained, and every command they use runs from this bot folder. On later `TIDY` events the session reads `skills/memory-tidy/SKILL.md` from the clone the same way.

Optional: to also register both skills for auto-discovery, open the session with `omo --skill omo-mate/skills/messenger --skill omo-mate/skills/memory-tidy` instead (they show up as `/skill:messenger` and `/skill:memory-tidy`). Reopen with `omo -c` (plus the same flags, if you use them) to continue the last session.

### Update

From the bot folder:

```sh
git -C omo-mate pull
```

### Remove

Stop the host (see [Usage](#usage), step 8), then delete the bot folder. These live outside the folder and stay behind:

- bot credentials in `~/.config/agent-messenger/`
- the agent's memory repo, `~/.omo/memory/agents/<id>/repo`
- memory-tidy backups in `~/.omo/memory-backups`
- the bot account itself on the platform

Installed the old plugin earlier? Remove it with `omo remove git:github.com/DevNewbie1826/omo-mate`.

## Usage

Run every `omosense` command from the bot folder. See [Install](skills/messenger/references/omosense.md#install) for how to install it and which version tag to use.

### 1. Clone and open the session

Follow [Install](#install): bot folder, clone, then open a session in a Herdr pane and hand it this repository (or the cloned `skills/messenger/SKILL.md` path) — no flags needed.

### 2. First run: configure the bot

Say `set up messenger mode` (in any language). The skill asks three questions (name, platform, avatar), creates the bot and writes the config.

You can also write `.omosense/config.json` by hand. It's flat and every key is optional:

```json
{
  "telegram": {
    "bot": "<bot-name>",
    "roles": { "123456789": "owner" }
  },
  "rpc": { "enabled": true },
  "tidy": { "enabled": true },
  "herdr": { "enabled": true },
  "memory": "<agent-id>"
}
```

`telegram.bot` (or `discord.bot`) is one bot name. `roles` maps user ids to roles, and `owner` is the privileged one. `memory` is the messenger agent's own `AGENT_ID`, from its memory metadata. When `herdr` is absent it counts as on. See [config](skills/messenger/references/omosense.md#config) for every key.

### 3. Check the config

```sh
bunx omosense@latest listen --dry-run
```

It exits 0, creates no state, and prints one line:

```text
PLAN {"dir":"<bot folder>/.omosense","discord":[],"lock":"listen","state":"<bot folder>/.omosense/state","telegram":["<bot-name>"]}
```

A wrong type exits 1 with a message such as `omosense: config.json: telegram.bot must be a string`.

### 4. Start the host

The agent runs the host itself, as one persistent monitor ([subscriptions](skills/messenger/references/omosense.md#subscriptions)):

```text
command: cd <bot folder> && exec bunx omosense@latest
filter:  ^(EVENT|CAL|SOON|MAIL|REMIND|HERDR|RPC|TIDY) 
```

For a one-time manual check before that, run it in the foreground:

```sh
bunx omosense@latest
```

Look for the `LOG omosense host starting dir=<bot folder>/.omosense sources=<names>` line (bunx may print its own resolve lines first). Ctrl-C stops it and it releases its locks. Never run it beside the agent's monitor: a second host logs `ALREADY_RUNNING` and retries every 30 s.

### 5. Run the session

Saying `<NAME> mode` turns the mode back on in a reopened session. Messages from the people in `roles` arrive as `EVENT` lines, and the agent answers through the bot. Work sessions are covered in [sessions](skills/messenger/references/sessions.md). If the host misbehaves, see [health and recovery](skills/messenger/references/omosense.md#health-and-recovery).

### 6. Replies and notices

`say` sends through the configured bot and prints the API response JSON ([outbound](skills/messenger/references/omosense.md#outbound)):

```sh
bunx omosense@latest say telegram send '{"chat_id":123456789,"text":"hello"}'
```

With no bot configured and no `"bot"` override, it exits 2 with `no bot: config.json has no telegram.bot; pass {"bot":"name"} to choose one`.

### 7. Reminders

Ask the agent to remind you, or add an entry yourself. There's no CLI for it: reminders live in `.omosense/state/reminders.json`, a JSON array ([reminders](skills/messenger/references/omosense.md#reminders)).

```json
[
  {
    "id": "dentist",
    "at": "2026-10-08T09:00:00+09:00",
    "platform": "telegram",
    "target": { "chat_id": 123456789 },
    "text": "Dentist at 10"
  }
]
```

- `target` is the `say` JSON without `text`.
- `at` is best as an ISO datetime with `Z` or an offset. A naive datetime means host-local time, a date alone also works, and an unparseable `at` is sent right away.
- The host checks at start and every 20 s. A due entry goes out via `say <platform> send` and gets marked `sent`, or `failed` plus `error` (never retried), or `skipped` when it's more than 6 h late. `cancelled` is a marker you or the agent set; the host never cancels. Entries with any of these markers are left alone.
- After writing the file it prints `REMIND sent <entry json>`, `REMIND failed <entry json>` or `REMIND skipped-late <entry json>`.
- The host rewrites the whole file when it marks entries, so re-read it and write the whole array back when you add one.

### 8. Stop and resume

To stop, the agent kills its host monitor ([stop](skills/messenger/references/omosense.md#stop)), or you Ctrl-C a foreground run. State stays in `.omosense/state`. Messages that arrive while the host is off aren't received.

To resume, reopen the session in the bot folder with the same `--skill` flags and say `<NAME> mode`.

## Requirements

- [OmO](https://github.com/code-yeongyu/oh-my-openagent)
- [Herdr](https://herdr.dev). The skill installs it if missing and stops if that fails, then run `herdr integration install <agent>` for the agent you run.
- [agent-messenger](https://github.com/agent-messenger/agent-messenger) for bot creation and platform calls.
- **omosense (required).** Released on npm as [`omosense`](https://www.npmjs.com/package/omosense) (macOS and Linux, arm64 and x64). The skills call it as `bunx omosense@latest ...` (see [Install](skills/messenger/references/omosense.md#install)), so there is nothing else to install.
- Optional: [zele](https://github.com/remorses/zele) on PATH for Google calendar and mail: omosense's google source always runs and reports nothing (only a LOG error) without it, and `ffmpeg` plus `mlx_whisper` on PATH if you want voice messages transcribed (omosense runs them with `mlx-community/whisper-large-v3-turbo`; the transcriber is not configurable).

## License

MIT
