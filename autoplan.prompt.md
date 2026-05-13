---
mode: agent
description: One-command review pipeline. Runs /plan-design-review then /plan-devex-review on a plan file, auto-deciding intermediate questions using 6 explicit principles. Surfaces only the genuine taste decisions to the user at a final approval gate. Use when you have a plan and want it fully reviewed without sitting through 20+ intermediate questions.
---

# /autoplan — Auto-Review Pipeline

One command. Plan in, fully reviewed plan out.

`/autoplan` runs the full `/plan-design-review` and `/plan-devex-review` skills from
this folder at full depth — same rigor, same passes, same methodology as running each
manually. The only difference: intermediate questions are **auto-decided** using the
6 principles below. Genuine taste decisions (where reasonable engineers could disagree)
are surfaced at a final approval gate.

You still produce every artifact each phase requires — sequence diagrams, resilience
config tables, contract specifications, persona cards. You don't compress. You decide
for the user the things you can defensibly decide, and surface the rest.

---

## The 6 Decision Principles

These auto-answer every intermediate question:

1. **Choose completeness.** Pick the option that covers more failure modes, more
   personas, more edge cases. Plans don't suffer from too much rigor.
2. **Fix the blast radius.** If a finding implies fixing other places that share the
   shape, include them — but only if they're in the **blast radius** (files this plan
   touches + their direct importers). Don't refactor the world. Auto-approve in-radius
   expansions of < 1 day of work (< 5 files, no new infra).
3. **Pragmatic.** If two options solve the same problem, pick the cleaner one. 5
   seconds choosing, not 5 minutes.
4. **DRY.** Reusing an existing pattern in this codebase (from `CONTEXT.md` §7 or the
   review of similar features) beats inventing a new one.
5. **Explicit over clever.** A 10-line obvious solution beats a 200-line abstraction.
   Pick what a new contributor reads in 30 seconds.
6. **Bias toward action.** Surfacing a concern as a deferred TODO beats blocking the
   plan with it.

**Tiebreakers by phase:**
- **Design pass:** principles 1 + 2 dominate (completeness + blast-radius coverage).
- **DevEx pass:** principles 5 + 3 dominate (explicit + pragmatic).

---

## Decision Classification

Every auto-decision is classified:

- **Mechanical** — one clearly right answer. Auto-decide silently.
  Examples: add a missing timeout to a vendor client → yes; reduce scope on a complete
  plan → no; cite an existing pattern rather than invent a new one → yes.
- **Taste** — reasonable engineers could disagree. Auto-decide with recommendation,
  but **surface at the final gate**. Three sources:
  1. **Close approaches** — top two options are both viable with different tradeoffs.
  2. **Borderline blast radius** — 3–5 files, or ambiguous downstream reach.
  3. **Defaults vs. existing convention** — when this codebase's convention conflicts
     with a stronger general best practice.
- **User Challenge** — the reviews surface that the user's stated direction may be
  wrong (merge two features that should stay separate, add a feature that contradicts
  the stated scope, remove something the user said was core). **Never auto-decided.**

User Challenges go to the final gate with richer context:
- What the user said
- What the review recommends
- Why
- What context the review might be missing
- If we're wrong about the user being wrong, the cost is…

The user's original direction is the default — the review must make the case for
change, not the other way around.

**Exception:** if the review surfaces a security or correctness blocker (not a
preference), label it explicitly: "This is a correctness/security risk, not a
preference." User still decides.

---

## Sequential execution — MANDATORY

Phases execute in strict order. Each fully completes before the next starts.

1. `/plan-design-review` — architectural soundness
2. `/plan-devex-review` — developer/operator experience

Why this order: design issues often invalidate DevEx decisions (the right contract
depends on the right system shape). Doing DevEx first wastes work when design changes
the API shape under it.

Between phases, emit a phase-transition summary and verify the prior phase's outputs
exist before starting the next.

---

## What "auto-decide" means

Auto-decide replaces the **user's judgment** with the 6 principles. It does **not**
replace the **analysis**. Every pass in each phase still runs at full depth:

