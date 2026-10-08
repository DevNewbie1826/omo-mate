---
name: memory-tidy
description: >-
  Tidies the messenger agent's memory hub. It folds the memories of the user's other OmO agents into the agent's own memory repo, deduplicated, with source pointers, and cleans out stale or wrong records. It runs on an omosense `TIDY {...}` event, or when the user asks to "tidy memory" (in any language). The agent's own repo is never a source, and omosense's `tidy.enabled`, `tidy.learnOthers` and `tidy.exclude` decide what is read. Use on a TIDY event or when the user asks for a memory tidy.
---

# Memory tidy

This skill gathers people, decisions and facts from the user's other agents into summaries in the messenger agent's own memory, and cleans out stale or wrong records. Source memories are read-only. Only the agent's own memory is written. It is the companion of the `messenger` skill, and omosense connects the two (see [omosense](../messenger/references/omosense.md)).

## Settings (from the session folder's omosense config)

omosense reads these from the messenger session folder's `.omosense/config.json` (see [config](../messenger/references/omosense.md#config)).

| Key | Effect |
| --- | --- |
| `memory` | The target: this agent's own repo id (the `AGENT_ID` in its system prompt's `<memory_metadata>`), under `~/.omo/memory/agents/<id>/repo`. |
| `tidy.enabled` | When false, there is no `TIDY` source and manual commands print `LOG tidy disabled`. Do nothing in that case. |
| `tidy.learnOthers` | When false, no other repo is read at all. |
| `tidy.exclude` | Repo ids that are never read (for example another person's agent). |

**Hard rule:** the target repo (`memory`) is never a source, whatever the settings say. Exclusions apply before `learnOthers`. omosense applies all of these when it builds the changed set. If a listed repo breaks a rule anyway, skip it and report it.

## Sources (read-only)

- Every `~/.omo/memory/agents/*/repo` that omosense reports in the changed set.
- OmO's file tools refuse other identities' memory roots. Read sources through the shell only (`cat`, `ls`, `rg`, `git -C <repo> log|show|diff|rev-parse|ls-files|status`).
- **Never** write, commit, checkout, reset, stash or `git gc` a source repo.

## Target (written only through the memory tool)

Every change goes through the `memory` tool, which commits it. Use `apply_patch` for multi-file changes and put the reason in `reason`. Never run raw git on the target.

| What | Where |
| --- | --- |
| One summary per project: status, dated key decisions, durable facts, lessons, original repo and cwd | `projects/<source-name>/summary.md` (frontmatter `description: "Project summary: <name> (<cwd>) - <topic>"`) |
| Name, cwd and summary map, plus one-line entries for one-off sources | `projects/INDEX.md` |
| Facts about a named person other than the user | `people/<slug>/card.md` (stable) + `people/<slug>/observations.md` (dated) |
| The user's identity (stable lines, kept short because it goes into every prompt) | `system/human.md` |
| The user's dated preferences and observations | `people/human/observations.md`. The user is always `human`. Record their names and handles as aliases and never give them a separate `people/<name>/` dir. |
| Durable facts that span projects (machine, accounts, tools) | `notes/facts/<YYYY-MM>.md` |
| Rules the user set | Not written by this skill. The messenger agent records rules itself when the user states them. |

Each fact has one home. Entries use the form `- [YYYY-MM-DD] <content> <!-- src: <pointer> -->`. Every new or edited line MUST end with `<!-- src: <source-repo>:<path>@<short-sha> -->` (or `<!-- src: projects/<name>/summary.md -->` for references inside the hub). A line without a pointer is not allowed.

## Rule scope

Every rule, preference or decision line carries `[scope: global]` or `[scope: <project-name>]` before its `src:` pointer.

- Project-scoped rules live only in `projects/<name>/summary.md` under `## Scoped rules`.
- A line goes global only when the user said it applies everywhere, or the same rule shows up in two or more projects. Its `src:` cites that evidence. When unsure, keep it project-scoped.
- When projects disagree, never merge or overwrite. Keep each rule in its own project and add one line to `people/human/observations.md`: `- [date] <topic> differs by project: A ... (projects/A/summary.md), B ... (projects/B/summary.md) [scope: global] <!-- src: ... -->`.
- Dedupe merges only lines with the same scope.

## Procedure

0. **Run in a write-capable background worker.** The worker must use the memory tool and run omosense. Use a general work category, never a read-only explore agent. Config and watermark are per folder, so the worker runs every omosense command from the messenger agent's session folder (`cd <session folder> && ...`). The lead passes that path in the worker prompt.
1. **Get the changed set.** One path covers both cases:
   - Automatic: the `TIDY {"changed":[{"repo","from","to"}]}` line.
   - Manual: `bunx omosense@latest tidy --now` runs the same check once and prints the same line, or nothing when nothing moved. Nothing moved means there is nothing to do.
   - Process only the listed repos: `git -C <repo> diff --stat <from> <to>`, then the file diffs. When `from` is null (a first run or a new repo), read the whole tree.
2. **Classify each source.** A template-only repo gets one line in INDEX under Skipped. One-off or scratch sources (temporary worktrees, single reviews, anything under a tmp dir) get one line under One-off (`- name | cwd | outcome`) or are dropped, and get no project dir. A durable preference or person fact in them still goes to its people/ home. Real projects get a summary.
3. **Distill.** Merge new information into the existing summary: update Status, add decisions and facts, and collapse per-session notes into outcomes and lessons. Drop progress logs. Keep each summary readable at a glance (about 6 KB at most). For large sources, fan out read-only helpers that write drafts to a temp dir, then integrate the drafts yourself.
4. **Route people, facts and preferences** to their homes, following Rule scope. Search the target first. Merge into an existing line (newest date, combined `src:`) instead of appending a duplicate.
5. **Fix stale or wrong records** when newer evidence contradicts them, and give the reason in the memory-tool `reason`. A contradiction about a person goes under `## Contradiction` in their observations. Leave the card alone.
6. **Secrets.** Never copy tokens, keys, passwords or OAuth secrets. Write `[REDACTED]`.
7. **Self-dedupe** the target on every run: merge same-meaning or superseded lines in `people/human/observations.md`, `notes/facts/*.md` and `people/*/`, and fold any `people/<slug>-N/` duplicate dir into `people/<slug>/`. Keep each card's description exactly `Person - <Name>`, with names and handles in `aliases`.
8. **Watermark.** After the target writes are committed, run `bunx omosense@latest tidy --write-watermark <repo>=<to> ...` for exactly the repos you processed. It logs `LOG memory-tidy watermark set <n> repos`. The no-argument form marks every current source head as processed. Use it only when every listed repo was done and nothing newer landed. omosense owns the watermark file (`<session folder>/.omosense/state/memory-tidy.json`). Do not edit it by hand.
9. **Verify.** `git -C <each source> status --porcelain` is unchanged. Every `src:` pointer you touched resolves: `git -C ~/.omo/memory/agents/<repo>/repo cat-file -e <sha>:<path>` exits 0, or the file exists inside the hub. Report any unresolved pointer with its repo, path, sha and error.

## Getting the detail

The summary is an index, not the record. For a detail question, open the `src:` pointer with `git -C <repo> show <sha>:<path>`. If there is no pointer, search the sources read-only (`git -C <repo> log -S <term> --all -p`). If a repo is gone, clone its bundle from `~/.omo/memory-backups/<date>/<name>.bundle` into a temp dir and search there. `bunx omosense@latest tidy --backup-now` makes today's backup once into `~/.omo/memory-backups/<date>/`. Backups cover every agent repo, excluded ones included. Say which source and version the answer came from.

## Report

Return one short line to the messenger agent, in the user's language, saying what was merged, fixed or removed. If nothing changed, return nothing. The messenger agent relays it only when the user asked or the change is worth surfacing.
