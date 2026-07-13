# Agent Line Map: Cortex PM Chief-of-Staff Agent

> Module 1 · The Agent Line

## The workflow, decision by decision

List every discrete decision or action in your agent's workflow, then score each one and place it **above** the line (a human owns it) or **below** (the agent owns it). Borderline calls get an HITL checkpoint.

| Decision / action | Reversibility (H/M/L) | Blast radius (H/M/L) | Measurability (H/M/L) | Above / Below | HITL? |
|---|---|---|---|---|---|
| Pull project state + recent GitHub/Jira activity (`get_project`, `get_activity`) | H | L | H | Below | · |
| Retrieve roadmap + team norms as grounding context (`get_roadmap`, `get_norms`) | H | L | H | Below | · |
| Search past updates/decisions for tone and precedent (`search_past_updates`) | H | L | H | Below | · |
| Draft the weekly leadership status update from pulled evidence | H | M | M | Below | spot-check |
| Assign the red/yellow/green call on a project's draft update | H | M | M | Below | spot-check |
| Propose next sprint's stories from the PRD, within the 10-item cap (`propose_stories`) | M | M | M | Below | spot-check |
| Recognize a stories batch that exceeds the queue cap and escalate instead of splitting it | H | L | H | Below | · |
| Halt the run when the cost cap, iteration cap, or revision cap trips, and escalate | H | L | H | Below | · |
| Detect an instruction embedded in a task brief/pasted notes trying to override norms, and refuse + escalate | H | L | H | Below | required |
| Post the update to a channel / send it to leadership | L | H | M | Above | required |
| Commit or confirm a ship / GA date (e.g. under pressure with no confirmed date) | L | H | L | Above | required |
| Mark a launch gate green, or create/close/merge a ticket or PR | L | H | M | Above | required |
| Surface a CONFIDENTIAL/embargoed roadmap item (e.g. Orbit) in an external or company-wide draft | L | H | H | Above | required |
| Report go/no-go on a project with an open Sev-1 or `launch_hold` flag | M | H | M | Above | required |

## Agent anatomy (sketch)

- **Model:** `gpt-4o-mini` as the default drafting model (cheap and fast enough for a weekly-cadence PM task, and it keeps the $0.50 cost cap meaningful). Today the critic call reuses the *same* model rather than escalating to a frontier model — that's a gap: the honest escalation trigger would be a second critic rejection or a low-confidence color call (Sev-1/launch_hold edge cases), where I'd route to a stronger model before ever routing to a human.
- **Tools:** read-only lookups — `get_project`, `get_activity`, `search_past_updates`, `get_roadmap`, `get_norms` — plus one write-shaped-as-read tool, `propose_stories`, which only queues (creates nothing, capped at 10, rejects and escalates over cap). Deliberately absent: `post_update`, `create_issue`, `merge_pr`, `commit_ship_date`, `mark_gate`, `close_bug`. The agent line is enforced in `tools.py`, not in a prompt.
- **Memory:** roadmap, team norms, past updates, and the decision log are read-only reference memory Cortex queries fresh each run — none of it is written by Cortex, and nothing Cortex produces persists run-to-run today. That's a real gap for M4: a chief-of-staff agent should accumulate its own precedent (e.g. which color calls got overridden by a human) instead of re-deriving norms from scratch every time.
- **Loop:** placeholder, defined in `02-loop-design/loop-spec.md`. In short: tool-call loop → draft → independent critic pass → pass (stop at HITL checkpoint) or revise (cap 2) or escalate, bounded by 8 iterations total.
- **Bounds:** placeholder, defined in `05-bounds-evals/bounds-and-evals.md`. In short: `CORTEX_MAX_ITERATIONS=8`, `CORTEX_MAX_REVISIONS=2`, `CORTEX_COST_CAP_USD=0.50`, `CORTEX_MAX_QUEUE_ITEMS=10`, all enforced outside the model.
- **Evals:** placeholder, defined in `05-bounds-evals/bounds-and-evals.md`.

## The golden rule, applied

- **Post the update / send to leadership** — reversibility fails: once it's in a channel or an inbox, people act on it, and there's no unsend.
- **Commit or confirm a ship/GA date** — measurability fails: with no confirmed date to check against, there's nothing to verify the claim against before someone forwards it as fact.
- **Mark a launch gate green / merge or close a ticket** — reversibility and blast radius both fail: a wrong green light can ship to customers before anyone notices.
- **Surface a CONFIDENTIAL/embargoed roadmap item externally** — blast radius fails hardest: an embargo leak can't be un-leaked even if a human catches it on the second read, so Cortex can't be trusted to draft the sentence at all, not just trusted to let a human strike it.
- **Report go/no-go on a Sev-1/launch_hold project** — blast radius fails: a wrong go signal reaches customers or leadership before a human can intervene.

## Hardest call

The red/yellow/green status call was the hardest to place. It's a draft, nothing gets sent without a human, so on paper it reads as safely reversible. But it's also the single number leadership skims first, and a distracted reviewer rubber-stamps color far more readily than they re-derive it from the underlying PRs. I kept it **below** the line because it's evidence-traceable (the norms give a checkable rule: Sev-1 or `launch_hold` present → never green) — but I made the spot-check **mandatory** rather than optional, and split the Sev-1/`launch_hold` case out into its own **above-the-line** row, because that's the one version of "what color is this" that isn't really a status call, it's a go/no-go call in disguise.
