# Main compatibility and CPU checks

PR: [#20755](https://github.com/vllm-project/vllm-ascend/pull/20755).

- Ascend main integrated: `219f44cabdca9957ee771fb0559ab30f1f71c4e5`.
- Verified vLLM pin: `24420b7a341af37a9bc3186e6fc6062512e0aa58`.
- Compatibility fix head: `9951818a65f6cb15997832b36cdd39f3511f84bc`; circular-test correction: `a69cadf34689e07f7665353f454b9d1553de9a2f`.
- Validation ran in the existing Pro AI CPU container without NPU devices. Model services were not restarted.

## Changes

- Added the CI-required `[Feature]` title prefix. The original hosted run stopped at title validation before checking code.
- Restored native receive-registration lifetime: request completion no longer removes `_loading_req_ids` before receive completion arrives. The existing regression now covers state reuse disabled and enabled.
- Updated constructor-bypassing test fixtures and the explicit C8 block-size expectation.
- Completed Store type annotations, separated page/state layout variable names, and used the native Transformers GLM configuration type. No tensor layout or numerical operation was changed.
- Expanded an ambiguous `TE` abbreviation in a backend error message for codespell.

## CPU results

| Scope | Result |
| --- | --- |
| AscendStore suite, Mamba splitter and focused MTP/runner tests | 1,052 passed, 12 skipped |
| Full V1 runner, block table, both proposer files and Mamba configuration | Initially 238 passed, 6 failed, 13 skipped; all six failures were the C8 fixture/expectation fixed below |
| Mamba configuration after that fixture fix | 42 passed |
| Distinct test cases, latest outcome across these runs | **1,275 passed, 25 skipped, 0 remaining failures** |
| Complete repository `bash format.sh ci` | Passed on Pro AI; [log](format-final.log) |
| PR diff secret scan | Passed; [log](gitleaks-pr-diff.log) |
| Store mypy, Python 3.10 and 3.11 | Passed, 36 source files per version |
| Examples mypy, Python 3.12 | Passed, 40 source files |
| Tests mypy, Python 3.12 | Passed, 1,105 source files |
| Production mypy, Python 3.12 | One environment-dependent error in unchanged `vllm_ascend/batch_invariant.py`: the preinstalled optional `batch_invariant_ops` extension has no typing marker; no remaining errors in this PR's files |

Counts overlap across runs; [cpu-summary.json](cpu-summary.json) deduplicates by test identity and uses the last result. The Store run preceded the typing-only edits; the runner run included them. Logs retain the initial failing configuration cases and the successful targeted rerun.

- [Store CPU log](cpu-fixed.log) / [JUnit](cpu-fixed.xml)
- [Runner integration log](runner-integration.log) / [JUnit](runner-integration.xml)
- [C8 configuration rerun](mamba-config-fixed.log) / [JUnit](mamba-config-fixed.xml)
- [Production mypy](mypy-fixed-vllm_ascend.log), [examples](mypy-fixed-examples.log), [tests](mypy-tests-final.log)
- [Store Python 3.10](mypy-store-3.10.log), [Store Python 3.11](mypy-store-3.11.log)

The container reused its installed CPU dependencies and repository test fixtures. Missing lint/test typing tools were installed only in the task's tooling environment. For the ARM64 host, the repository's downloaded x86 Gitleaks executable was replaced in the tooling PATH with official ARM64 Gitleaks 8.24.2; repository checks were not bypassed.

## Commands

Run with the selected Ascend and vLLM source trees on `PYTHONPATH` in the CPU test environment:

```bash
python -m pytest -q tests/ut/distributed/ascend_store \
  tests/ut/patch/platform/test_patch_mamba_block_aligned_split.py \
  tests/ut/spec_decode/test_eagle_proposer.py::TestPrepareNextTokenIdsPadded \
  tests/ut/spec_decode/test_extract_hidden_states_proposer.py::test_prepare_next_token_ids_padded \
  tests/ut/worker/test_model_runner_v1.py::TestDrafterMaxModelLen
python -m pytest -q tests/ut/worker/test_model_runner_v1.py \
  tests/ut/worker/test_block_table.py \
  tests/ut/spec_decode/test_eagle_proposer.py \
  tests/ut/spec_decode/test_extract_hidden_states_proposer.py \
  tests/ut/patch/platform/test_patch_mamba_config.py
python -m pytest -q tests/ut/patch/platform/test_patch_mamba_config.py
bash format.sh ci
mypy --follow-imports skip --check-untyped-defs --python-version 3.12 vllm_ascend
mypy --follow-imports skip --check-untyped-defs --python-version 3.12 examples
mypy --follow-imports skip --check-untyped-defs --python-version 3.12 tests
```

The earlier NPU quality and performance tables remain measurements of the archived r1 build. No NPU/model experiment was repeated for this compatibility update.


## Hosted CI follow-up

[Run 41604522827](https://github.com/vllm-project/vllm-ascend/actions/runs/41604522827) passed pre-commit and complete mypy checks for Python 3.10, 3.11 and 3.12 (production, examples and tests). The local optional-extension typing error did not occur in the official lint environment.

Both CPU matrices found the same two obsolete cases in `tests/ut/core/test_circular_buffer.py`, which still expected every circular slot to become PAD. Main pin: 15,545 passed, 2 failed, 109 skipped. Release v0.32.0: 15,522 passed, 2 failed, 129 skipped.

The test-only correction checks wrapped draft positions, padding and dispatch to the native circular mapping kernel. Production mapping code is unchanged. Pro AI checks: [19 circular/block-table cases passed](circular-fixed.log); the final complete-fixture refinement passed [all 5 circular cases](circular-final.log). [File-level repository checks passed](circular-format.log). The corrected head passed both hosted CPU matrices; final results are below.


## Corrected-head hosted result

[Run 41607760523](https://github.com/vllm-project/vllm-ascend/actions/runs/41607760523), head `a69cadf34689e07f7665353f454b9d1553de9a2f`:

| Check | Result |
| --- | --- |
| Pre-commit, including complete mypy for Python 3.10/3.11/3.12 | Passed |
| CPU UT, vLLM main `24420b7a` | **15,547 passed, 109 skipped** |
| CPU UT, vLLM `v0.32.0` | **15,524 passed, 129 skipped** |
| Documentation links | Passed |
| Selected/device tests | Not run; require a maintainer run-enabling label |
| Overall CI gate | Blocked only by the missing run-enabling label |

[Machine-readable summary](hosted-ci-summary.json). No local model service was restarted. The historical NPU measurements above remain unchanged.

The [final gate log](https://github.com/vllm-project/vllm-ascend/actions/runs/41607760523/job/125830395838) confirms `pre-commit: success`, `cpu-ut: success`, `select-tests: skipped`, and all run-enabling labels false. A maintainer must choose `ready-precise`, `ready-all`, or `main2main` to continue. No label was added by this task.
