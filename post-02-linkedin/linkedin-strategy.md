# LinkedIn Strategy — Post #2 Amplification

## LinkedIn Post Draft

> Ready to paste. 1,180 characters.

---

17 AI agents. One $12/month VPS. Three production incidents in 72 hours.

I deployed a full AI agent company — CEO, directors, 15 specialists — each running on a heartbeat schedule, working through a task queue without me touching anything.

What I expected: side projects moving on their own while I reviewed PRs.

What I got: an incident report.

→ **Thundering herd.** Two agents on 8-hour intervals fired in sync, every time. Session quota hit 40% in 5 minutes. Fix: stagger so the LCM is 40 hours. LCM scheduling is a real systems problem, not a theoretical one.

→ **Three writers, one repo.** Parallel git ops in the same working directory corrupted each other's branches. Both posted "done." Neither change was committed. Fix: every write-agent now runs in an isolated git worktree.

→ **Budget caps were theater.** Auto-pause fired correctly — but only for API consumers. Subscription users ran uncapped. I had a circuit breaker with no wire. Fix: 15 tool calls per heartbeat, enforced by the agent's own instructions.

One week in: running cleanly. My job shifted from execution to judgment.

That's the actual value.

Full incident report + code: [link to Hashnode post]

If you build with AI agents, follow me — I post what actually happens, not the highlight reel.

---

## Engagement Reply Templates

### Template 1: "Great post!" / Generic praise

> Use when someone drops a short positive comment ("Love this!", "Great share!", "Nice work!")

Thanks! The part that surprised me most was the budget caps story — I was genuinely confident that was solved. What's the trickiest constraint you've hit when automating anything? Always curious what breaks first for other people.

---

### Template 2: Skeptic / "AI can't do X"

> Use when someone pushes back ("AI agents aren't reliable enough", "This is still a toy", "LLMs hallucinate too much for production work")

Fair pushback — and worth being specific about. These agents aren't making architecture decisions or writing novel algorithms. They're doing bounded, reviewable tasks: drafting content, creating PRs, posting status updates. The value is in shifting _my_ time from execution to review. The failure modes are real (three incidents in the post), but they're engineering problems, not fundamental limits. What specific task were you thinking of when you said that?

---

### Template 3: "How did you build this?"

> Use when someone asks about the stack, tools, or how to get started

The orchestration layer is Paperclip AI (local, open-source) — it handles heartbeat scheduling, shared inboxes, and task management for Claude Code agents. Each agent is essentially a Claude Code instance with role instructions and a cron-style wake schedule. I wrote about the full setup in [Post #1 — link]. The short version: it runs on a $12 VPS and the hardest part is the git isolation, not the AI.

---

### Template 4: Fellow builder sharing their experience

> Use when someone shares a similar story or pain point ("I tried something like this and hit X", "We ran into the same thing at work")

That's exactly the pattern I keep seeing — the AI part is usually fine, the _systems_ part (scheduling, isolation, state management) is where it breaks. How did you end up solving X? I'm curious whether the worktree approach works for your setup or if you needed something different. Always learning what generalizes and what's specific to my stack.

---

### Template 5: Recruiter / DM-bait

> Use when someone says "Let's connect!", drops a company pitch, or asks if you're open to opportunities

Thanks for reaching out. I'm deep in building Srinibytes right now and not looking for roles. If you're building with AI agents and want to trade notes, I'm genuinely interested in that conversation — drop a technical question here or in my DMs and let's go from there.

---

## Hashtag Set

```
#AIAgents #BuildingWithAI #SoftwareEngineering #IndieHacker #AIAutomation #MultiAgent #Claude
```

**Rationale:**

- `#AIAgents` — primary topic, high engagement in the AI dev community
- `#BuildingWithAI` — practitioner-focused, less polluted than #AI
- `#SoftwareEngineering` — reaches the engineering audience, not just AI enthusiasts
- `#IndieHacker` — fits the $12 VPS / solo founder angle, active community
- `#AIAutomation` — relevant to the automation framing
- `#MultiAgent` — specific to the architecture discussed, reaches researchers + practitioners
- `#Claude` — Anthropic ecosystem community, relevant given the stack

**Usage note:** LinkedIn recommends 3–5 hashtags for optimal reach. Lead with `#AIAgents #BuildingWithAI #SoftwareEngineering` if trimming — those three cover the broadest relevant audience.
