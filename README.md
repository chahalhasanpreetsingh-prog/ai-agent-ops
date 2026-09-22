# ai-agent-ops

Notes on running LLM agents against real infrastructure: what I let them touch, how they are
stopped, what they cost, and how I tell whether they actually worked.

Everything here comes out of systems that run unattended on my homelab. The linked repos are the
implementations; this is the reasoning behind them.

## 1. An agent is the last resort, not the first

Every watchdog I run checks cheaply, then applies the deterministic fix that worked the last time
this broke, and only calls an agent when both fail. Known failures get scripted; the model is
reserved for novel ones. That keeps the cost near zero on a normal day and keeps the blast radius
small, because most incidents never reach the part of the system that can improvise.

Implementation: [self-healing-watchdogs](https://github.com/chahalhasanpreetsingh-prog/self-healing-watchdogs)

## 2. A fixed allow-list beats a clever prompt

Headless sessions run with an explicit list of permitted commands, never with permission checks
disabled. The agent gets read access, container inspection and restarts, and exactly one
privileged action. The prompt says plainly that anything else will be refused and that it should
report what it would need instead of working around the limit. A prompt is a request; an
allow-list is a boundary.

Alongside it: never delete volumes, never touch database data, never edit code or config.

## 3. Bound the run

One session at a time behind a lock file, because a fix can take twenty minutes while the timer
fires every two. A timeout on the session. A cooldown so a persistent failure cannot spend money
in a loop. One-shot execution with no background work, so nothing outlives the run that started
it.

## 4. Verify outside the agent

The agent's own summary is logged, not trusted. The watchdog re-runs its checks after the session
ends and records what it found. "The model said it fixed it" is not a state; a health check
returning 200 is.

This matters more than it sounds. A morning-music routine reported success for weeks while the
room was silent, because it checked that the player process existed rather than that the speaker
was the active output device.

## 5. Give agents structured access, not shell access

MCP is the difference between an assistant that can run commands and one that can call
operations you defined, with arguments you validated and actions you chose to expose. Two of mine:

* [saavn-mcp](https://github.com/chahalhasanpreetsingh-prog/saavn-mcp): one tool, one job, errors
  returned as text an agent can relay
* [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) fork: added the
  write tools (`create_post`, `like_post`), each gated behind an explicit confirm flag so an agent
  cannot post by accident

Confirm flags on write tools are the MCP equivalent of an allow-list.

## 6. Watch the cost model before you schedule anything

Anything that runs on a timer multiplies. Scheduled workflows here run through an existing Claude
subscription via the CLI rather than per-token API billing, which turned a recurring cost into a
fixed one. Where the API is the right answer, the work is batched and cached instead of being
called per item.

Implementation: [linkedin-operator](https://github.com/chahalhasanpreetsingh-prog/linkedin-operator)

## 7. Let the agent run where the work is

Driving a Windows machine over SSH from Linux means session 0, three layers of quoting, and jobs
that die with the connection. Delegating to an agent that already runs on that machine, or to a
small authenticated HTTP agent, removes an entire class of failure.

Implementation: [windows-node-agent](https://github.com/chahalhasanpreetsingh-prog/windows-node-agent)

## 8. Persist what the agent learned

Long-lived automation needs state that outlives a context window: what ran, what it found, what
was already tried. Mine keep small JSON or SQLite state files and append to journals, so the next
run starts from evidence rather than from scratch.

## What I use

Claude Code (interactive and headless `-p` sessions), the Model Context Protocol over stdio and
streamable HTTP, OpenClaw as a chat gateway into these systems, Codex on the Windows box for
local execution, and systemd timers or cron as the scheduler underneath all of it.
