# documentdb-contribution-log

# Contribution [#1]: Add compatibility test for $lastN (second pass)

**Contribution Number:** 1 
**Contributor:** Subarna  
**Issue:** [documentdb/functional-tests#199](https://github.com/documentdb/functional-tests/issues/199)
**Status:** [Phase III in progress]

---

## Why I Chose This Issue

My background in data science and interests in clincal data drew me to DocumentDB, a MongoDB-compatible engine where both document queries and vector search coexist. I wanted to understand how compatibility testing works in a database.

## Understanding the Issue

### Problem Description

The issue is to add a compatibility test for the $lastN operator in aggregation pipelines. This operator is used to return the last N elements of an array. The test should verify that the implementation of $lastN in DocumentDB behaves as expected and is compatible with MongoDB's behavior.

### Expected Behavior

The `$lastN` operator should have comprehensive compatibility coverage in both of its forms — the **array expression** (used inside `$project`) and the **accumulator** (used inside `$group`) — not just a single happy-path check. Coverage should match the depth of the non-`N` sibling `$last`, which is tested across ~10 property groups (null/missing, sort order, special numerics, boundaries, arrays, mixed types, BSON constants, empty group) plus dedicated error, BSON-type, and return-type files.

For `$lastN` specifically, the suite should verify:
- `n` larger than the array/group size returns all available elements; `n == 1` returns a single-element list.
- Ordering is correct and deterministic after an explicit `$sort`.
- Null/missing `input`, empty arrays, non-array `input`, nested arrays, and mixed BSON types are handled per spec.
- Invalid arguments fail with the documented error codes — missing `n` (`N_ACCUMULATOR_MISSING_N_FIRSTN_FAMILY_ERROR = 5787906`) and invalid `n` such as zero, negative, or non-integer (`N_ACCUMULATOR_INVALID_N_ERROR = 7548606`).
- Legitimate DocumentDB-vs-MongoDB divergences are marked with `pytest.mark.engine_xfail(...)` rather than weakened assertions.

Additionally, a full `pytest documentdb_tests` run should **collect cleanly** so comprehensive runs aren't interrupted.

### Current Behavior

`$lastN` has only a **single happy-path smoke test per form** — no edge-case, type, or error coverage:
- `expressions/array/lastN/test_smoke_expression_lastN.py` — one test, `{"$lastN": {"n": 2, "input": "$values"}}` returns the last two elements (`pytestmark = pytest.mark.smoke`).
- `accumulators/lastN/test_smoke_accumulator_lastN.py` — one test, `$lastN` inside `$group` returns the last two grouped values.

The other `N`-family operators (`$firstN`, `$maxN`, `$minN`, `$topN`, `$bottomN`) are in the same smoke-only state — they were stubbed and never expanded to match `$last`.

A secondary infrastructure bug blocks running the whole suite: `compatibility/result_analyzer/test_analyzer.py` uses `@pytest.mark.unit`, but `unit` is not declared in the `markers =` list in `documentdb_tests/pytest.ini`. Because that file enables `--strict-markers`, the undeclared marker is a hard collection error that aborts the entire run (`'unit' not found in markers configuration option`).

Also observed during reproduction: the custom `--connection-string` / `--engine-name` options are only registered when the collection path includes `documentdb_tests/` (their `pytest_addoption` lives in `documentdb_tests/conftest.py`, and there is no `testpaths` in the root `pyproject.toml`). Running `pytest` without a path under that folder yields `unrecognized arguments: --connection-string --engine-name`.

### Affected Components

- **`$lastN` smoke tests (existing, kept as-is):**
  - `documentdb_tests/compatibility/tests/core/operator/expressions/array/lastN/test_smoke_expression_lastN.py`
  - `documentdb_tests/compatibility/tests/core/operator/accumulators/lastN/test_smoke_accumulator_lastN.py`
- **New comprehensive test files to add:**
  - `documentdb_tests/compatibility/tests/core/operator/accumulators/lastN/test_accumulator_lastN.py`
  - `documentdb_tests/compatibility/tests/core/operator/accumulators/lastN/test_accumulator_lastN_errors.py`
  - `documentdb_tests/compatibility/tests/core/operator/expressions/array/lastN/test_expression_lastN.py`
- **Reference pattern (model to follow):** `documentdb_tests/compatibility/tests/core/operator/accumulators/last/` (`test_accumulator_last.py`, `test_accumulator_last_errors.py`, etc.).
- **Framework helpers used:** `documentdb_tests/framework/executor.py` (`execute_command`), `documentdb_tests/framework/assertions.py` (`assertSuccess`, `assertSuccessNaN`, `assertFailureCode`), `documentdb_tests/framework/parametrize.py` (`pytest_params`), and the `AccumulatorTestCase` dataclass in `accumulators/utils/accumulator_test_case.py`.
- **Error codes:** `documentdb_tests/framework/error_codes.py` (`N_ACCUMULATOR_MISSING_N_FIRSTN_FAMILY_ERROR`, `N_ACCUMULATOR_INVALID_N_ERROR`).
- **Config to fix:** `documentdb_tests/pytest.ini` — register the `unit` marker (and note: include a path under `documentdb_tests/` when invoking pytest so `conftest.py` options load).
- **Fixtures relied on:** `collection`, `database_client`, `engine_client` from `documentdb_tests/conftest.py`.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges I faced, how I solved them TBD]

