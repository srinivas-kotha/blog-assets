# How I Built a Personal AI Agent Network with Claude Code + Paperclip

**Published:** Hashnode (srinibytes.hashnode.dev) + LinkedIn cross-post
**Target publish:** May 7, 2026
**Pillar:** AI/Engineering Lessons + Systems Thinking
**Word count:** ~1,400 words

---

<!-- LINKEDIN PULL-QUOTES (3 identified below for amplification post) -->
<!-- PQ1: "A single LLM is a smart generalist. An agent network is a team — and teams need governance." -->
<!-- PQ2: "The architecture took me two weeks. Getting the checkout protocol right took another three days. The governance layer — who can do what, when, and with what authority — took the longest of all." -->
<!-- PQ3: "I expected the models to be the hard part. Turns out, the hard part is convincing 33 agents to wait their turn." -->

---

I started this project convinced that the hard part would be the AI.

Pick the right model. Tune the prompts. Get good outputs. Then ship.

That's not how it went. Six weeks in, with 33 agents running across 10 repositories on a single VPS, I can tell you clearly: the models were the easy part. The governance layer was the hard part. And I didn't even know "governance layer" was a concept I needed until I was three production incidents deep.

This is the field notes version of how I built a personal AI agent network — not a tutorial, not a marketing post, but an honest account of what it took, what broke, and what I'd do differently.

## Why One LLM Wasn't Enough

The first version of my AI setup was simple: Claude in a terminal. I'd paste context, get outputs, implement things myself or paste the output back. Classic single-model workflow.

The limitation showed up fast. Every session, I'd re-explain the context. Every task, I'd mentally track which things were done and which weren't. The model was doing half my thinking for me, but I was still the only persistent memory in the system.

The insight that changed everything: **specialisation improves quality, and specialisation requires multiple agents.**

A single generalist LLM context-switches constantly. It's writing code one minute, thinking about design the next, then evaluating a PR it just wrote. The quality suffers because the context window carries all that noise.

An agent network solves this differently. My Frontend Engineer only thinks about frontend. My Content Writer (yes, that's the agent writing this post's brief) only thinks about content. Each agent loads a lean, focused context — their role, their tools, their current task. Nothing else.

The quality jump was immediate and measurable. PRs from the specialised agents were cleaner than anything the generalist approach produced. The code reviews caught real issues. The content had consistent voice.

One LLM is a smart generalist. An agent network is a team — and teams need governance.

## The Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     CEO Agent (Opus)                     │
│         Strategy, budget, cross-team delegation          │
└───────────────────┬─────────────────┬───────────────────┘
                    │                 │
        ┌───────────▼──┐     ┌────────▼──────────┐
        │ Engineering  │     │   Brand & Content  │
        │   Lead       │     │      Lead          │
        └──┬───────────┘     └────────┬───────────┘
           │                          │
    ┌──────┴──────┐           ┌───────┴────────┐
    │  Frontend   │           │  Content Writer │
    │  Backend    │           │  Diagram Gen    │
    │  DevOps     │           │  Social Copy    │
    │  QA Lead    │           └────────────────┘
    └─────────────┘
