# omo-mate

Two OmO skills that turn your agent into an always-on messenger friend: it lives in a chat bot, listens only to the people you register, does the work itself, and keeps its memory tidy.

| Skill | What it does |
| --- | --- |
| [`messenger`](skills/messenger/SKILL.md) | One-time setup (name, platform, avatar), then runs `<NAME> mode`: inbound messages, replies, work threads, work sessions, monitors. |
| [`memory-tidy`](skills/memory-tidy/SKILL.md) | Folds the memories of your other OmO agents into the messenger agent's own memory, deduplicated, with source pointers. |

omosense connects the two: it is the resident daemon that streams messages, calendar, session and `TIDY` events to the agent.

## Install

```sh
omo install git:github.com/DevNewbie1826/omo-mate
```

Then start a session inside [Herdr](https://herdr.dev) and say: `set up messenger mode` (in any language). The skill asks three questions (name, platform, avatar) and does the rest.

## Requirements

- [OmO](https://github.com/code-yeongyu/oh-my-openagent)
- [Herdr](https://herdr.dev). The skill installs it if missing and stops if that fails.
- [agent-messenger](https://github.com/agent-messenger/agent-messenger) for bot creation and platform calls.
- **omosense (required).** The skills call it as `bunx omosense ...`.
  > **Status:** the profile-based config these skills target is being finished in omosense now (not yet released). The npm package for `bunx omosense`, its install script and its release are **still to do**. Until they ship, the messenger skill stops at the omosense step and tells you so.
- Optional: [zele](https://github.com/remorses/zele) for Google accounts, and any voice transcriber you like.

## License

MIT
