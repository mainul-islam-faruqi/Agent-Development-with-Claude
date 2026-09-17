---
description: Run the e-commerce team's PR review checklist against a pull request or local diff
argument-hint: <pr-number | "HEAD~1..HEAD" | path/to/file>
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git show:*)
  - Bash(git status:*)
  - Bash(gh pr diff:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)
---

---
# TODO: Write the YAML frontmatter for this command. It must include:
#
#   description:   one line, ≥10 characters, naming what the command does
#   argument-hint: short description of how to pass a target (PR ref, diff range, or file path)
#   allowed-tools: a list of READ-ORIENTED tools only — no Edit, Write, NotebookEdit, or
#                  unrestricted Bash.
#
# Granular Bash sub-command allowlisting is the canonical pattern. For example:
#   - Bash(git diff:*)
#   - Bash(git log:*)
#   - Bash(gh pr view:*)
# This lets you allow READ operations on git without unlocking the full shell. The
# command should be able to run `git diff`, `git log`, `gh pr view`, `gh pr checks`,
# etc. — but NOT `git push`, `git commit`, or arbitrary `Bash(...)`.
---

# /review — team PR review

You are reviewing the diff identified by `$ARGUMENTS` against this repository's conventions. Apply the path-scoped rules in `.claude/rules/` (they auto-load based on the files touched) and the shared standards in `.claude/standards/`.

## Phase 1 — Interview pattern

**Before reporting any findings, ask clarifying questions when intent is ambiguous.** It is cheaper for the reviewer to ask than for the author to argue with a confidently-wrong review.

Trigger the interview pattern when any of these are true:

- The diff adds or removes a dependency (`package.json` or `pyproject.toml` changes). Ask: *"What problem does this dependency solve that the existing stack does not?"*
- The diff changes a public API response shape or a database column. Ask: *"Who consumes this? Are there clients that will break?"*
- The diff is a refactor with no test changes. Ask: *"Was the behavior already covered by tests? If yes, point me at them. If no, should we add coverage in this PR?"*
- The diff touches `src/api/` *and* `src/components/` *and* `src/db/` together. Ask: *"Is this one logical change, or three small PRs that landed together?"*

If the author has already answered these in the PR description, acknowledge that and proceed without re-asking.