```

The architecture is a company org chart, not a workflow pipeline. That distinction matters.

A pipeline is sequential — step A feeds step B feeds step C. A company is event-driven — people (agents) pull work from a shared queue, execute it asynchronously, and leave outputs for others to pick up.

The technology making this work is [Paperclip AI](https://paper.srinivaskotha.uk) — a local orchestration layer that gives each agent:

- A **heartbeat schedule** (each agent wakes on a cron interval)
- An **issue inbox** (their assigned tasks, prioritised and filterable)
- A **checkout protocol** (a distributed lock so two agents can't work the same task simultaneously)
- A **comment thread** (asynchronous communication between agents and humans)
- A **DoD gate** (definition-of-done enforcement before any task closes)

The heartbeat protocol is the core primitive. Every agent wakes up, checks their inbox, picks the highest-priority assigned task, checks it out (acquires a distributed lock), does the work, and marks it done or blocked. Then it sleeps until the next heartbeat.

No two agents own the same task at once. The checkout lock prevents it. If Agent A has a task checked out and Agent B tries to check it out too, Agent B gets a `409 Conflict` and moves on to its next assignment.

## What I Got Wrong (And How I Fixed It)

### Token discipline

Early on, every agent was Opus. It felt right — I wanted the best model for every task. The subscription bill after the first week was not right.

The fix: **model routing by task weight.**

- Opus: orchestration only (CEO, my main Claude Code session)
- Sonnet: writing and reasoning-heavy tasks (engineering, content, planning)
- Haiku: reading, scanning, searching, status checks

This sounds obvious in retrospect. It wasn't obvious when I was building it. I defaulted to "best model" thinking, which is the wrong frame. The right frame is "what task weight requires what capability?"

A haiku reading a file and summarising three lines doesn't need Opus. An Opus model spending 4,000 tokens to grep a log file is waste.

### Worktree isolation

Three agents, one repo, zero isolation — this combination produced the worst incident of the project. Two agents switched branches in the same working directory simultaneously. Work got corrupted. Both posted "done." Neither was actually done.

The fix: every write-heavy agent now runs in a `/tmp/worktree-$TASK` directory via `git worktree add`. Isolated filesystem, isolated branch state. Clean up after commit. No shared state.

This is the kind of thing that's obvious to any senior engineer who's worked on a large team. It's less obvious when you're building your first agent system and you're thinking about prompts, not filesystem semantics.

### The checkout deadlock

The most subtle bug: two agents both checking out subtasks of the same parent issue, then each waiting for the other's subtask to complete before they could mark their own done.

A classic deadlock. Solved by adding a dependency declaration in the issue description — if Subtask B depends on Subtask A, that's explicit, and the agent checks whether its dependencies are done before checking out.

The pattern: **always model dependencies explicitly. Never let agents discover ordering by failing.**

## The Part That Surprised Me Most

The architecture took me two weeks. Getting the checkout protocol right took another three days. The governance layer — who can do what, when, and with what authority — took the longest of all.

Governance sounds bureaucratic. It's not. It's the difference between a functional team and chaos.

Examples from my system:

- **Budget authority**: Each agent has a spending limit. An IC agent can't spawn sub-agents beyond a certain cost. A director can. The CEO has full authority.
- **Cross-team delegation**: A frontend agent can't assign a backend agent. It has to escalate to the Engineering Lead who routes it appropriately.
- **DoD gates**: No task closes as "done" without a definition-of-done check. The system reads the acceptance criteria, verifies them against the work product, and rejects incomplete closures.
- **Escalation chains**: If an IC agent is stuck, it flags the task as "blocked" with a comment explaining why. The director sees it in their next heartbeat. The director escalates to CEO if needed. No task just disappears into silence.

I expected the models to be the hard part. Turns out, the hard part is convincing 33 agents to wait their turn.

## What I'd Do Differently

**Start with governance design, not model selection.** Before I picked models or wrote a single AGENTS.md file, I should have spent a day mapping: what decisions does each role make? What are the authority boundaries? What does escalation look like? The models can be swapped. The governance structure shapes everything else.

**Design for agents to fail gracefully.** Half of the early incidents were agents encountering unexpected states and either crashing silently or posting a confident "done" when they hadn't finished. Now every agent checks its outputs before marking done. The DoD enforcement layer catches the rest.

**LCM scheduling matters.** If your agents fire every 8 hours and every 10 hours, think about when they'll both fire simultaneously. Schedule them so collisions are rare. The thundering herd is real even at small scale.

## What's Running Now

Thirty-three agents. Ten repositories. Five project companies. One VPS with 7.8 GB of RAM.

The StreamVault redesign (26 active issues) is progressing without me writing a single line of code. The CEO Dashboard shipped five sprints ahead of schedule. The Content Writer (meta note: that's the agent working the brief for this post) maintains a content calendar and drafts on schedule.

I still review PRs. I still make architecture decisions. I still handle things that need judgment calls only I can make.

But the boring, mechanical parts of software development — implementation, testing, deployment verification, content drafting — those are running. Autonomously. On a schedule.

The surprising thing isn't that it works. The surprising thing is that it works better than the single-model approach ever did.

---

## Keep Building

If you found this useful, I write about AI-forward engineering, personal systems, and building in public at [Srinibytes on Hashnode](https://hashnode.com/@srinibytes). The next post covers the Ambient Depth design system — why my developer portfolio deliberately doesn't look like a developer portfolio.

Follow along: [@srinibytes](https://hashnode.com/@srinibytes) | [LinkedIn](https://www.linkedin.com/in/srinik7/)

---

## LinkedIn Amplification Post (required)

**Hook:** I expected the models to be the hard part of building a personal AI agent network. 33 agents, 10 repos, 1 VPS. The models were the easy part.

**Lesson:** The governance layer — who can do what, when, with what authority — took longer to design than the architecture. Checkout protocols. Budget limits. DoD gates. Escalation chains. A single LLM is a smart generalist. An agent network is a team. Teams need governance, not just intelligence.

**Invitation:** Full field notes (what broke, what I'd do differently, what's running now) on Srinibytes ↓

#AIEngineering #AgentOrchestration #BuildInPublic
