# CI test replay

The Travis tox test environments activate `ci_replay.py` with
`-p ci_replay --maxfail=1`. The plugin checkpoints collected test IDs before
running tests, saves failures and collection errors promptly, and stops pytest
after the first failure. The existing `runtests.py` coverage command and full
test suite remain in place. The separate lint and documentation jobs do not
load the plugin.

## Order

1. Previously failed tests or collection files.
2. Tests left pending by an interrupted run.
3. Tests in the changed-source risk group, with historic failures first.
4. Remaining tests, ordered by their recorded failure counts.

There is no verified source-to-test dependency map. Any source change or
unknown Git ancestry conservatively puts the suite in the changed-source risk
group. This can make the initial group large; the plugin still preserves the
full suite. Tests that pass during replay are not run twice, and only an
actual passing call without setup or teardown failures clears an old failure.

## Storage and limits

History is scoped by repository, branch, tox environment, and Python runtime.
It records test IDs, pending work, bounded failure counts, and the last 12
run contexts (repository, branch, PR, commit, run, job, lane, and Python
version). It does not store assertions, exception bodies, logs, or environment
dumps. Checkpoints use an atomic rename.

This repository does not configure a persistent Travis cache for
`.ci-replay-cache`, so jobs may start cold and use normal discovery order.
History is a best-effort optimization, not a durable error ledger or a
correctness requirement. Corrupt or unsupported history fails the test
invocation before tests continue; missing history starts a full run.

Travis runs matrix entries as separate jobs and may continue sibling jobs
that were already running when one job fails. This change contains failure
within the current pytest invocation; it does not add branch-wide,
commit-wide, or pull-request-wide cancellation.

## Focused adapter checks

Run `python -m pytest ci_replay_tests/check_replay.py
--confcutdir=ci_replay_tests -q -x` with a modern Python and pytest. These
isolated checks need no application database or Redis service. The plugin
remains compatible with this application's Python 2.7 floor.
