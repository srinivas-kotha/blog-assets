# What Actually Happens When You Deploy 17 AI Agents on One Server

I spent the last two weeks building an autonomous AI agent company on a $12/month VPS. Seventeen agents, a full org chart — CEO, directors, specialists — each one waking up on a schedule and working through a task queue without me touching anything.

Here's what I thought would happen: seamless automation, side projects progressing on their own, me reviewing PRs instead of writing them.

Here's what actually happened: three production incidents in the first 72 hours.

## The Setup

I'm using [Paperclip AI](https://paper.srinivaskotha.uk) — a local orchestration layer that gives Claude agents heartbeat scheduling, a shared inbox, and API-based task management. The idea is simple: you hire agents (each one a Claude Code instance with a role and instructions), assign them issues, and they wake up on a cron-like schedule to do the work.

My org chart:

- 1 CEO agent (strategy, delegation, budget oversight)
- 1 Engineering Lead + 1 Brand & Content Lead (directors)
- 15 specialist agents (frontend, backend, content writer, diagram generator, etc.)

Total: 17 agents. All running on the same Hostinger VPS with 7.8 GB RAM and no GPU.

The first heartbeat ran cleanly. The second did too. Then things got interesting.

## Incident 1: The Thundering Herd

I set the CEO to an 8-hour heartbeat and the Brand Lead to the same. Clean numbers, easy to reason about. What I didn't think about: they'd both fire at the same time, every single time.

The first synchronized heartbeat hit and subscription quota spiked. Two Opus-class agents woke up simultaneously, both read the full context, both started doing non-trivial work. The session quota hit 40% in under 5 minutes.

The fix was embarrassingly simple: stagger the intervals. CEO stays at 8 hours, Brand Lead moved to 10 hours. Now the LCM is 40 hours before they ever fire together. The triple overlap (CEO + Engineering Lead + Brand Lead) only happens every 5 days instead of every 8 hours.

The key insight: LCM scheduling is a real systems problem, not just a theoretical one. Any time you're running multiple periodic jobs, the first question should be "when do these collide?"

## Incident 2: Three Agents, One Repo, Zero Isolation

I have three agents with write access to the same repository: a Frontend Engineer, a Backend Engineer, and a Content Writer. In the first parallel run, all three woke up, pulled the repo, and started working. The Frontend Engineer created a branch, committed a change, then switched branches to verify something. While that was happening, the Backend Engineer ran `git checkout main` in the same working directory.

The Frontend Engineer's branch switch got corrupted. Neither agent knew what happened. Both posted "done" on their issues. Neither change was actually committed correctly.

The fix: every write-heavy agent now runs in a `/tmp/worktree-$AGENT_ID-$TASK` directory via `git worktree add`. Each agent gets a completely isolated copy of the repo. They create their branch, do their work, push, and clean up the worktree. No shared state, no branch collisions.

```bash
# What every write-agent now does at the start of a task:
cd /home/crawler/blog-assets
git worktree add /tmp/worktree-post-02 -b feat/post-02
cd /tmp/worktree-post-02
# ... do all work here ...
git add . && git commit -m "feat: ..."
git push origin feat/post-02
cd /home/crawler
git worktree remove /tmp/worktree-post-02
```

This pattern applies to any system where multiple processes write to the same Git repo. It's not AI-specific — it's just a distributed systems problem that AI agents make easier to run into at scale.

## Incident 3: Budget Caps Were Theater

I set spending limits on every agent: $20/month for the CEO, $10 for directors, $5 for specialists. Paperclip tracks costs and auto-pauses at 100% budget. I felt good about this.

Then I checked how the billing actually works: these agents run on Claude Code subscription, not API keys. Paperclip tracks costs internally but can't signal the subscription to pause. The auto-pause mechanism fires correctly — but only for API key consumers. Subscription users were running uncapped.

I'd built a circuit breaker with no wire connecting it to the circuit.

The real fix: hard limits baked into the agent instructions themselves. Each agent now has a `15 tool calls per heartbeat` cap. If it hits that limit, it posts a status update and exits gracefully, resuming on the next heartbeat. The limit is enforced by the agent's own instructions, not by external billing logic.

This is the broader lesson: **don't rely on external systems to enforce constraints that your code should enforce.** If you need something to stop, build the stop into the thing itself.

## What Actually Works

After those three incidents, the system has been running cleanly for a week. The Brand & Content Lead woke up this morning, checked the issue queue, found this post assigned to it, and started writing. (Yes — this post was written by the agent describing its own incident report. I've read it and it's accurate.)

The things that work well:

- **Heartbeat scheduling** for non-urgent work (content drafts, weekly status updates, code cleanup)
- **Event-triggered wakes** for time-sensitive work (PR review comments, deployment status)
- **Issue-based communication** — agents post comments, tag each other, escalate blockers without me in the loop

The things that require more thought than I expected:

- Schedule collision avoidance (use LCM math, not "they're different numbers")
- Shared resource isolation (worktrees for Git, separate DB schemas for data agents)
- Constraint enforcement at the source, not the billing layer

## One Week In

I haven't opened my task manager in four days. The agents are drafting content, reviewing PRs, flagging blockers, and updating context files. My job right now is reviewing their output and making the calls they can't make — architecture decisions, "is this good enough to ship", resolving ambiguous requirements.

That shift — from doing to reviewing — is the actual value. Not the automation itself, but the fact that my cognitive load moved from execution to judgment.

The bugs were real. The fixes were straightforward. The pattern works.

---

**Tags:** ai-agents, claude, automation, devops, side-projects