### Steps to Reproduce

1. From the repo root, list the existing `$lastN` coverage — only two files exist, both named `test_smoke_*` (one expression, one accumulator), each with a single happy-path function.
2. Run the `$lastN` smoke tests:
   ```cmd
   pytest documentdb_tests\compatibility\tests\core\operator\expressions\array\lastN documentdb_tests\compatibility\tests\core\operator\accumulators\lastN --connection-string mongodb://localhost:27017 --engine-name documentdb -v
   ```
3. **Observed result:** both smoke tests pass, but there are no edge-case, type, or error tests — confirming the coverage gap. Attempting a full `pytest documentdb_tests` run instead aborts during collection with `'unit' not found in markers configuration option`.

### Reproduction Evidence

- **Commit showing reproduction:** [TBD]
- **Screenshots/logs:**
  - `2 passed, 34127 deselected, 1 error` — the two `$lastN` smoke tests pass; the lone collection error is the unrelated `result_analyzer/test_analyzer.py` `unit`-marker issue.
  - Earlier failures while reproducing: `unrecognized arguments: --connection-string --engine-name` (options not loaded when no `documentdb_tests/` path is passed), and a full-drive collection blow-up caused by using bash-style `\` line continuations in `cmd.exe`.
- **My findings:**
  - `$lastN` has smoke-only coverage in both forms; the rest of the `N`-family is the same.
  - `--strict-markers` + an undeclared `unit` marker in `pytest.ini` blocks full-suite collection.
  - Custom pytest options live in `documentdb_tests/conftest.py` and only register when the collection path includes that folder (no `testpaths` in root `pyproject.toml`).

---

## Solution Approach

### Analysis

The root cause is not a code defect in the engine — it is a **coverage gap**. `$lastN` was added to the suite with only a single smoke test per form (expression and accumulator), each asserting one happy path. The whole `N`-family (`$firstN`, `$lastN`, `$maxN`, `$minN`, `$topN`, `$bottomN`) is in this same stubbed state, whereas the non-`N` sibling `$last` was expanded into a full multi-file suite. So the behavior I'm "fixing" is missing assertions: edge cases, type handling, and error conditions for `$lastN` are simply never exercised, which means a regression or a DocumentDB/MongoDB divergence in those areas would go undetected.

A second, independent issue surfaced while reproducing: a full `pytest documentdb_tests` run aborts during collection because `result_analyzer/test_analyzer.py` uses an undeclared `@pytest.mark.unit` marker and `pytest.ini` enables `--strict-markers`. This isn't caused by `$lastN`, but it blocks running the comprehensive suite, so it's in scope to fix.

### Proposed Solution

Add comprehensive test files for both `$lastN` forms, modeled on the existing `$last` suite and using the framework's parametrized-test-case idiom (`AccumulatorTestCase` + `pytest_params`), so each case is a declarative data row rather than a bespoke function. Drive success cases through `execute_command` + `assertSuccessNaN` and error cases through `assertFailureCode` against the documented `N`-family error codes. Where DocumentDB and MongoDB legitimately diverge, mark the case with `pytest.mark.engine_xfail(engine=..., reason=..., raises=AssertionError)` instead of weakening the assertion. Keep the existing smoke tests as the fast `-m smoke` feature-detection layer. Separately, register the missing `unit` marker in `pytest.ini` so the full suite collects cleanly.

### Implementation Plan

**Understand:** `$lastN` (expression form in `$project`, accumulator form in `$group`) returns the last N elements but is only smoke-tested with one happy path each. Goal: bring it to parity with `$last`'s coverage and unblock full-suite collection.

**Match:** Follow `accumulators/last/` — `test_accumulator_last.py` groups cases into property lists (null/missing, sort order, special numerics, boundaries, arrays, mixed types, BSON constants, empty group), concatenates them into one `*_SUCCESS_TESTS` list, and runs them through a single `@pytest.mark.parametrize("test_case", pytest_params(...))` function. Errors live in `test_accumulator_last_errors.py` using `assertFailureCode`. `engine_xfail` usage mirrors the `$toDate` tests (`expressions/type/toDate/test_toDate_basic.py`).

**Plan:**
1. Add `accumulators/lastN/test_accumulator_lastN.py` — success property groups (`LASTN_N_VALUE_TESTS`, `LASTN_SORT_ORDER_TESTS`, `LASTN_NULL_MISSING_TESTS`, `LASTN_MIXED_TYPE_TESTS`, `LASTN_EMPTY_GROUP_TESTS`) → `LASTN_SUCCESS_TESTS`, parametrized via `pytest_params`, asserted with `assertSuccessNaN`.
2. Add `accumulators/lastN/test_accumulator_lastN_errors.py` — missing `n` (`5787906`) and invalid `n` (zero/negative/non-integer/decimal, `7548606`), asserted with `assertFailureCode`.
3. Add `expressions/array/lastN/test_expression_lastN.py` — `n` vs length, `n == 1`, empty/missing/null/non-array `input`, nested arrays, invalid `n`.
4. Add `engine_xfail` marks to any case where DocumentDB and MongoDB legitimately differ.
5. Register `unit: Unit tests for framework/result-analyzer tooling` in the `markers =` block of `documentdb_tests/pytest.ini`.
6. Leave both `test_smoke_*_lastN.py` files unchanged.

**Implement:** [https://github.com/sbhattap/functional-tests/tree/feature/add-lastn-compatibility-test]

**Review:** Self-review checklist:
- [ ] New files follow the `$last` structure and naming (`test_<form>_lastN[_errors].py`).
- [ ] Every test case has a `msg` (required by `BaseTestCase.__post_init__`).
- [ ] Ordering assertions always include an explicit `$sort` (determinism).
- [ ] Error tests assert specific codes, not just "any failure."
- [ ] No production/engine code touched; smoke tests untouched.
- [ ] Full suite collects cleanly after the `pytest.ini` change.

**Evaluate:** See Testing Strategy below — run the new files directly against DocumentDB and MongoDB, confirm clean full-suite collection, and verify `engine_xfail` placement is correct on both engines.

---

## Testing Strategy

### Unit Tests

Accumulator success (`test_accumulator_lastN.py`):
- [ ] `n` larger than group size returns all available values
- [ ] `n == 1` returns a single-element list
- [ ] Correct ordering after an explicit `$sort`
- [ ] Null/missing `input` values handled per spec
- [ ] Mixed BSON types in the grouped field
- [ ] Empty group returns an empty list

Accumulator errors (`test_accumulator_lastN_errors.py`):
- [ ] Missing `n` → `5787906` (`N_ACCUMULATOR_MISSING_N_FIRSTN_FAMILY_ERROR`)
- [ ] Invalid `n` (zero / negative / non-integer / decimal) → `7548606` (`N_ACCUMULATOR_INVALID_N_ERROR`)

Expression (`test_expression_lastN.py`):
- [ ] `n` vs array length (greater, equal, less); `n == 1`
- [ ] Empty `input` array → empty list
- [ ] Missing / null / non-array `input`
- [ ] Nested arrays preserved
- [ ] Invalid `n` values rejected

### Integration Tests

- [ ] Run against DocumentDB (`--engine-name documentdb`) — all non-`xfail` cases pass
- [ ] Run against MongoDB (`--engine-name mongodb`) — confirm `engine_xfail`-marked cases `xfail` and everything else passes (validates divergence markers)

### Manual Testing

Commands used (Windows `cmd.exe` — single line, no `\` continuations; path under `documentdb_tests/` so `conftest.py` registers the custom options):

Run just the `$lastN` tests (both forms):
```cmd
pytest documentdb_tests\compatibility\tests\core\operator\expressions\array\lastN documentdb_tests\compatibility\tests\core\operator\accumulators\lastN --connection-string mongodb://localhost:27017 --engine-name documentdb -v
```

Confirm full-suite collection is clean after the `unit` marker fix:
```cmd
pytest documentdb_tests -m smoke --connection-string mongodb://localhost:27017 --engine-name documentdb
```

Baseline (already verified this session): both existing smoke tests pass —
`test_smoke_accumulator_lastN` PASSED, `test_smoke_expression_lastN` PASSED.

---

## Implementation Notes

### Week [3] Progress

[Implementation almost completed for the accumulator test files of $lastN but not ready to be pushed yet]

### Week [Y] Progress

[TBD]

### Code Changes

- **Files modified:** [test_accumulator_lastN_errors.py, test_accumulator_lastN.py]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why I chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How I addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What I learned technically]

### Challenges Overcome

[What was hard and how I solved it]

### What I'd Do Differently Next Time

[Reflection]

---

## Resources Used

- [TBD]
