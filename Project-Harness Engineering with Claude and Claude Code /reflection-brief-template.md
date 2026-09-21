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
   → In Evidence\system 2\budget.json, the four sections are case_facts: 204, resolved_refund: 384, resolved_subscription: 534, and active: 15,789 tokens. The rule is to keep the active issue byte-exact because it dominates the assembled context and is the only section still changing, while the resolved issues are summarized because they are stable and far smaller. This matches the README’s design: preserve the unresolved thread verbatim, compress the completed history.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The regressed question is Q6: “What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?” In Evidence\system 2\eval.jsonl, the model answered with the exact token in_progress and passed, but in Evidence\system 2\eval_control.jsonl it failed by saying there was no status token and paraphrasing the issue instead. That proves the facts block matters: preserving the active issue’s exact status token is necessary for correctness. The control run shows that without the verbatim fact, the model drifts into a generic explanation and loses the required structured answer.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In ...\.claude\rules\react.md, the rule frontmatter is path-scoped with a glob, not tied to one directory. This is better than a directory-level CLAUDE.md because the rule applies only to files matching the pattern, so the same convention can be reused across the repo without being duplicated in every folder. It is more precise, easier to maintain, and less likely to leak into unrelated files.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → → In `.../.claude/skills/deploy-check/SKILL.md`, the skill declares `context: fork` and `allowed-tools` limited to `Read`, `Grep`, `Glob`, and read-only `git`/`gh` commands. Running in a forked, read-only context keeps the noisy diff and status exploration out of the main session while returning only a final `pass | fail` summary. Without that boundary, the parent session gets polluted with intermediate output and the check can accidentally become mutating or misleading. The read-only allowlist makes the deploy gate a safe inspection step instead of a side-effectful operation.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → → In this config, the project-level scope is the repo-owned `.claude` setup, such as `.../.claude/skills/deploy-check/SKILL.md`, which is committed with the team’s config and applies to the repository. The user-level scope is the personal variant described in the same file: a parallel skill in `~/.claude/skills/deploy-check-strict/`, which is outside the repo and does not affect teammates. The validator makes this distinction by location: repo-root `.claude/...` is project scope, while `~/.claude/...` is user scope. That split lets the team share defaults while keeping one-off personal rules private.

### System 4 — Orchestration

11.  **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In the System 4 run, `defects_since` returned **0 new defects**, shown by `run_shift done: shift=C new=0`, while `data/warm.sqlite` contains **40 warm-tier rows**. The query uses the `idx_defects_ts` index on `defects(ts)` to filter by the `since` timestamp before orchestration builds the model context. Because the run used `since=2026-09-20T17:03:24Z` and the stored defects are from April 2026, none matched; the model therefore did not receive the full historical table.

12.  **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In `recovery.py`, persisted state older than approximately **30 minutes** is treated as stale and causes a fresh start; newer state can be resumed. A fresh start with an injected summary avoids carrying stale tool state, partial work, or outdated assumptions while preserving the important checkpoint information. This is safer for repeated shift runs because the new run starts from bounded, validated context.

13.  **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → After the System 4 run, `Evidence/system 4/hot_state.json` was **1 KB**, below the approximately **5 KB** budget. Keeping this checkpoint bounded prevents restart context, token cost, and recovery risk from growing across indefinite shift runs.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify. 
    → Model: `Build a Claims Intake Agent with a stop_reason-Driven Loop\exercises\03-dynamic-decomposition\solution\claims_intake\loop.py` is the model layer because the agent’s control flow is determined by `stop_reason`: `tool_use` continues the loop and `end_turn` terminates it. 
    → Harness: `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot` is the harness layer because it compresses older context and preserves the active issue verbatim to keep the working set high-signal. 
    → Orchestration: `Build a Multi-Shift Quality Monitoring System with Claude Orchestration\recovery.py` is the orchestration layer because it persists state and decides whether to resume or start fresh based on staleness. Together they form a stack: the model decides, the harness reduces noise, and orchestration keeps state resumable and bounded.
    

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the run-time enforcement in the orchestration system: the read-only allowlist and the stale-state threshold are enforced in code, not by persuasion. A prompt-guided behavior is the context strategy in the retail support copilot: the system tells the model to preserve active facts verbatim and summarize older history, which influences what it sees but does not enforce it at runtime. Code is right when the rule must always hold, such as safety and mutation boundaries; prompt is right when the goal is to bias model behavior within a bounded context. The distinction is: enforce invariants in code, instruct preferences in the prompt.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → In `Evidence\system 2\budget.json`, System 2 reduced the context from **38,708 baseline tokens** to **16,893 assembled tokens**, a **56.36% reduction**, while preserving the active issue verbatim. In the System 4 run artifact, `defects_since` returned **0 defects** from **40 warm-tier rows**, so SQL filtering kept the full history out of the model context. The principle is the same—retain high-signal information while discarding or summarizing unnecessary history but the mechanisms differ: System 2 trims tokens within a conversation, whereas System 4 filters database records and persists only bounded cross-session state.
    

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    →  A test guarantees the loop invariant in Build a Claims Intake Agent with a stop_reason-Driven Loop\exercises\01-loop: the model must not decide continue-vs-stop by scanning raw text; it must use the structured stop_reason signal. A single run can look successful by chance, even if the loop is relying on brittle string matching, because the particular model output may not trigger the bug in that session. The test matters because it locks in the control flow under edge cases and prevents silent drift before shipping. That is the sort of reliability a lucky one-off run does not reveal.
    

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In the multi-shift orchestration system, the blast radius is stale or corrupted state: if recovery.py resumes from outdated state, the agent may act on stale assumptions, keep old scratchpad data, or continue from the wrong checkpoint. The kill switch is the staleness threshold in recovery.py plus the read-only/forked execution pattern in the deploy-check skill, which prevents the session from mutating the main branch while the system inspects state. The enforced boundary is the small persisted state in hot_state.json and the decision to restart fresh when staleness exceeds the threshold. That keeps the damage contained and makes the failure visible before it spreads.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → I struggled with Python version and had to pin httpx==0.27.2 to get the system running.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would make the context and stale-state policy more explicit in code instead of relying on prompt discipline alone. In System 2, budget.json shows the model is only safe when the active issue is preserved verbatim and older history is summarized; in System 4, recovery.py decides whether to resume or restart based on a staleness threshold, which means stale state can silently poison the next run if the boundary is not enforced strongly enough. My architectural change would be to fail closed on stale or oversized state: reject a resume when the snapshot exceeds the byte budget or freshness threshold, and require an injected summary or a fresh start instead. That would reduce silent drift and make the system’s safety boundaries explicit, rather than trusting the prompt and the state file to remain “good enough” over time.