- READ the actual plan, code, and `CONTEXT.md` each pass references.
- PRODUCE every artifact the pass requires (diagrams, tables, persona cards).
- IDENTIFY every issue the pass is designed to catch.
- DECIDE each issue using the 6 principles, instead of asking the user.
- LOG each decision in the audit trail (see below).
- WRITE all required edits to the plan.

You may NOT:
- Compress a pass into a one-liner.
- Write "no issues found" without showing what you examined (1–2 sentences min).
- Skip a pass because "it doesn't apply" without stating what you checked and why
  nothing surfaced.

"No issues found" is a valid outcome for a pass — but only after doing the analysis.

---

## Phase 0 — Intake + restore point

1. Read `.copilot/CONTEXT.md`. Absent → "Run `/load-context` first" and stop.
2. Locate the plan file (same logic as `/plan-design-review` Step 0).
3. **Save a restore point.** Before any edit, copy the plan file:
   ```bash
   mkdir -p .copilot/autoplan-restore
   DATETIME=$(date +%Y%m%d-%H%M%S)
   cp <plan-file> ".copilot/autoplan-restore/${DATETIME}-$(basename <plan-file>)"
   echo "Restore point: .copilot/autoplan-restore/${DATETIME}-$(basename <plan-file>)"
   ```
4. Prepend a one-line HTML comment to the plan file:
   `<!-- /autoplan restore point: .copilot/autoplan-restore/{datetime}-{filename} -->`
5. Detect scope from the plan:
   - **Backend / architectural scope?** Yes if the plan mentions controllers, services,
     vendors, batch, scheduling, persistence — almost always yes for this codebase.
     Run Phase 1.
   - **DevEx scope?** Yes if the plan mentions a new endpoint, event topic, webhook,
     SDK, CLI, runbook addition, or is consumed by another team/service. Run Phase 2.
   - Otherwise → tell the user "This plan has neither architectural nor DevEx scope I
     can review with `/autoplan`. Use `/review` on the diff instead." Stop.

Print the detected scope and the planned phase sequence.

---

## Phase 1 — Architectural review (auto-decided)

Run the full `/plan-design-review` skill (read its prompt file end-to-end and follow it).
At every AskUserQuestion-style decision point:

1. Decide using the 6 principles.
2. Classify as Mechanical, Taste, or User Challenge.
3. Apply the decision (edit the plan, add the section, fix the gap).
4. Log to the audit trail:
   ```
   PHASE 1 DECISION {N}
     Pass:       {pass number + name}
     Issue:      {one line}
     Options:    A) ... B) ... C) ...
     Auto-chose: {A|B|C}
     Why:        principle #{n} — {brief reason}
     Class:      Mechanical | Taste | User Challenge
   ```

**Exception — always pause for User Challenge.** When the principles say "your plan is
asking for the wrong thing," do NOT auto-decide. Save it for the final gate.

**Required outputs of Phase 1 (must exist before moving to Phase 2):**
- All 8 passes' ratings recorded (initial → final).
- Plan file edited to address every Mechanical decision.
- "Decisions made" table appended/updated in the plan.
- "Unresolved" section appended/updated if any deferrals.
- TODOs proposed (auto-add for in-blast-radius mechanical ones; surface bigger ones
  at the final gate).
- Completion Summary printed.

Print phase transition:
```
─── Phase 1 complete ───
Initial design score: X/10  →  Final: Y/10
Mechanical decisions: N
Taste decisions:      M    (surfacing at final gate)
User Challenges:      K    (surfacing at final gate)
TODOs auto-added:     P
```

---

## Phase 2 — DevEx review (auto-decided)

Run the full `/plan-devex-review` skill end-to-end.

**Persona auto-selection.** In `/plan-devex-review` Step 0a, the persona is normally an
ASK. Auto-decide using:

- If plan mentions partner / external / public API → C (partner integrator).
- Else if plan mentions mobile or frontend consumers → B (app team).
- Else if plan adds runbook / on-call / alerts → D (operator).
- Otherwise → A (internal service team).

If multiple apply, pick the most demanding one (C > B > D > A). Log the choice and
why; surface at the final gate as a Taste decision so the user can override.

