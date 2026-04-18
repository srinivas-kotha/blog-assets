# Why I Banned Wait Nodes from My n8n Workflows (and What I Use Instead)

I built the feature in 20 minutes. I spent the next three weeks debugging it.

The culprit was not a missing edge case or a race condition. It was an architectural choice that looked perfectly clever in the n8n UI but had three documented failure modes I had not read about. This is the story of how a 20-minute shortcut created 13 bug-fix PRs, three separate debugging sessions, and a rule that is now permanently baked into my engineering practice.

## The Setup

Kokilla is an AI music video pipeline I've been building — it takes a concept brief, generates storyboards, queues GPU jobs for video rendering, and produces a final cut. One step in the pipeline requires a human in the loop: a director needs to review and approve the concept before the expensive GPU compute starts.

In n8n, the obvious solution is a **Wait node**. It pauses the execution, sends a webhook URL to the approver, and resumes when that URL is hit. I'd seen it in tutorials. It matched the diagram in my head exactly. I had it working in 20 minutes.

What I should have done instead would have taken 2–3 hours. I chose fast. That was the mistake.

## The Three Gotchas

The n8n Wait node works by freezing the execution state and waiting for a resume signal. That sounds clean — but the implementation details matter a lot.

### 1. Stale code after deploy

When a Wait node pauses an execution, n8n serializes the current state to disk and waits. When the webhook fires, n8n _resumes from that serialized snapshot_ — using the workflow code that was active **when the wait started**, not the code that exists now.

This means if you deploy a new version of your workflow while executions are waiting, those executions will resume with the **old code**. You fixed a bug. The waiting executions don't know that. They finish the old, broken path and you have no idea why your "fixed" pipeline is still producing bad output.

I spent the first debugging session chasing this. Every time I deployed a fix, the in-flight executions ignored it.

### 2. Context loss on process restart

If the n8n process restarts — container redeploy, VPS maintenance, OOM kill — any execution that was in a Wait state is silently lost. No error. No webhook. No retry. The approval request just vanishes.

My VPS has a weekly maintenance window. The first time it hit during a Kokilla run, three approvals in flight disappeared and nobody on the pipeline noticed until a user asked why their video hadn't moved forward in two days.

### 3. The 24-hour timeout

n8n Wait nodes have a maximum wait duration. The default is 24 hours. Reviewers take weekends off. Productions span days. When an execution silently expires, you don't get an error — you get a dead run that looks like it's still in progress.

The third debugging session was me figuring out why approvals that I could _see_ in the database were not resuming the pipeline. They were there. The Wait was gone. Nothing connected them anymore.

## The Debugging Tax

Three separate debugging sessions across three different weeks. Thirteen bug-fix PRs to patch around these failure modes. I invented a "Reload pattern" — a workaround that refreshed execution state on resume — which made the code significantly harder to read. The redo path and the fresh-start path diverged into two separate code branches that both needed to be maintained.

The 20 minutes I saved upfront cost approximately 6+ hours of debugging time, plus the cognitive overhead of explaining to myself why the same system worked differently on different days.

## The Correct Pattern

The event-driven approach with a database state machine looks more complex on a diagram. It is actually simpler to reason about.

Here is how it works:

1. **Each approval step is a separate, short-lived, stateless workflow.** No long-running executions. No waiting.
2. **The database holds the state.** A `status` column tracks where the pipeline is: `pending_review → approved → processing → done`. The database is the execution state. n8n is just the trigger layer.
3. **Human action fires a webhook.** The approver clicks a link. The webhook fires a new n8n workflow.
4. **The new workflow reads fresh DB state.** It doesn't resume anything. It starts fresh, reads the current status from the database, and decides what to do next.
5. **No execution snapshot. No resuming. No stale code.**

The contrast in concrete terms:

| Concern              | Wait Node              | Event-Driven                    |
| -------------------- | ---------------------- | ------------------------------- |
| Code after deploy    | Runs old code          | Always runs current code        |
| Process restart      | Execution lost         | DB state survives restart       |
| 24h timeout          | Silent failure         | No timeout — DB record persists |
| Redo vs. fresh path  | Two code branches      | One path, `feedback?` parameter |
| Debugging on failure | Which execution? When? | Read the DB row. Done.          |

The "redo" use case is the clearest example of where this shines. With Wait nodes, a redo required a different code path — you couldn't easily restart at step 3 without special logic. With the DB state machine, redo is just setting the status column back to `pending_review` and firing the same webhook. One code path handles both.

## The Rule It Created

This experience is now baked into `architectural-integrity.md` in my engineering rules:

> Wait nodes for human-in-the-loop flows are an **anti-pattern**. They lose execution context, run stale code after deploy, and have a 24-hour timeout risk. Use the event-driven pattern: DB state machine + separate stateless workflows per step.

I also added a decision framework with five dimensions to score patterns before choosing them: statelessness, deployability, testability, redo simplicity, and failure isolation. The Wait node approach scores poorly on four of the five. The event-driven approach scores well on all five.

The information to make the correct call was available at planning time. The n8n documentation mentions the timeout. Community posts describe the stale-execution problem. I just didn't read them before choosing the fast path.

## What I Do Now

Before I touch any pipeline or multi-step workflow, I spend 5 minutes reading the skill docs and gotchas for that system. Not because I've gotten slower — because I ship fewer surprises.

The 20-minute shortcut is still available. I just know what it actually costs.

If you're building anything with n8n that involves human review, approval flows, or long-running state, the architecture question to answer first is: _who holds the state?_ If the answer is "n8n's execution engine," you will eventually have a bad week. If the answer is "the database," you've built something you can debug, redeploy, and restart without losing work.

That's the whole lesson. Everything else is implementation detail.

---

_Kokilla is a side project I'm building with AI agents. If you want to follow the architecture decisions (including the ones that blow up), I write about them here and on [LinkedIn](https://linkedin.com/in/srinivaskotha)._
