# Lab 12: what to show the TA

Fork: https://github.com/EtixP/f26-lab12
Open in tabs before recitation: **Actions** (https://github.com/EtixP/f26-lab12/actions) and **PR #1** (https://github.com/EtixP/f26-lab12/pull/1).

| Milestone | Evidence | Commit | Result |
|---|---|---|---|
| 1. Pipeline passes | [run 44597229991](https://github.com/EtixP/f26-lab12/actions/runs/44597229991) | `435adaf` | pytest passes |
| 2. Gate fails | [run 44597494035](https://github.com/EtixP/f26-lab12/actions/runs/44597494035) | `05ce5e9` | pytest passes; ruff, mypy, pip-audit fail |
| 2. Gate passes | [run 44597763347](https://github.com/EtixP/f26-lab12/actions/runs/44597763347) | `be4bcde` | all 4 jobs pass |
| 3. Bad change blocked | [PR #1](https://github.com/EtixP/f26-lab12/pull/1) (PR-event [run 44597939506](https://github.com/EtixP/f26-lab12/actions/runs/44597939506)) | `154e277` | pytest and pip-audit pass; ruff and mypy fail |

---

## Milestone 1: the pipeline passes

**Show:** [run 44597229991](https://github.com/EtixP/f26-lab12/actions/runs/44597229991), which passed on commit `435adaf` ("Add CI workflow that runs the test suite").

**Explaining `.github/workflows/ci.yml`:**
- `on: push` and `on: pull_request` run CI on every push to any branch and on every PR.
- `permissions: contents: read` gives the job token read-only access. CI only needs to read the code.
- Steps:
  1. `actions/checkout` fetches the commit.
  2. `actions/setup-python` installs Python 3.11, the minimum in `requires-python` and the version mypy is configured for. It also caches pip downloads, keyed on `requirements-dev.txt`.
  3. `pip install -r requirements-dev.txt` installs pytest, ruff, mypy and pip-audit at exact pinned versions, so CI and every laptop run the same checks.
  4. `pytest` runs the suite.

## Milestone 2: the gate

The gate was added in its own commit (`05ce5e9`), with no change to code or `requirements.txt`. Each check is its own job (`ruff`, `mypy`, `pip-audit`). None of them uses `needs:`, so they run in parallel and one failure can't hide another.

**Failing run:** [run 44597494035](https://github.com/EtixP/f26-lab12/actions/runs/44597494035). On the same commit, pytest passed and all three gate jobs failed.

**Passing run:** [run 44597763347](https://github.com/EtixP/f26-lab12/actions/runs/44597763347) on `be4bcde`, after one commit per fix:
`2434d5c` (ruff fix), `5bf5be1` (mypy fix), `4434815` (regression tests), `2c90a22` (babel removed), `be4bcde` (README tools line).

### What ruff caught: [`refunds/service.py:48`](https://github.com/EtixP/f26-lab12/blob/05ce5e9/refunds/service.py#L48)

```python
def build_statement(booking, refund, tier_name, adjustments: list[str] = []) -> RefundStatement:
    adjustments.append(f"settled under the {tier_name} tier")
```

- **B006, mutable default argument.** Python creates the `[]` once, when the function is defined, so every call that leaves out `adjustments` appends to the same list. A second statement would carry the first statement's notes. I checked this: the second call's notes included `settled under the full tier`.
- **Why the tests missed it:** every test, and `decide_refund`, passes `adjustments=` explicitly, so the default was never used.
- **Fix:** default to `None` and build a fresh list inside the function (copying, so the caller's list isn't changed either).

### What mypy caught: [`refunds/policy.py:102`](https://github.com/EtixP/f26-lab12/blob/05ce5e9/refunds/policy.py#L102)

```python
def refund_cents(booking, request) -> int:
    if booking.interval.has_begun_by(request.requested_at):
        return None      # error: Incompatible return value type (got "None", expected "int")
```

- **What breaks:** a cancellation made after the booking starts (a no-show) returned `None`. `decide_refund` then computes `price - None` and crashes with `TypeError`. I checked this. The docstring and README say a no-show settles at **zero**.
- **Why the tests missed it:** no test cancels at or after the start time. Every policy test cancels ahead of time, and validation doesn't reject late cancellations, so the bad path was reachable but never tested.
- **Fix:** `return 0`.

I also added two regression tests (`tests/test_service.py`) that fail on the old code and pass on the fixed code. That brings the suite from 18 tests to 20.

### What pip-audit found, and the decision

- `babel 2.9.0`: **PYSEC-2021-421** (also CVE-2021-42771 / GHSA-h4m5-qpfp-3mpv). It's a directory traversal in `babel.Locale`: an attacker-controlled locale name can load an arbitrary `.dat` file, which is a pickle, so it can lead to code execution. Fixed in 2.9.1.
- **Decision: remove it.** Nothing in `refunds/` or `tests/` imports babel (`format_cents` formats money itself), so it is a runtime dependency that never runs. Removing it beats the other two options:
  - *Upgrading* keeps a package we don't use, and its future vulnerabilities.
  - *Ignoring with `--ignore-vuln`* would accept a code-execution bug to keep something we don't need.
- `requirements.txt` now has only a comment saying the service has no third-party runtime dependencies. pip-audit reports "No known vulnerabilities found" for it. No `--ignore-vuln` was needed, so `ci.yml` has no accepted-finding comment.

## Milestone 3: blocking the bad change

**Show:** [PR #1](https://github.com/EtixP/f26-lab12/pull/1), `maintenance-credits` → `main` on **EtixP/f26-lab12** (the fork, not the course repo). It is **open and not merged**.

- Branch commit `154e277` "Add maintenance-window credits" is exactly `changes/bad-change.patch` applied.
- The PR shows two CI runs on the same commit: one from the branch push ([44597889401](https://github.com/EtixP/f26-lab12/actions/runs/44597889401)) and one from the PR ([44597939506](https://github.com/EtixP/f26-lab12/actions/runs/44597939506)). In both, **pytest passes, pip-audit passes, ruff fails, mypy fails.**

**What the gate caught in the patch:** it repeats both bug patterns fixed in Milestone 2.
- ruff B006 at `refunds/policy.py:122`: `notes: list[str] = []`, a shared mutable default again.
- mypy at `refunds/policy.py:131`: `return None` when `closure_minutes <= 0`, from a function typed `-> int`.
- The tests pass because the patch adds none and nothing calls the new function yet. Its cover note says wiring it up "lands in a follow-up."

**The two bullets (already in the PR description):**
1. **A bad change the gate still lets through:** wrong business logic that is still well-typed and lint-clean, especially in untested code. For example, `price // duration * lost` (rounding before multiplying, which shortchanges organizers) or a tier boundary changed from `>=` to `>`. ruff and mypy check how code is written, not what it means, and nothing requires new code to have tests.
2. **A check I wouldn't let block a merge: `pip-audit`.** Its verdict depends on a vulnerability database that changes daily, not on the diff. A CVE published tomorrow against an unchanged pin would turn every open PR red, including ones that never touch dependencies. Run it on a schedule and on PRs that change `requirements.txt`, and track what it finds as issues, instead of making it a required check on every merge.

**Extra example if asked "what does your gate miss?":** on main right now, a no-show with a waived reason (e.g. `room-closed`) gets a statement whose tier is **`waived`** but whose refund is **$0**. `tier_for` checks the waiver first, and `refund_cents` checks the no-show first. It's type-correct, lint-clean and untested, so the gate passes it. I left it alone because which rule should win is a policy question for the team.

## Other requirement

- README has an "AI tools used" section: Claude Code (VS Code extension) with Claude Opus 5.5 (`claude-opus-5-5`). If you used other tools too, add them.
