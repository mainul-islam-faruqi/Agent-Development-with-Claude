# Reflection Brief — Harness Engineering Capstone

**Name:**
**Date:**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):
- OS / Python:
- Approx. API spend:

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In the trace `Evidence\system 1\claim_04_neighbor_injury.jsonl`, the stop_reason sequence is: tool_use, tool_use, tool_use, tool_use, then end_turn. The control decision is in loop.py, inside the run(...) function. The loop does if response.stop_reason == "end_turn": return ..., if response.stop_reason == "tool_use": append tool results and continue, and otherwise raises UnexpectedStopReason. That means the loop continues only on a structured tool_use signal and stops only on end_turn, rather than guessing from raw assistant text.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One anti-pattern is string-membership checks against assistant text to control the loop, such as if "end_turn" in model_output:. The test forbids it because a random message could contain that text and falsely trigger the loop to end, which breaks the run. The proper pattern is to use stop_reason as the explicit control signal.
   

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → In `Evidence\system 1\claim_04_neighbor_injury.jsonl`, `classify_claim` and `assess_severity` receive the same claim facts but perform different jobs. The first returns the claim category (`liability`), while the second returns urgency (`high`), so their descriptions and schemas establish a clear boundary and reduce misrouting. A structured tool error identifies the exact invalid field or enum value, allowing the agent to repair and retry the call; a generic string would provide less precise guidance.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → In `Evidence\system 1\summary.md`, `claim_04_neighbor_injury` completed as `routed` in 4 turns, using 14,192 input tokens and 846 output tokens, with an estimated cost of `$0.0184`. The detailed trace contains three `tool_use` turns and one final `end_turn`, confirming the four-turn count. Its cost reflects the policy lookup, fact recording, classification, severity assessment, and adjuster routing performed during the run.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → In Evidence\system 2\budget.json, the baseline was 38,708 tokens, the assembled context was 16,893 tokens, which is a 56.36% reduction. The section that dominates the assembled context is active, at 15,789 tokens — about 93% of the total — so that is the main budget driver. I keep it verbatim because the active issue is the live, unresolved thread where every turn can matter to the next decision; the README explicitly says the active issue is preserved byte-exact at the bottom boundary to protect coherence and fidelity. The resolved issues (resolved_refund and resolved_subscription) are the compressible parts, because their facts are stable and can be summarized without losing the decisions that still matter.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → In `Evidence\system 2\budget.json`, the four sections are case_facts: 204, resolved_refund: 384, resolved_subscription: 534, and active: 15,789 tokens. The rule is to keep the active issue byte-exact because it dominates the assembled context and is the only section still changing, while the resolved issues are summarized because they are stable and far smaller. This matches the README’s design: preserve the unresolved thread verbatim, compress the completed history.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The regressed question is Q6: “What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?” In `Evidence\system 2\eval.jsonl`, the model answered with the exact token in_progress and passed, but in `Evidence\system 2\eval_control.jsonl` it failed by saying there was no status token and paraphrasing the issue instead. That proves the facts block matters: preserving the active issue’s exact status token is necessary for correctness. The control run shows that without the verbatim fact, the model drifts into a generic explanation and loses the required structured answer.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In `...\.claude\rules\react.md`, the rule frontmatter is path-scoped with a glob, not tied to one directory. This is better than a directory-level CLAUDE.md because the rule applies only to files matching the pattern, so the same convention can be reused across the repo without being duplicated in every folder. It is more precise, easier to maintain, and less likely to leak into unrelated files.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → → In `.../.claude/skills/deploy-check/SKILL.md`, the skill declares `context: fork` and `allowed-tools` limited to `Read`, `Grep`, `Glob`, and read-only `git`/`gh` commands. Running in a forked, read-only context keeps the noisy diff and status exploration out of the main session while returning only a final `pass | fail` summary. Without that boundary, the parent session gets polluted with intermediate output and the check can accidentally become mutating or misleading. The read-only allowlist makes the deploy gate a safe inspection step instead of a side-effectful operation.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → → In this config, the project-level scope is the repo-owned `.claude` setup, such as `.../.claude/skills/deploy-check/SKILL.md`, which is committed with the team’s config and applies to the repository. The user-level scope is the personal variant described in the same file: a parallel skill in `~/.claude/skills/deploy-check-strict/`, which is outside the repo and does not affect teammates. The validator makes this distinction by location: repo-root `.claude/...` is project scope, while `~/.claude/...` is user scope. That split lets the team share defaults while keeping one-off personal rules private.

### System 4 — Orchestration

11.  **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In the System 4 run `Evidence\system 4\Screenshot 2026-09-21 shift output.png`, `defects_since` returned **0 new defects**, shown by `run_shift done: shift=C new=0`, while `Evidence\system 4\warm.sqlite` contains **40 warm-tier rows**. The query uses the `idx_defects_ts` index on `defects(ts)` to filter by the `since` timestamp before orchestration builds the model context. Because the run used `since=2026-09-20T17:03:24Z` and the stored defects are from April 2026, none matched; the model therefore did not receive the full historical table.
    

12.  **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\recovery.py`, persisted state older than approximately **30 minutes** is treated as stale and causes a fresh start; newer state can be resumed. A fresh start with an injected summary is more reliable because it avoids carrying stale tool state, partial work, or outdated assumptions while preserving the important checkpoint information. Forked investigation is isolated separately in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\fork.py`: `fork_for_hypothesis` creates `data/forks/<hypothesis_id>/`, copies the base `hot_state.json` with `shutil.copyfile`, and leaves the base state untouched. The solution README confirms this behavior, and `shift_monitor\scratchpad.py` backs investigations with append-only JSON-lines scratchpads, so concurrent hypotheses write to separate fork state instead of polluting the main checkpoint.


