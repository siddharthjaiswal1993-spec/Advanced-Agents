# Agent Line Map: Cortex PM Chief-of-Staff Agent

> Module 1 · The Agent Line

## Step 1 — Every discrete decision, atomic

1. Pull a project's state and recent activity from internal systems (`get_project`, `get_activity`).
2. Decide which past updates / context are relevant to the task (`search_past_updates`).
3. Draft the leadership status update.
4. Decide the tone / commitment level of the update (e.g. promising a date).
5. Flag a project as at-risk or an issue as a likely escalation (red/yellow/green, Sev-1/`launch_hold`).
6. Choose which risk call to escalate to a human (including refusing an injected instruction).
7. Propose a story batch within the cap, with justification (`propose_stories`).
8. Post an update, or approve a company-wide one.

Each item is a single action at a single risk level — none bundle a decide-step with a do-step (e.g. drafting is separate from sending; flagging risk is separate from deciding whether to escalate it).

## Step 2 & 3 — Scored, and placed above/below the line

Default is **above the line**. An item only drops below when it earns it on all three axes; anything mixed becomes **HITL** rather than forced into a bucket.

| # | Decision / action | Revers. | Blast | Measur. | Verdict | One-line justification |
|---|---|---|---|---|---|---|
| 1 | Pull project state / recent activity (`get_project`, `get_activity`) | High | Low | High | **Below** | Read-only and checkable against the source system, so Cortex owns it. |
| 2 | Decide which past updates/context are relevant (`search_past_updates`) | High | Low | Med | **Below** | Cheap to re-run and low-risk; a bad pick only weakens the draft, so the deciding factor is reversibility, not the softer measurability score. |
| 3 | Draft the leadership status update | High | Low | High | **Below** | Nothing leaves the building until a human sends it, and it's fully checkable against pulled evidence, so Cortex owns the draft. |
| 4 | Decide the tone / commitment level of the update (e.g. promising a date) | Low | Med | Low | **Above** | Once a "confirmed" date or gate-passing tone is stated and forwarded to leadership, retracting it costs real trust even though it was "just a draft" — reversibility is the deciding factor, which is why Cortex's norms make this an unconditional escalation, not a discretionary approval. |
| 5 | Flag a project as at-risk / an issue as a likely escalation (red/yellow/green, Sev-1/`launch_hold`) | High | Med | Med | **HITL** | It's evidence-checkable against a hard rule (Sev-1 or `launch_hold` present → never green), but a rushed reviewer rubber-stamps a color far more readily than they re-derive it, so blast radius earns it a mandatory spot-check rather than a silent below. |
| 6 | Choose which risk call to escalate to a human (including refusing a prompt-injection attempt) | Med | Med | Med | **HITL** | Under-escalating risks real damage before anyone notices, over-escalating only costs a human's time, and there's no clean measure of "should this have escalated" in the moment — all three axes are middling, so a human confirms the call. |
| 7 | Propose a story batch within the cap, with justification (`propose_stories`) | Med | Med | Med | **Below**, mandatory spot-check | Queued and capped by infrastructure, not created — a wrong batch is cheap to reject before it becomes real sprint work, so Cortex owns the proposal but every batch gets checked. |
| 8 | Post an update, or approve a company-wide one | Low | High | Low | **Above** | Irreversible reach and no way to unsend or un-leak, a human owns this, always. |

## Agent anatomy (sketch)

- **Model:** `claude-haiku-4-5` as the default drafting model (cheap and fast enough for a weekly-cadence PM task, and it keeps the $0.50 cost cap meaningful). Today the critic call reuses the *same* model rather than escalating to a frontier model — that's a gap: the honest escalation trigger would be a second critic rejection or a low-confidence color call (row 5's Sev-1/`launch_hold` edge cases), where I'd route to a stronger model (e.g. `claude-opus-4-8`) before ever routing to a human.
- **Tools:** read-only lookups — `get_project`, `get_activity`, `search_past_updates`, `get_roadmap`, `get_norms` — plus one write-shaped-as-read tool, `propose_stories`, which only queues (creates nothing, capped at 10, rejects and escalates over cap). Deliberately absent: `post_update`, `create_issue`, `merge_pr`, `commit_ship_date`, `mark_gate`, `close_bug`. The agent line is enforced in `tools.py`, not in a prompt.
- **Memory:** roadmap, team norms, past updates, and the decision log are read-only reference memory Cortex queries fresh each run — none of it is written by Cortex, and nothing Cortex produces persists run-to-run today. That's a real gap for M4: a chief-of-staff agent should accumulate its own precedent (e.g. which color calls got overridden by a human) instead of re-deriving norms from scratch every time.
- **Loop:** placeholder, defined in `02-loop-design/loop-spec.md`. In short: tool-call loop → draft → independent critic pass → pass (stop at HITL checkpoint) or revise (cap 2) or escalate, bounded by 8 iterations total.
- **Bounds:** placeholder, defined in `05-bounds-evals/bounds-and-evals.md`. In short: `CORTEX_MAX_ITERATIONS=8`, `CORTEX_MAX_REVISIONS=2`, `CORTEX_COST_CAP_USD=0.50`, `CORTEX_MAX_QUEUE_ITEMS=10`, all enforced outside the model.
- **Evals:** placeholder, defined in `05-bounds-evals/bounds-and-evals.md`.

## Step 4 — Hardest above-vs-below call (for #cohort-channel)

**Row 4 — deciding the tone/commitment level of the update (e.g. promising a date).** I went back and forth between HITL ("Cortex drafts a proposed commitment, a human approves before it goes out") and Above ("Cortex never even proposes committing"). The case for HITL: nothing is irreversible yet, it's still a draft, and a human always reviews before send anyway. What settled it was **reversibility, not blast radius**: the failure mode isn't the draft itself, it's that a stated commitment reads as fact to whoever forwards it, and by the time it's wrong, the cost has already been paid in trust, not in the draft. That's why Cortex's own norms don't leave this to case-by-case approval — they make it an unconditional escalation ("if an update would require an unconfirmed date, escalate the date question rather than committing one"). A HITL checkpoint implies a human sometimes says yes to a proposed commitment; the actual design says Cortex should never propose one to begin with.

Runner-up: row 5 (the red/yellow/green call) — see the note in that row's justification. It looks safely reversible (it's a draft) but blast radius earns it a mandatory spot-check rather than a silent below-the-line pass, because a wrong color is the one line a rushed reviewer is most likely to skim past.
