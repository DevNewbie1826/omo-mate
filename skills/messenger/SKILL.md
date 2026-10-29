---
name: messenger
description: >-
  Points this bot folder at the messenger skill that ships with agent-sense, and
  says what the sense installs. Read it before setting up a messenger agent or
  running "<NAME> mode": this repository keeps no copy of the skill. Use when
  the user asks to set up a messenger agent or chat bot friend, says
  "<NAME> mode", or asks about the messenger agent's status.
---

# Messenger

The messenger skill lives with the sense that runs it. This repository is the
bot-folder bootstrap and keeps no copy, because a second copy drifts from the
maintained one.

## Where the skill is

- In the bot folder's clone: `agent-sense/skills/messenger/SKILL.md`, with
  `setup.md`, `inbound.md`, `outbound.md`, `sessions.md` and
  `troubleshooting.md` in `references/` beside it.
- Upstream:
  <https://github.com/DevNewbie1826/agent-sense/tree/main/skills/messenger>

Read `SKILL.md` first: it is the index, and it links a reference only when that
capability is needed.

## What the sense installs

One daemon and one command line, the OmO extension that registers the
`agent_sense_*` tools (start it with `agent_sense_bridge`) and delivers ledger
events as turns, and the two skills (`messenger` and `memory-tidy`). This
repository's `README.md` has the folder layout and the OmO-specific Herdr note;
the skill's `references/setup.md` has the install, the first config, bot creation
and first contact.

## Do not

- Follow a copy of this skill that tells you to write an old sense config
  directory, to run the old npm host, or to scrape a host process: it predates
  the sense. An existing old folder converts with the sense's `migrate` verb.
- Copy the skill back into this repository. Fix it upstream instead.
