# Work sessions: open, message, track, close

This page holds the concrete procedures behind SKILL.md's Sessions rules. In it `<session folder>` is the messenger agent session's working directory, and every omosense command is written `bunx omosense@0.1.0 ...`, run from that folder (see [omosense.md](omosense.md)).

## Opening a work session

The standard is a webchat session tracked by rpc. omosense's rpc source sees its completions, so the done claim below runs on its own trigger.

1. Connect to the multi-session host socket. Its path is `$OMOSENSE_RPC_SOCK`, else `$OMO_CODING_AGENT_DIR/rpc/rpc.sock`, else `~/.omo/agent/rpc/rpc.sock`. Framing is one LF-terminated JSON object per request, and each answer is one response carrying the request's `id`. senpi's `docs/rpc.md` is the reference for the socket.
2. Send `open_session` with the project folder and `retain_on_disconnect: true`, so the session outlives the connection that opened it:

   ```json
   {"id": "o1", "type": "open_session", "cwd": "/abs/path/<project>", "retain_on_disconnect": true}
   ```

   `provider`, `modelId`, `kind` and `auto_title` are optional fields of the same request. The response carries the new session's `sessionId`.
3. Right after, name it with the job name. On the multi-session host every session command carries `sessionId`, the one from the `open_session` response:

   ```json
   {"id": "n1", "type": "set_session_name", "sessionId": "<sessionId>", "name": "<job name>"}
   ```

   Don't skip this. An unnamed session shows as "New session" in the webchat session list, and nobody can tell the jobs apart. A blank name is refused.
4. Find the session with `omo thread list --all-scope --json`. Match it by `cwd` and name, and take its `sessionId` and `durableSessionId`; its `status` is `live`. Without `--all-scope`, a session in another cwd isn't listed.

   In every `omo thread` command below, `<target>` is the session's `thread_id` (its `durableSessionId`) or its unique session name. The routing handle in `sessionId` (like `rpc-6`) isn't a valid target. Pass `--all-scope` on `send` and `read` too: without it, a session in another cwd fails with `scope_denied`.