For Step 0b empathy walkthrough — generate it, but don't ask if it "rings true";
proceed and surface the walkthrough in the final gate so the user can object once.

For all subsequent passes, decide via the 6 principles. Same logging.

**Required outputs:**
- Persona card recorded.
- All 7 passes' ratings (skip Pass 7 if not operator persona; or include for operator).
- Plan edits applied.
- TODOs proposed.
- Completion Summary printed.

Print phase transition.

---

## Phase 3 — Final approval gate

Aggregate everything that wasn't Mechanical. Present in this exact order:

```
─── /autoplan REVIEW COMPLETE ───

Plan:           {path}
Restore point:  .copilot/autoplan-restore/{filename}

Phase 1 (design):  X/10 → Y/10
Phase 2 (devex):   X/10 → Y/10

Auto-applied:      {N} mechanical decisions
Surfacing now:     {M} taste decisions + {K} user challenges

═══════════════════════════════════════════
USER CHALLENGES — your stated direction may need revision
(default = keep your direction; review must convince you otherwise)
═══════════════════════════════════════════

UC-1. {one-line description}
  You said:       {original direction from plan}
  Review says:    {the change the review recommends}
  Why:            {reasoning from the relevant pass}
  Cost if we're wrong about you: {what happens if your direction was correct}
  → A) Keep my direction (default)
  → B) Apply the review's recommendation
  → C) Discuss

UC-2. ...

═══════════════════════════════════════════
TASTE DECISIONS — auto-picked but worth confirming
═══════════════════════════════════════════

T-1. {one-line description}
  Auto-chose:  {choice}
  Why:         principle #{n}
  Alternative: {the other defensible option}
  → A) Confirm (default)
  → B) Switch to the alternative
  → C) Revisit later (mark as unresolved)

T-2. ...

═══════════════════════════════════════════
TODO PROPOSALS — review surfaced these as defer-or-do-now
═══════════════════════════════════════════

(per TODO: What, Why, Pros, Cons, Context, Depends on)
TD-1. {summary}  → A) Add to TODOS.md (default)  B) Build now  C) Skip
TD-2. ...
```

Ask the user to resolve in one pass — accept their answers, apply each, log each
to the audit trail, then write the final plan.

---

## Phase 4 — Persist + suggest next step

Append the full audit trail to `.copilot/reviews/autoplan-{ISO date}.md`:

```markdown
# /autoplan audit — {plan path} — {date}

Restore: .copilot/autoplan-restore/{filename}

## Phase 1: /plan-design-review
- Initial: X/10  →  Final: Y/10
{full decision list from Phase 1 logs}

## Phase 2: /plan-devex-review
- Persona: {chosen}
- Initial: X/10  →  Final: Y/10
{full decision list from Phase 2 logs}

## Final gate
- User challenges resolved: {n}, with outcomes
- Taste decisions confirmed: {n}, with outcomes
- TODOs added: {n}
```

Print:
```
Plan is review-complete.

Next step: implement the plan, then run /review on the diff before /ship.
If you want a security-specific pass after implementation, run /cso.
Restore point is at .copilot/autoplan-restore/... in case you want to undo.
```

If recurring patterns surfaced across the two phases, append a learning to
`.copilot/learnings.jsonl`.

---

## Rules

- **Read each skill's prompt file end-to-end before running it.** Don't shortcut.
- **Every pass produces its required artifact.** No compression.
- **Audit trail is non-negotiable.** Every auto-decision logged with the principle that
  drove it.
- **Never auto-decide a User Challenge.** Even if you're 100% sure, surface it.
- **Never auto-decide a security/correctness blocker.** Frame as urgent; user still
  decides.
- **Restore point first.** Before any edit, save the plan as it was. Print the path.
- **One final gate.** Don't surface decisions piecemeal mid-flow; that defeats the
  point of `/autoplan`. Save everything for Phase 3.
- **If the user halts mid-flow** (says "stop", "wait", "let's revisit"), pause cleanly,
  print everything decided so far + the audit trail, and respect their pause.