<!--
TODO: Write the interview-pattern subsection. The reviewer agent should ask clarifying
      questions BEFORE reporting findings whenever the author's intent is ambiguous.

      Why this matters (and why it's not obvious on first read): without an interview
      step, the reviewer commits to a confidently-wrong verdict on ambiguous changes —
      e.g. a refactor that touched no test files. The author then has to argue against
      a "no test coverage" finding when the behavior was already covered upstream. The
      interview pattern asks first.

      Name at least three concrete trigger conditions where the reviewer should
      interview rather than report. Suggested triggers:
        - dependency changes (package.json or pyproject.toml diffs)
        - public API response shape changes or DB column changes
        - refactors with no corresponding test changes
        - changes that touch multiple surfaces (API + components + DB) at once
-->

## Phase 2 — Review criteria

### Must report

- **Bugs:** logic errors, off-by-one, null-deref risk, race conditions, missing `await`, unhandled rejections, wrong error code.
- **Security:** raw SQL with user input, missing auth check on a new handler, secrets in code, `dangerouslySetInnerHTML` introduced, CORS broadened.
- **Convention violations the path-scoped rules catch:** handler returning `{ status: 500 }` instead of throwing `ApiError`; component with `onClick` on a `div`; test using `setTimeout`; repository function used outside `src/db/`.
- **Breaking changes** not flagged in the PR description.
- **Migration safety:** any migration that adds NOT NULL without a default to a non-empty table.

### Skip

- Minor formatting (Prettier handles it).
- Naming bikeshed when the existing name is fine ("rename `getCart` to `fetchCart`").
- Personal stylistic preferences not encoded in `.claude/rules/` or `.claude/standards/`.
- TODO/FIXME comments added with a tracked ticket reference — flag only if there's no ticket.
- Test descriptions that could be clearer but are still accurate.

The point of the skip list: false-positive reviews destroy author trust. If you wouldn't block on it, do not raise it.

<!--
TODO: Write the explicit must-report vs. skip taxonomy. The point of explicit criteria
      (vs vague "be thorough" instructions) is that two teammates running this command
      on the same PR should reach the same verdict.

      Must report: at minimum, name categories for bugs (logic errors, missing await,
      race conditions, off-by-one), security (raw SQL with user input, missing auth,
      secrets in code, dangerouslySetInnerHTML), convention violations the path-scoped
      rules catch, breaking changes, and migration safety.

      Skip: at minimum, name categories the reviewer should NOT block on — minor
      formatting (Prettier handles it), naming bikeshed when the name is fine, personal
      stylistic preferences not encoded in rules/standards, and TODOs that already have
      tracked tickets.

      The skip list is what keeps false positives from destroying author trust.
-->

## Phase 3 — Bundling: interacting vs. independent findings

**Interacting findings** (those that overlap or affect the same fix) → bundle into one detailed message with all context. Example: a missing `await` and a missing error handler on the same function are interacting — the fix changes both lines, the author needs to see both together.

**Independent findings** (separate functions, separate files, no shared fix) → report sequentially as discrete items. Reporting independent issues one-at-a-time keeps each piece of feedback actionable and lets the author land fixes incrementally.

Default heuristic: if fixing finding A changes the code that finding B references, they are interacting. Otherwise they are independent.
<!--
TODO: Write the bundling guidance. The contrast is hard to internalize without a
      concrete pair of examples (which you'll write in the Examples section below).

      Interacting findings: those that overlap or affect the same fix. Bundle into
      ONE detailed message with all context. Example: a missing `await` and a missing
      transaction wrapper on the same function — the author needs to see them together
      because the fix changes both lines.

      Independent findings: separate functions, separate files, no shared fix. Report
      SEQUENTIALLY as discrete items so each piece of feedback is actionable.

      Give a one-line heuristic for telling them apart (e.g. "if fixing A changes the
      code B references, they are interacting").
-->

## Phase 4 — Output format

For each finding, produce:

```
[severity] path/to/file.ts:line
finding: <one-sentence description>
fix: <one-sentence recommendation>
```

Severity is `bug` | `security` | `convention` | `breaking`. End with a one-line verdict: `approve`, `comment`, or `request-changes`.

## Examples


### Example 1 — convention violation (independent)

**Input:**
```diff
-export async function getOrder(id: string) {
-  const order = await db.query("SELECT * FROM orders WHERE id = $1", [id]);
-  if (!order) return { status: 404, body: null };
-  return { status: 200, body: order };
-}
+export async function getOrder(id: string) {
+  const order = await db.query("SELECT * FROM orders WHERE id = $1", [id]);
+  if (!order) return { status: 404, body: { error: "not found" } };
+  return { status: 200, body: order };
+}
```

**Output:**
```
[convention] src/api/orders/get.ts:3
finding: handler returns a `{ status: 404 }` shape instead of throwing `ApiError(404, ...)`.
fix: replace the `if (!order) return ...` with `if (!order) throw new ApiError(404, "NOT_FOUND", \`order ${id} not found\`);`
[convention] src/api/orders/get.ts:2
finding: handler calls `db.query` directly with raw SQL; this bypasses the repository layer and lets user input reach SQL composition.
fix: call `ordersRepo.findById(id)` from `src/db/orders.ts` instead.
verdict: request-changes
```

### Example 2 — interacting findings (bundle)

**Input:**
```diff
+export async function refundOrder(id: string, amount: number) {
+  const order = ordersRepo.findById(id);
+  const updated = ordersRepo.markRefunded(id);
+  paymentService.refund(order.paymentId, amount);
+  return { status: 200, body: updated };
+}
```

**Output:**
```
[bug] src/api/orders/refund.ts:2-4
finding: three interacting issues on the same code path —
  (1) `ordersRepo.findById` is not awaited; `order` is a Promise, so `order.paymentId` is undefined;
  (2) `ordersRepo.markRefunded` is also not awaited;
  (3) the three writes are not wrapped in `withTransaction`, so a paymentService failure leaves the DB marked-refunded while no money moved.
fix: wrap the whole body in `await withTransaction(async (tx) => { ... })`, await each repo call, and order the writes so paymentService.refund runs *before* markRefunded.
verdict: request-changes
```

These are bundled because the fix changes the same three lines and the author needs to see them together to understand the ordering constraint.


<!--
TODO: Write at least TWO concrete input/output examples using the format above.

Per Task 3.5, examples are the difference between a command that produces
consistent output across team members and one that produces wildly varying output.

Example 1 — INDEPENDENT findings. Show a diff with two unrelated convention
violations on the same file (e.g. a handler returning a `{ status: 404 }` shape
instead of throwing ApiError, AND a raw SQL call that bypasses the repository
layer). Report each as a separate `[convention]` finding. End with a verdict.

Example 2 — INTERACTING findings. Show a diff where multiple problems share a
single fix (e.g. a refundOrder handler that has missing awaits AND no transaction
wrapper AND wrong ordering between paymentService and DB writes). Bundle them
into ONE `[bug]` finding that lists the three issues together because the fix
changes the same three lines. End with a verdict.
-->

## Notes

- This command is project-scoped (`.claude/commands/review.md`) so everyone on the team gets the same review. If you want to layer personal preferences (e.g. a stricter accessibility check on your own PRs), put a `~/.claude/commands/review-strict.md` in your home directory — it will not affect teammates.
- If a finding requires running code or modifying files to verify, ask the author to run a specific check rather than attempting it from the review session. This command's `allowed-tools` are read-only by design.