5. Register it in `threads.json` (see [Registry](#registry)) with `state` `working`.
6. Hand off the brief with `omo thread send <target> "<brief>" --idempotency-key <msg-id> --all-scope`, then wait for its ACK as in [Messaging a session](#messaging-a-session). The brief must ask the session to end with a final message listing its evidence: the merged PR, the pushed commit SHA.

Herdr fallback: some work runs as an agent started in a herdr tab. Run `herdr integration install <agent>` once, then `herdr tab create --cwd <project> --label <job> --no-focus` and `herdr pane run <pane> '<agent command>'`. rpc doesn't track that completion, so the messenger session runs the same done verification itself.

## Messaging a session

Every message carries a fresh `<msg-id>` and tells the receiver to answer `ACK <msg-id>`. Until that ACK shows up, the message is pending. Exit 0, echoed input and state changes aren't proof. Never resend blindly; read first.

Webchat session:

1. `omo thread send <target> "<text>" --idempotency-key <msg-id> --all-scope`. Pass the text as one argument.
2. `omo thread read <target> --limit <n> --all-scope`, repeated until the receiver's reply contains `ACK <msg-id>`.

herdr pane:

1. `herdr pane process-info --pane <id>`. The foreground process must be the agent, not a shell.
2. `herdr pane read <id> --source recent-unwrapped --lines 40`. The input line must be empty, with no approval or question UI open.
3. `herdr pane send-text <id> "<text>"`, then `herdr pane send-keys <id> enter` once.
4. `herdr pane wait-output <id> --match "ACK <msg-id>" --timeout <ms>`. On a timeout, read the pane again before deciding anything.

## Registry

The registry is `<session folder>/.omosense/state/threads.json`: an object keyed by thread key (an array is also accepted, the index being the key). omosense reads a few fields. Everything else belongs to the agent, and omosense ignores fields it doesn't read, so the additions below are safe.

Read by omosense:

| Field | Used for |
| --- | --- |
| `pane` | herdr: the pane to watch for job done. A falsy value is skipped. |
| `machine` | herdr: the machine of that pane. Missing or null means `local`. |
| `session_id`, `session`, `durable_session_id` | rpc: matched against the session's durable id, or its session id when `cwd` equals the session's cwd. |
| `cwd` | rpc: the session's working folder, used in that match. |

Agent-owned:

| Field | Meaning |
| --- | --- |
| `platform`, `chat_id` / `channel_id`, `thread_id` | Where the job was asked for and where to report back. |
| `name`, `brief` | Job name and the brief sent. |
| `tab` | herdr tab, for herdr work. |
| `started`, `closed` | Timestamps. |
| `note` | Free text, including the verified evidence. |
| `state` | `working`, `done-claimed`, `not-done` or `closed`. |
| `history` | Array of state changes, each `{at, from, to, reason, evidence?}`. |

`state` transitions:

- `working` to `done-claimed`: its completion arrives in a done batch.
- `done-claimed` to `closed`: the evidence was verified and the completion acked.
- `done-claimed` to `not-done`: verification found a problem; `reason` lists it.
- `not-done` to `working`: the reasons were sent and the session is back at work.

Every change appends one `history` item. Example:

```json
{
  "<thread-key>": {
    "session_id": "<durableSessionId>",
    "cwd": "/abs/path/<project>",
    "platform": "discord",
    "channel_id": "<channel-id>",
    "name": "<job name>",
    "brief": "<brief>",
    "started": "<ISO time>",
    "note": "",
    "state": "working",
    "history": [
      {"at": "<ISO time>", "from": null, "to": "working", "reason": "opened"}
    ]
  }
}
```

## Live rebuild

The file can drift from reality. On every "`<NAME>` mode" re-entry, and whenever a report looks wrong, check each entry against live state:

- `herdr pane list` and `herdr pane process-info --pane <id>`: the pane and tab still exist and the agent is alive.
- `omo thread list --all-scope --json`: the `session_id` and `cwd` are still live.
- `bunx omosense@0.1.0 rpc pending`: a pending completion whose entry is still `working` means set it to `done-claimed`.

Report the drift to the owner. Never delete entries. Write every correction as a `history` item.

## Done claims

Who: the messenger session. Timing: the done batch's 5-minute quiet window is the automatic trigger.

omosense batches done completions and sends one message to the subscribed session once 5 minutes pass with no new completion. For each entry in the batch:

1. Set `state` to `done-claimed`.
2. Read the evidence the work session left in its final message with `omo thread read <target> --all-scope`: the merged PR, the pushed commit SHA.
3. Verify it live, for example `gh pr view <n> --json state,mergeCommit` or `git branch -r --contains <sha>`.
4. Record what you found in the entry's `note` and in the `history` item.
5. OK: run the entry's ack command as written, set `state` to `closed`, and close the session. For a webchat session, send `close_session` on the rpc socket:

   ```json
   {"id": "c1", "type": "close_session", "sessionId": "<sessionId>"}
   ```

   The host refuses it with `unknown_session` when your connection never attached to that handle. Attach first with `open_session` and the session's `sessionPath`.
6. Problem: set `state` to `not-done`, message the work session with the reasons (see [Messaging a session](#messaging-a-session)), and don't ack.

No script automates this. It works only if each brief asks the work session to end with a final message listing its evidence.

## Response sessions

`<session folder>/.omosense/state/sessions.json` lists the named response sessions, each `{"pane": "<herdr pane id>", "cwd": "<folder>"}`:

```json
{
  "<name>": {"pane": "<herdr pane id>", "cwd": "/abs/path/<folder>"}
}
```

On every "`<NAME>` mode" entry, a response session registers itself. It reads its own pane id from `herdr pane current` (`result.pane.pane_id`) and its cwd, then merges its own entry into the file without replacing the others.

omosense 0.1.0 reads only the `family` entry's `pane`, and leaves that pane out of herdr job-done events. So a second always-on response session in the same herdr is registered as `family` in the herdr-enabled session's `sessions.json`.

## herdr commands

| Command | Use |
| --- | --- |
| `herdr integration install <agent>` | Install herdr's integration for an agent (pi, omp, claude, codex, ...). |
| `herdr integration status` | Show which integrations are installed. |
| `herdr pane current` | This pane's id (`result.pane.pane_id`). |
| `herdr pane list` | All panes. |
| `herdr pane process-info --pane <id>` | Foreground process of a pane. |
| `herdr pane read <id> --source recent-unwrapped --lines <N>` | Recent pane text. |
| `herdr pane send-text <id> "<text>"` | Type text into a pane. |
| `herdr pane send-keys <id> enter` | Press keys in a pane. |
| `herdr pane wait-output <id> --match "<text>" --timeout <ms>` | Wait for text to appear. |
| `herdr pane run <id> '<command>'` | Run a command in a pane. |
| `herdr tab create --cwd <path> --label <text> --no-focus` | Open a tab without stealing focus. |
| `herdr tab close` | Close a tab. |
| `herdr --skill` | Print herdr's own agent skill. |
