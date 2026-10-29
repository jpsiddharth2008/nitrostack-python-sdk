# Issue #73 implementation verification

Validated locally on 2026-10-29. Branch: `feature/issue-73-widget-registry`, based
on upstream `develop` at `63ecd9240d98bbe06c879089dd887494c7e99293`.
No PR #94, spillover, OAuth-client or HTTP-factory branch commits are included.

## Acceptance criteria

- Actual starter, Pizzaz and OAuth/flight-booking modules bootstrap in isolated
  subprocesses with networking blocked. Their manifests contain exactly 1, 3
  and 6 unique routes, respectively, with URI and mode-correct MIME type.
- Pizzaz helper-built options retain their CSP/border metadata and examples.
- Shared `order-summary` appears once with both tool names. Registry unit tests
  additionally verify merging all explicit examples and deterministic ordering.
- Empty applications keep their resource catalog unchanged and report an empty,
  loaded Python registry. Reserved-URI collisions are rejected only in widget apps.
- Modern and legacy HTTP clients can list/read the manifest. A real stdio
  subprocess serves both operations with no non-protocol stdout. Additional MCP
  server instances have the same resource handlers. Existing widget HTML reads
  and metadata continue to match their components.

## Results

All runs use official MCP **2.3.0**. Linux runs use local WSL Ubuntu, not hosted
GitHub Actions. Each complete run includes all tests, without ignores,
deselections, or new skips/xfail.

| Environment | Complete-suite result | Coverage |
|---|---|---|
| Linux Python 3.10.21 | 847 passed | 86.44% |
| Linux Python 3.11.16 | 847 passed | 86.42% |
| Linux Python 3.12.14 | 847 passed | 86.42% |
| Windows Python 3.10.21 | 847 passed | 86.44% |
| Windows Python 3.11.16 | 847 passed | 86.42% |
| Windows Python 3.12.14 | 846 passed, 1 failed; see below | 86.42% |

- Complete collection: **847 tests**, no errors.
- New feature tests: **22 passed** on Windows 3.12, also included in all complete
  matrix runs. New registry module: **100% statement coverage**.
- Initial combined widget regression selection: **72 passed**, before adding
  two additional registry cases; the final complete runs include all 74 cases.
- `git diff --check` passes. No dependency or version changes were made.
- SHA-256 comparisons confirm Linux tested the same production and new test
  source bytes as the Windows worktree.

## Windows 3.12 socket limitation and baseline

The final full Windows 3.12 run failed the existing
`test_discovery_server_idempotent_start_and_dcr_http` in `tests/test_oauth.py`.
Its localhost HTTP request raised `ConnectionAbortedError: [WinError 10053]`.
An earlier Windows 3.12 run also encountered that error in the existing
`test_discovery_dcr_404_when_disabled_or_missing_client_id`. All widget tests pass.

For comparison, the unchanged `63ecd924...` baseline was exported into a separate
directory and tested with the same Windows Python 3.12.14/MCP 2.3.0 environment:

- Full baseline suite: **825 passed**, 86.22% coverage.
- Baseline OAuth module alone: **34 passed**.
- The two affected baseline OAuth tests were then run together three times.
  Attempts 1 and 2 each passed both tests. Attempt 3 reproduced **the same
  `test_discovery_server_idempotent_start_and_dcr_http` WinError 10053 failure**
  (1 passed, 1 failed).

This establishes that the socket failure also occurs without the feature and is
intermittent on this host. It does not establish its underlying operating-system
or server-lifecycle cause. The Windows 3.12 complete suite is not reported green,
and no OAuth code or tests were changed to hide it.

## Reproduction commands

```powershell
python -m pytest tests --collect-only -o addopts= -q
python -m pytest tests/test_widget_registry.py -o addopts=--timeout=60 -q
python -m pytest tests --cov=nitrostack --cov-report=term-missing --timeout=60 --tb=short -o addopts= -q
```

Run the same complete-suite command on Linux Python 3.10, 3.11 and 3.12. To probe
the Windows failure on the unchanged baseline:

```powershell
python -m pytest tests/test_oauth.py::test_discovery_server_idempotent_start_and_dcr_http tests/test_oauth.py::test_discovery_dcr_404_when_disabled_or_missing_client_id -o addopts=--timeout=60 -q --tb=short
```

Local logs are retained outside tracked sources in `D:/NitroStack/issue73-*.log`
and the worktree's ignored `.pytest_cache/linux-3.*.log`. Hosted CI has not been
run because this implementation has not been pushed or published.
