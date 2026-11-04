# Manual check notes - test-coverage batch (#91, #96, #97)

Covers tickets **#91** (vehicle-taxonomy Trim level and item-compatibility), **#96** (approval decision
guard and Tier-2 atomicity) and **#97** (shelf-scoped opname creation and add-shelf guards). Merged to
`main` via PR #72 (merge commit `e90890c`, 2026-11-02). The batch is tests only: no production code,
schema or migration changed, so there is no app screen to click through.

Check off each box as you finish it, fill in **Result** and **Notes**, then paste the file back.

Baseline on `main` after the merge is 22 test files and 265 tests, all passing.

---

## A. Confirm the tests are trustworthy

- [ ] **A.1 Run the suite twice in a row**
  - Steps: from the repo root, run `pnpm test:run` two times back to back.
  - Expect: 22 files and 265 tests passing both times. A second clean run shows the new fixtures delete
    their own rows. The #96 fixture fails loudly if a company-wide approval policy is left behind.
  - Result:
  - Notes:

- [ ] **A.2 Break the approval guard on purpose (#96)**
  - Steps: in `recordApprovalDecisionDB` (`features/approval/DAL/approval-dal.ts:730`), remove the
    `status = 'pending'` condition from the guarded `UPDATE`. Run
    `pnpm test:run tests/integration/approval-services.test.ts`. Then `git checkout` the file.
  - Expect: 2 tests fail, both `Promise.all` race tests (one winner with one decision row, and the
    contradictory approve-versus-reject case).
  - Result:
  - Notes:

- [ ] **A.3 Move the decision insert outside the transaction (#96)**
  - Steps: in the same function, change the decision insert from `tx.insert` to `db.insert`. Run the same
    test file, then `git checkout` the file.
  - Expect: 1 test fails, the rollback-on-throwing-`onDecided` test (a leftover decision row).
  - Result:
  - Notes:

- [ ] **A.4 Disable the add-shelf empty-selection guard (#97)**
  - Steps: in `addOpnameShelvesDB` (`features/stock-adjustment/DAL/stock-adjustment-dal.ts:792-799`),
    change the guard condition to `if (false)`. Run
    `pnpm test:run tests/integration/stock-adjustment-opname.test.ts`. Then `git checkout` the file.
  - Expect: the DAL short-circuit test fails and the other 13 pass.
  - Result:
  - Notes:

---

## B. Read the judgment calls

- [ ] **B.1 `execute_failed` comment assertion (#96)**
  - File: `tests/integration/approval-services.test.ts:596-599`.
  - What to judge: the test asserts `toBeTruthy()` plus `toContain('Item not found')` instead of an exact
    string, because the error factory prefixes the message with
    `[NotFoundError][updateItemSellingPriceService] ::`. An exact match would break on any wording change
    in that prefix.
  - Decision: accept, or tighten to an exact match?
  - Notes:

- [ ] **B.2 Optional `quantity` parameter on `insertStockBatchOnShelf` (#97)**
  - File: `tests/integration/stock-adjustment-opname.test.ts:143-174`.
  - What to judge: the helper stays local to the file and takes a fifth `quantity` parameter, so the
    tests build batches of 10, 5 and 2. A wrong shelf filter then produces a visibly wrong total
    instead of a plausible one.
  - Decision: accept, or move the parameter into the shared `createTestStockBatch`?
  - Notes:

- [ ] **B.3 Shared zone with per-test cleanup (#97)**
  - File: `tests/integration/stock-adjustment-opname.test.ts:213-241`.
  - What to judge: one zone is created in `beforeAll`, and `afterEach` deletes that test's rows. The
    one-active-opname guard runs before the empty-array guard, so a leftover draft would make the next
    test fail on the wrong guard.
  - Decision: accept, or switch to a fresh zone per test?
  - Notes:

---

## C. Decide what to do next

- [ ] **C.1 Stale comment at `features/approval/auto-execute-dispatch.ts:27-28`**
  - Noted during #96 and left alone because the batch was tests only.
  - Decision: fix in a small commit, or ignore?

- [ ] **C.2 `confirmOpnameDB` and `confirmOpnameService` have no tests**
  - Excluded from #97 on purpose. Needs its own ticket.
  - Decision: queue a ticket?

- [ ] **C.3 Smaller gaps left on purpose**
  - `createOpnameDB`'s empty-array branch is covered by code reading only, because its own service guard
    blocks an empty array before the DAL.
  - An already-deleted opname id is not tested separately from a missing id. Both hit the same
    `getAdjustmentByIdDB` null branch.
  - Decision: accept both?

- [ ] **C.4 `testing.md` backlog entries are now out of date**
  - `.claude/rules/testing.md` still lists the approval decision concurrency guard and the shelf-scoped
    opname empty-selection guard as untested. Ticket #94 set the precedent of rewriting an entry to say
    automated coverage now exists.
  - Decision: update both entries?
