# omo-mate

Two OmO skills that turn your agent into an always-on messenger friend: it lives in a chat bot, listens only to the people you register, does the work itself, and keeps its memory tidy.

| Skill | What it does |
| --- | --- |
| [`messenger`](skills/messenger/SKILL.md) | One-time setup (name, platform, avatar), then runs `<NAME> mode`: inbound messages, replies, work threads, work sessions, monitors. |
| [`memory-tidy`](skills/memory-tidy/SKILL.md) | Folds the memories of your other OmO agents into the messenger agent's own memory, deduplicated, with source pointers. |

omosense connects the two: one foreground process per agent session, run in that session's folder, that streams messages, calendar, session and `TIDY` events to the agent (one omosense per session, one bot per session).

## Install

```sh
omo install git:github.com/DevNewbie1826/omo-mate
```

Then start a session inside [Herdr](https://herdr.dev) and say: `set up messenger mode` (in any language). The skill asks three questions (name, platform, avatar) and does the rest.

## Requirements

- [OmO](https://github.com/code-yeongyu/oh-my-openagent)
- [Herdr](https://herdr.dev). The skill installs it if missing and stops if that fails, then run `herdr integration install <agent>` for the agent you run.
- [agent-messenger](https://github.com/agent-messenger/agent-messenger) for bot creation and platform calls.
- **omosense (required).** Released on npm as [`omosense`](https://www.npmjs.com/package/omosense) (latest `0.1.0`, macOS and Linux, arm64 and x64). The skills call it as `bunx omosense@0.1.0 ...` (pinned, because an unpinned bunx can keep running an older cached release), so there is nothing else to install.
- Optional: [zele](https://github.com/remorses/zele) on PATH for Google calendar and mail: omosense's google source always runs and reports nothing (only a LOG error) without it, and `ffmpeg` plus `mlx_whisper` on PATH if you want voice messages transcribed (omosense runs them with `mlx-community/whisper-large-v3-turbo`; the transcriber is not configurable).

## License

MIT
