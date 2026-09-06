# Herdr fork: agent-to-agent messaging for every agent kind

Read this file first when opening this repository. It is the orientation for
any LLM or human starting work here. Last updated 2026-09-04.

## What this repository is

- A personal development fork of [herdrdev/herdr](https://github.com/herdrdev/herdr),
  a Rust terminal multiplexer built for hosting AI coding agents. It is not the
  upstream project and its owner is not an upstream maintainer.
- Forked from upstream `master` at v0.8.2 (commit `3150bd92`) on 2026-09-04.
- Git layout:
  - remote `upstream` = herdrdev/herdr, fetch only. Pushing is disabled with a
    `no_push` URL on purpose.
  - branch `master` mirrors `upstream/master`. Keep it clean; never commit to it.
  - branch `dev` is where fork work happens. Rebase or merge `upstream/master`
    into it periodically.
  - no fork remote (`origin`) existed when this file was written. Add one when
    the owner decides between a public GitHub fork and a private repository.
- A pristine, untouched clone of upstream lives next door at `../herdr`. Use it
  as a read-only reference when you need to compare against unmodified code.

## Ground rules for agents working here

1. Never push to `upstream`, never open pull requests or issues against
   upstream from this work, and never run upstream release tooling. Upstream
   closes unsolicited PRs automatically; see `CONTRIBUTING.md`.
2. The engineering rules in `AGENTS.md` still apply to code written here:
   state separated from runtime, pure render, no god objects, platform code
   isolated under `src/platform/`, evidence-based screen detection, the
   runtime/client boundary, the multiplicative performance paths, the stable
   endpoint contract, no `unwrap()` in production code, `tracing` for logs,
   `just` recipes for tests. Sections about maintainer workflow, Can's local
   machine, release channels, and the external contributor guardrail describe
   upstream's process and do not govern this fork.
3. Stay upstream-syncable. Prefer additive modules, new API methods, plugins,
   and new files over edits to existing upstream files. Do not touch
   `docs/preview/`, `docs/versions/`, `distribution/`, root `CHANGELOG.md`, or
   `skills/herdr/SKILL.md` unless the change is deliberately fork-specific.
4. Use lowercase conventional commit subjects. Propose the commit message
   before committing. Do not commit or push unless asked.
5. When testing a fork build from inside a running Herdr session, clear the
   inherited socket overrides so the debug binary talks to its own `herdr-dev`
   server and not the installed stable one:

   ```bash
   env -u HERDR_SOCKET_PATH -u HERDR_CLIENT_SOCKET_PATH cargo run -- <command>
   ```

   For experiments that need real panes and agents, use a disposable named
   session as described in `.agents/skills/herdr-throwaway-repro/SKILL.md`.
   Never run `herdr server stop` against the owner's main session.

## The goal

Agents running in the same Herdr space should be able to find each other, ask
each other questions, consult, hand off work, and operate as a swarm when
needed.

Hard requirements:

- Works for every agent kind Herdr recognizes (23 kinds in
  `src/detect/mod.rs`: Claude Code, Codex, OpenCode, Pi, Gemini, Cursor, Grok,
  Copilot, Kimi, Droid, Amp, Hermes, and the rest). Claude-only channels are
  not acceptable as the primary mechanism.
- Reliable delivery: a message must not be lost because the target was
  mid-turn, and the sender should learn whether it was delivered.
- Sender attribution: a receiving agent must be able to tell a peer message
  from the human operator's instruction, and must treat peer messages as
  requests rather than commands with operator authority.
- Discovery by role or name within a space, not by pane ID alone.
- Large payloads travel as files; messages carry pointers.
- The human keeps full visibility and can intervene from the Herdr UI.

## What Herdr already provides (verified against v0.8.2)

Herdr never talks to the agent process. It talks to the terminal the agent
lives in, which is why its primitives are agent-agnostic.

| Need | Existing primitive | Where |
| --- | --- | --- |
| Discover peers | `herdr agent list`, `agent get`, `agent rename <target> <name>` | `agent.list`, `agent.get`, `agent.rename` |
| Publish short status | `herdr pane report-metadata --token key=value` (up to 32 keys, 80 chars each, optional TTL) | `pane.report_metadata`, `workspace.report_metadata` |
| Send text to an agent | `herdr agent prompt <target> "<text>" [--wait --until <state>]` | `agent.prompt` |
| Wait for a lifecycle state | `herdr agent wait <target> --until idle|done|blocked` | `agent.wait` |
| React to state changes | `events.subscribe` with `pane.agent_status_changed`, `pane.agent_detected`, `pane.output_matched` | `src/api/subscriptions.rs`, `src/api/event_hub.rs` |
| Read output back | `herdr agent read <target> --source recent-unwrapped --lines N` | `agent.read`, `pane.read` |
| Start a helper | `herdr agent start <name> --kind <kind> --pane <id>` | `agent.start` |
| Teach an agent all of this | `herdr --skill` prints `skills/herdr/SKILL.md`; `npx skills add herdrdev/herdr --skill herdr -g` installs it into 77+ agents | `docs/next/website/src/content/docs/agent-skill.mdx` |
| Plugin hooks | `herdr-plugin.toml` with `[[events]]`, `[[actions]]`, `[[panes]]`; the whole CLI is the plugin API | `docs/next/website/src/content/docs/plugins.mdx` |

Agent lifecycle states are `idle`, `working`, `blocked`, `unknown`, plus a
derived `done` (idle and not yet seen by the user). Per-kind reliability of
that state is what decides delivery timing:

| State authority | Kinds |
| --- | --- |
| Full lifecycle hooks when the integration is installed | Pi, OMP, OpenCode, Kilo, Kimi, MastraCode |
| Screen manifest for state; integration adds session identity only | Claude Code, Codex, Cursor, Copilot, Devin, Droid, Qoder, Qwen, Hermes, Antigravity, Grok |
| Screen manifest only, no integration | Amp, Kiro, Maki, Muse |
| Detected but less tested | Gemini CLI, Cline |

Install integrations with `herdr integration install <kind>`. None were
installed on the owner's machine on 2026-09-04.

## What is missing (the gap this fork exists to close)

- `agent prompt` types into the target immediately regardless of state. If
  the target is mid-turn the text lands inside a running turn and is lost.
  No queue, no retry, no delivery acknowledgement.
- No sender attribution. Upstream discussion 2180 documents a worker treating
  another agent's text as the operator's instruction.
- The obvious client-side workaround (`agent wait --until idle` then
  `agent prompt`) races: two senders waiting on one target fire together.
- No discovery by role. Names exist but are cleared when the agent exits.
- No shared convention for envelopes, reply channels, or payload files.
- All 111 method and event names in `docs/next/api/herdr-api.schema.json`
  were checked on 2026-09-04: there is no `agent.send`, inbox, or queue method.

## Prior art and references

Upstream:

- Discussion 741, "Inter-agent communication & @agent task delegation"
  (June 2026): proposes `@agent` syntax and Google A2A integration. Idea
  stage, no maintainer commitment.
  <https://github.com/herdrdev/herdr/discussions/741>
- Discussion 2401, "agent.send: queue messages server-side and deliver when
  the target agent is idle" (August 2026): the smallest useful primitive,
  FIFO queue, TTL, `--from` attribution, queued/delivered/dropped events.
  The author runs a private implementation against 0.8.0. Idea stage.
  <https://github.com/herdrdev/herdr/discussions/2401>

Outside Herdr:

- Claude Code cross-session messaging and agent teams: native, works between
  local Claude sessions over per-session sockets, verified live from a Herdr
  pane. Claude-only. <https://code.claude.com/docs/en/cross-session-messaging.md>
- AMQ, agent-message-queue: Maildir mailbox, no daemon, skill-based, supports
  Claude, Codex, Grok, Cursor, Pi. Its wake step is a terminal notification and
  does not know whether the agent is busy.
  <https://github.com/avivsinai/agent-message-queue>
- cross-agent-teams (xats): SQLite daemon exposed over MCP, wakes Claude
  natively and pastes into other agents via tmux.
  <https://jtianling.com/en/cross-agent-teams-release.html>
- Herdr plugins closest to the space: herdr-dagr (swarm as a DAG from an
  orchestrator's run file), herdr-board (kanban cards dispatched as prompts),
  herdr-lantern (human-facing status dashboard). None is a message bus.
- agent-toolkit SWARM_HERDR: uses Herdr purely as UI with state and handoffs
  on disk. <https://github.com/ulises-jeremias/agent-toolkit/blob/main/docs/SWARM_HERDR.md>

## Design direction (current hypothesis, not yet built)

A "postmaster" that owns delivery, built on Herdr's existing primitives so it
works for every agent kind:

- Mailbox: durable, per session, scoped by workspace. Files or SQLite; AMQ's
  Maildir format is a candidate to reuse rather than reinvent.
- Send: a CLI the agent runs from its pane, `send --to <name|role> --from
  <self>`, taught through the skill file. Every agent can run a shell command,
  which is the only sender capability required.
- Deliver: the postmaster subscribes to `pane.agent_status_changed` and
  injects through `agent prompt` only after the target has been idle for a
  stable window. Before injecting, read the detection snapshot and skip
  anything that looks like a question or approval dialog. Herdr already
  refuses to type into a `blocked` agent. FIFO per target, bounded, TTL,
  delivery acknowledgement back to the sender's mailbox.
- Envelope: plain text, because a text input box is the only thing all agent
  kinds have in common. Header carries sender name, kind, pane, a
  request-not-command marker, and either the body or a payload file path.
- Discovery: `agent list` filtered to the workspace plus metadata tokens for
  role and current task, refreshed by each agent.
- Where it lives: first as a plugin or background pane outside the server,
  which needs no core change. Move the queue into the server (a real
  `agent.send` method, as in discussion 2401) only if the plugin approach
  proves too racy or too slow.

Known constraints to state up front:

- No CLI agent can be interrupted mid-turn through the terminal. Messages
  wait for the turn to end. Only Claude's private channel can interrupt, and
  that is out of scope as a primary path.
- Full-screen agents (Claude Code, OpenCode) render history on the alternate
  screen, so reading a long reply back through `agent read` is lossy. Replies
  go to files.
- Each agent kind needs one verified send-and-read round trip before it is
  trusted in the swarm; multi-line input handling differs between kinds.
- Shared-code conflicts are not solved by messaging. Herdr's worktree-per-
  workspace model is the natural answer and should be part of the design.

## Map of the upstream code that matters for this work

Data model, from the actual structs:

```
Session (one server + socket)
└── Workspace ("Space" in the sidebar)     src/workspace.rs        id w1
    └── Tab (one split layout)             src/workspace/tab.rs    id w1:t1
        └── Pane (a viewport slot)         src/pane/state.rs       id w1:p1
            └── Terminal (PTY + process)   src/terminal/state.rs   id term_...
                └── Agent = detected state on the terminal, not a container
```

- `src/api/`: socket API. `schema.rs` and `schema/` define request and
  response types (the JSON schema is generated into
  `docs/next/api/herdr-api.schema.json`). `server/` dispatches methods,
  `wait.rs` implements agent and output waits, `subscriptions.rs` and
  `event_hub.rs` implement event streams.
- `src/app/api.rs` and `src/app/agents.rs`: how the app state answers API
  calls, including prompt submission and agent resolution by name.
- `src/terminal/state.rs`: `TerminalState` holds `detected_agent`, `state`,
  hook authority, metadata tokens, and session references.
- `src/detect/` and `src/detect/manifests/*.toml`: per-kind screen detection.
  `herdr agent explain <target>` shows why a pane has its state.
- `src/integration/`: per-kind hook installers under `assets/`.
- `src/protocol/`: wire format and frozen endpoint contracts. Do not change
  generation-1 codecs; add new methods and optional fields instead.
- `src/cli/agent.rs`, `src/cli/pane.rs`: CLI wrappers over the socket API.
- `skills/herdr/SKILL.md`: what agents are taught today.
- `docs/next/website/src/content/docs/`: `agent-automation.mdx`,
  `socket-api.mdx`, `agents.mdx`, `integrations.mdx`, `plugins.mdx` are the
  authoritative unreleased docs.

## Environment and build notes

- Toolchain pinned by `rust-toolchain.toml`: Rust 1.96.1 with clippy and
  rustfmt.
- `build.rs` compiles the vendored Ghostty terminal core with Zig and links it
  statically. Zig 0.15.2 is mandatory. On macOS: `brew install zig@0.15`.
- Tests and checks run through `just` (`just test`, `just check`) and need
  `cargo-nextest`, `bun`, and `python3`. Install: `brew install just`,
  `cargo install cargo-nextest`.
- Machine state on 2026-09-04: Rust, bun, node, python3 present; Zig, just,
  cargo-nextest missing; only about 7.6 GiB free on the data volume. A debug
  `target/` directory for this project is several GiB. Free space before the
  first build.
- The owner runs Claude Code inside a Herdr pane on the installed stable
  Herdr 0.8.2 server (`HERDR_ENV=1`, `HERDR_SOCKET_PATH` set). Rule 5 above
  applies to every `cargo run`.

## Status log

- 2026-09-04: fork created from upstream v0.8.2. Research on prior art and
  existing primitives done and recorded above. No code written yet. Open
  decisions: fork remote (public fork vs private repo), whether to prototype
  with AMQ as the mailbox before writing anything, and the envelope format.

## Suggested first steps

1. Free disk space, install Zig 0.15.2, `just`, and `cargo-nextest`, and get
   `just test` green on an unmodified `dev` branch.
2. Install Herdr integrations for the agent kinds in daily use so idle
   detection is hook-backed where possible.
3. Prototype the postmaster as a plugin or script against the installed
   Herdr, using a disposable named session and two cheap agents. Measure how
   often idle-gated delivery lands cleanly per kind.
4. Only then decide whether anything must move into the server.
