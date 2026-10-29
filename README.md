# omo-mate

The bot-folder bootstrap for the messenger skill that ships with
[agent-sense](https://github.com/DevNewbie1826/agent-sense), plus the OmO
additions the sense cannot carry itself.

This repository used to hold its own copy of the messenger skill and its own
runtime notes for `omosense`. Both now live with the sense that runs them, so
this repository points there instead of keeping a copy that drifts. What is left
here is the pointer, the OmO-specific Herdr note, and a `memory-tidy` copy the
package manifest registers.

## Where the messenger skill lives

| What | Where |
| --- | --- |
| The skill | `agent-sense/skills/messenger/SKILL.md`, in the clone inside the bot folder |
| Upstream | <https://github.com/DevNewbie1826/agent-sense/tree/main/skills/messenger> |
| Setup, inbound, outbound, sessions, troubleshooting | `references/` beside that file |

Read the skill from the clone, and read `SKILL.md` first: it is the index and
links a reference only when that capability is needed. A copy that tells you to
write an old sense config directory, to run the old npm host, or to scrape a
host process is older than the sense and must not be followed; a folder those
copies already set up converts with the sense's `migrate` verb.

## One bot, one folder

A messenger bot means one bot, one folder: one daemon, one config, one memory
repository. A global install would put the skill descriptions into every OmO
session you open, which wastes tokens, and it leaves room for a session that
never meant to make a bot to try making one. With a folder install, only the
session opened in that folder sees the skills. Removing a bot means deleting its
folder; updating means a `git pull` in the clone.

```text
~/bots/<bot-name>/          bot folder = the agent session's working folder
├── agent-sense/            the sense: daemon, command line, extension, skills
└── .agent-sense/
    ├── config.json         the sense config
    └── state/              ledger, cursors, thread book, alarms, watches
```

## Set up a bot folder

Clone the sense into the folder, then follow its own setup document. That
document owns the install, the first config and the first check, so this page
does not repeat them.

```sh
mkdir -p ~/bots/<bot-name> && cd ~/bots/<bot-name>
git clone https://github.com/DevNewbie1826/agent-sense.git
```

Follow
[`skills/messenger/references/setup.md`](https://github.com/DevNewbie1826/agent-sense/blob/main/skills/messenger/references/setup.md)
in that clone: it builds or installs the binary, installs the extension, writes
the first config, runs the sense's own check, starts the daemon as its own
service, and creates the bot.

Then open the session in the bot folder - not in the clone - and hand it the
skill:

```sh
omo
```

In that session, start the bridge (`agent_sense_bridge` with `start`), read
`agent-sense/skills/messenger/SKILL.md`, and follow it from there. That first
read is the entry point: the skill's own setup section takes over from the
config onward, and every later session opened in this folder picks up the same
file. To register the skills for `/skill:` discovery instead, open with
`omo --skill agent-sense/skills/messenger --skill agent-sense/skills/memory-tidy`.

Two things are OmO's rather than the sense's, and are below: where OmO loads an
extension from, and the Herdr agent-state integration.

## Herdr for OmO

Without Herdr's agent-state integration, Herdr cannot detect an OmO pane's agent
state: `herdr agent explain <pane>` may show `rule: none` and fall back to idle,
and `herdr agent prompt <pane> <text> --wait` can then time out even though the
prompt was delivered. The integration gives the pane
`full_lifecycle_hook_authority`, which the default `--wait` needs.

OmO loads extensions from `~/.omo/agent/extensions/`, not
`~/.pi/agent/extensions/`. Herdr has no `omo` install target, but OmO uses the
senpi/pi extension API, so generate its `pi` integration in a temporary home and
copy the file:

```sh
SB=$(mktemp -d)
mkdir -p "$SB/.pi/agent/extensions"
HOME="$SB" herdr integration install pi
mkdir -p ~/.omo/agent/extensions
cp "$SB/.pi/agent/extensions/herdr-agent-state.ts" ~/.omo/agent/extensions/
rm -rf "$SB"
```

Restart OmO sessions to load it; a running session does not pick it up. Check
that `herdr-agent-state.ts` appears under `[Extensions]` in the OmO start banner
and that `herdr agent explain <pane>` shows
`screen_detection_skip_reason: full_lifecycle_hook_authority`. The file is
managed by Herdr and may be overwritten on reinstall, so install it once per
machine or container user. To remove it, delete
`~/.omo/agent/extensions/herdr-agent-state.ts`.

## Memory tidy

Memory tidying is a separate job with its own skill: a tidy event means another
agent's memory repository moved, and the job folds that into the messenger
agent's own memory. agent-sense ships the maintained copy at
`skills/memory-tidy/SKILL.md` in the clone; this repository also carries
`skills/memory-tidy/SKILL.md`, registered by `package.json`, and the sense's copy
is the one to follow.

## Update

From the bot folder:

```sh
git -C agent-sense pull
```

Rebuild the binary if it came from source, then restart the daemon and the
session. The extension and the skills are read from the clone, so the pull is
the update for them too.

## Remove

Stop the daemon (the sense's own stop verb), then delete the bot folder. These
live outside it and stay behind:

- bot credentials in the credential store the config names
  (`agent-messenger` by default)
- the agent's memory repo, `~/.omo/memory/agents/<id>/repo`
- the platform's own bot account

The extension was registered in the folder's project settings, so deleting the
folder removes the registration with it. Installed it user-wide instead? Remove
it with `omo remove agent-sense/agent/pi`.

## Requirements

- [OmO](https://github.com/code-yeongyu/oh-my-openagent)
- [Herdr](https://herdr.dev), for the panes the sense reads and drives, plus the
  agent-state integration above for OmO panes
- [agent-sense](https://github.com/DevNewbie1826/agent-sense), with the Go
  version its `go.mod` pins for a build from source; a release asset needs no
  toolchain
- a bot on Telegram or Discord, and its token in the credential store
- optional: `zele` on `PATH` for the calendar and mail source
- optional: a transcriber argv in the platform section if voice messages should
  be transcribed

## License

MIT