13.  **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → After the System 4 run, `Evidence\system 4\hot_state.json` was **643 bytes**, below the approximately **5 KB** budget. Keeping this checkpoint bounded prevents restart context, token cost, and recovery risk from growing across indefinite shift runs. The small state also makes the resume-vs-fresh decision in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\recovery.py` predictable because recovery reads a compact checkpoint instead of an expanding history.
    

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify. 
    → Model: `Evidence\system 1\claim_04_neighbor_injury.jsonl` shows the model layer because the model itself chooses the next action by emitting tool calls such as `classify_claim`, `assess_severity`, and the final terminal action. The model is making the semantic decision about which tool to call and with what arguments.  
    → Harness: `Build a Claims Intake Agent with a stop_reason-Driven Loop\exercises\03-dynamic-decomposition\solution\claims_intake\loop.py` is the harness layer because it wraps the model call, reads `stop_reason`, dispatches tools, appends tool results, and enforces continue-vs-stop behavior. That is not the model deciding; it is deterministic code controlling the interaction boundary.  
    → Orchestration: `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\recovery.py` is the orchestration layer because it manages cross-run state and decides whether to resume or start fresh based on staleness. Together, the model selects actions, the harness executes the agent loop, and orchestration manages durable multi-shift state.

    

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the stale-state recovery rule in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\recovery.py`: persisted state older than about **30 minutes** is treated as stale and the system starts fresh instead of resuming. This is enforced by code and verified in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\tests\test_us03_crash_recovery.py`: `test_threshold_constant_is_30_minutes` asserts `STALE_RESUME_THRESHOLD_MINUTES == 30`, and `test_recovery_decide_truth_table` verifies that 29 and 30 minutes resume while 31 minutes starts fresh. A prompt-guided behavior is System 2’s compression of older resolved issues: the prompt guides how to summarize old context, while `Evidence\system 2\budget.json` shows the result, reducing **38,708 baseline tokens** to **16,893 assembled tokens**. Code is right for safety invariants like freshness thresholds and byte-exact preservation; prompts are right for judgment-heavy summarization where the model chooses compact wording.
    

16.  **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → In `Evidence\system 2\budget.json`, System 2 reduced the context from **38,708 baseline tokens** to **16,893 assembled tokens**, a **56.36% reduction**, while preserving the active issue verbatim. In the System 4 run output, `defects_since` returned **0 defects** from **40 warm-tier rows** in `Evidence\system 4\warm.sqlite`, because the run used `since=2026-09-20T17:03:24Z` and the stored defects were from April 2026. The principle is the same: keep high-signal context and exclude irrelevant history. The mechanism differs: System 2 manages intra-session text/token budget, while System 4 filters cross-session database history before the model sees it.


17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `Build a Claims Intake Agent with a stop_reason-Driven Loop\exercises\01-loop\solution\tests\test_antipatterns.py` guarantees that the loop must not decide continue-vs-stop by scanning raw assistant text; it must use the structured `stop_reason` signal. A single run can look successful by chance even if brittle string matching is present, because that run may not contain the misleading text that triggers the bug. The test matters because it prevents control-flow drift before shipping and protects traces like `Evidence\system 1\claim_04_neighbor_injury.jsonl`, where the loop depends on `tool_use` and `end_turn` signals.
    

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In the multi-shift orchestration system, the blast radius is stale or polluted state affecting later shift decisions. The fork isolation mechanism is in `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\fork.py`: `fork_for_hypothesis` creates a separate `data/forks/<hypothesis_id>/` directory and copies the base `hot_state.json` into the fork with `shutil.copyfile`, leaving the base state untouched. The README for `04-fork-scratchpad\solution` confirms this behavior, and `scratchpad.py` backs each investigation with an append-only JSON-lines scratchpad, so concurrent hypotheses get separate scratchpads instead of writing into the main state. The kill switch is to discard the fork directory or ignore its summary; because the base hot state is never mutated by the fork, a bad hypothesis investigation cannot corrupt the main orchestration checkpoint.


---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
   → My first System 1 runs could end with an incomplete claim because the model sometimes reached `end_turn` before calling a terminal tool such as route or escalate. I fixed this in `Build a Claims Intake Agent with a stop_reason-Driven Loop\exercises\03-dynamic-decomposition\solution\claims_intake\run.py` by checking `session.terminal_called` after `run_loop(...)`; if no terminal tool was called, the runner adds a follow-up user message telling the model: “The claim is not complete yet. Continue processing and call the appropriate terminal tool before ending the turn.” After that change, the System 1 run completed all traces: `Evidence\system 1\summary.md` shows **8 fixtures processed**, and `claim_04_neighbor_injury` completed as `routed` in **4 turns** with estimated cost **$0.0184**. This fixed the incomplete-output failure by enforcing completion at the harness/runner layer instead of accepting a premature final turn.


20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would add an explicit “fork promotion” step to the System 4 orchestration design instead of letting fork results be handled only as informal summaries. In `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\04-fork-scratchpad\solution\shift_monitor\fork.py`, a hypothesis fork copies the base `hot_state.json` into `data/forks/<hypothesis_id>/`, and `shift_monitor\scratchpad.py` keeps fork notes in separate append-only JSON-lines scratchpads; that isolation is good, but I would also require a validated merge/adoption record before any fork conclusion can affect the main shift state. The need for strict state boundaries showed up elsewhere too: `Evidence\system 4\hot_state.json` stayed small at **643 bytes**, and `recovery.py` uses the **30-minute** stale-state threshold to decide resume versus fresh start. My architectural change would be to make every cross-session state transition explicit: resume, fresh start, fork creation, and fork adoption should each leave a typed manifest entry, so a bad or stale investigation cannot silently influence later shifts.
