# Contribution [#]: [Copier Template Caching Issue]

**Contribution Number:** 1  
**Student:** Kartavya Suhagiya  
**Issue:** [[GitHub issue link](https://github.com/copier-org/copier/issues/450)]  
**Status:** [Phase IV] [In Progress]

---

## Why I Chose This Issue

This issue interests me because it focuses on improving the performance and developer experience of Copier. The current implementation downloads or fetches remote Git templates every time they are used, which can introduce noticeable delays, especially when repeatedly generating or updating projects from the same template repository. Performance optimizations that have a direct impact on users are particularly interesting to me because they combine software design, Git internals, and practical engineering considerations.

From a learning perspective, this issue provides an opportunity to gain a deeper understanding of how Copier interacts with Git repositories, how repository caching strategies are implemented, and how features such as Git mirrors and worktrees can be used to improve efficiency. I also hope to learn more about contributing to a mature open-source codebase, including understanding existing architecture, handling edge cases, and collaborating with maintainers during the review process.

---

## Understanding the Issue

### Problem Description

When Copier uses a template stored in a remote Git repository, it performs cloning or fetching operations that can take several seconds, even when the same template has been used previously. This repeated network activity makes project generation and updates slower than necessary.

The issue proposes adding a local cache for Git templates so that Copier can reuse previously downloaded repository data instead of downloading it again for every operation.

### Expected Behavior

Copier should maintain a local cache of remote Git templates. When a template repository is requested, Copier should first check the cache and reuse the existing repository data whenever possible. The cache should be updated using Git fetch operations when needed, and temporary working copies (such as Git worktrees) can be created from the cached repository for project generation or updates.

This would significantly reduce the amount of network traffic and improve the speed of repeated operations involving the same template repository.

### Current Behavior

Currently, Copier performs Git operations directly against the remote repository each time a template is used. Although previous optimizations such as blob-less cloning have improved performance, users still report that operations involving remote templates are considerably slower than using locally available templates.

For example, a user reported that updating from a remote template took approximately 5.5 seconds, while using a locally checked-out template took approximately 0.7 seconds. The absence of a persistent local cache means Copier cannot fully benefit from data that has already been downloaded.

### Affected Components

The primary components involved are:

The Git template retrieval and cloning logic used by Copier.
Code responsible for resolving and preparing template repositories before project generation or updates.
Temporary workspace management used during template processing.
Potential new cache management functionality, including:
Creating and maintaining cached Git mirrors.
Updating cached repositories through fetch operations.
Creating temporary worktrees from cached repositories.
Handling cache cleanup and repository synchronization.

Additional consideration may be required for repository references such as branches, tags, and commits, as well as concurrent access to the cache by multiple Copier processes.

---

## Reproduction Process

### Environment Setup

I set up Copier locally on Windows 11 (PowerShell, Python 3.13, Git 2.52):

```powershell
git clone https://github.com/copier-org/copier.git
cd copier
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e .
python -m pip install pytest pytest-cov pytest-gitconfig pytest-xdist coverage pexpect pre-commit poethepoet
```

**Challenges I faced and how I solved them:**

- **Some VCS tests assume a Unix environment.** A few tests rely on the Unix `true` command, which doesn't exist on Windows; those failures are unrelated to this issue, so I focused on `tests/test_vcs.py`, which runs cleanly.
- **`safe.bareRepository=explicit` in my environment.** Git refused to operate on a bare repository discovered implicitly (`fatal: cannot use bare repository ...`). This bit me later when working with mirrors; I solved it by always passing an explicit `--git-dir` for every operation on the bare mirror.
- **Read-only Git files on Windows.** Git marks pack files read-only, so a naive `shutil.rmtree` (or `rmtree(..., ignore_errors=True)`) silently leaves directories behind. Copier already ships a `handle_remove_readonly` helper for exactly this, which I reused.
- **The test suite runs in parallel.** Copier configures `pytest` with `addopts = "-n auto"` (pytest-xdist), so any caching code has to be safe against two processes touching the same cache at once. This shaped my implementation (see below).

### Steps to Reproduce

I wrote a small benchmark (`benchmark_clone.py`) that calls `copier._vcs.clone()` directly and times it across several runs, comparing a remote Git URL against a local copy of the same repository.

1. Pick a real template (e.g. `https://github.com/copier-org/autopretty.git`).
2. Clone it once into a temp directory to act as the "local template".
3. Time `clone(url, ref)` for the remote URL 5 times, deleting the result between runs (mirroring Copier's temp-clone cleanup).
4. Do the same for the local copy.
5. Compare the timings.

**Observed result (before any change):** every remote run paid the full network clone cost and never improved across runs — remote averaged noticeably slower than local, exactly the "clone → delete → clone again" cycle described in the issue. There was zero reuse between runs.

### Reproduction Evidence

- **My findings:**
  - The single entry point is `clone()` in `copier/_vcs.py`; its only real caller is `Template.local_abspath` in `copier/_template.py`, which allocates a temp dir, calls `clone()`, and later removes it via `Template._cleanup` (registered as a cleanup hook in `copier/_main.py`).
  - Remote vs. local is already distinguishable: `get_repo()` returns a `_PathStr` for local repos/bundles and a plain `str` for remote URLs.
  - Profiling a single run showed where the time actually goes (Windows, autopretty):
    - Remote (warm, after this change): `git remote update` ≈ 1.4s, `git worktree add` ≈ 0.3s, `git submodule update` ≈ 2.4s.
    - Local: `git clone --no-checkout --filter=blob:none` ≈ 0.36s, `git checkout -f` ≈ 0.13s, `git submodule update` ≈ 2.87s.
  - The key insight: caching removes the repeated **object download**; the remaining per-run cost (dominated by `git submodule update`) is fixed and identical for local and remote. The size of the win therefore scales with the template's download size.

---

## Solution Approach

### Analysis

The root cause is that Copier treats every remote-template operation as a throwaway: it clones into a temp directory, uses it, and deletes it. Nothing persists between runs, so even back-to-back invocations of the same template re-download everything. PR #472's `--filter=blob:none` reduced how much is transferred but didn't remove the repeated transfer itself.

### Proposed Solution

Follow the maintainer's suggestion: keep a **persistent local Git mirror** of each remote template and check out **temporary worktrees** from it.

- **First use:** `git clone --mirror <url>` into a per-user cache, then `git worktree add` a temporary checkout.
- **Subsequent uses:** refresh the existing mirror with `git remote update --prune` (cheap — no full re-download), then add a fresh worktree.
- **Local templates are untouched** — they keep the original clone behavior, which also preserves the "include dirty changes" feature.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** Remote templates are re-cloned on every run because the clone is temporary. Make repeated use of the same remote template fast by caching it locally, without changing behavior for local templates.

**Match:** The codebase already (a) distinguishes local vs. remote via `_PathStr`, (b) ships `handle_remove_readonly` for Windows-safe deletion, and (c) depends on `platformdirs` (so a per-user cache dir is the natural home). Git's own `clone --mirror` + `worktree` features provide everything needed.

**Plan:**
1. Add a cache location helper (`get_cache_dir()`) using `platformdirs.user_cache_dir("copier")`, overridable via a `COPIER_CACHE_DIR` env var (for tests/power users).
2. Add deterministic mirror naming (`_mirror_path()` via `sha256(url)`).
3. Add `_is_remote()` to robustly decide whether to cache (handles `_PathStr` and plain-string local paths).
4. Add `get_or_create_mirror()` to create/refresh the mirror.
5. Add `_clone_via_cache()` to produce a temporary worktree.
6. Branch inside `clone()`: remote → cache path; local → original path.
7. Add tests; keep `Template`/`Worker` cleanup unchanged.

**Implement:** All logic lives in `copier/_vcs.py`; `copier/_template.py` and `copier/_main.py` are untouched. (Branch/commits: _to be linked when I push my fork._)

**Review:** Existing `tests/test_vcs.py` passes unchanged (local behavior preserved); new tests cover the cache path; no new runtime dependency added (`platformdirs` was already required).

**Evaluate:** Re-run the benchmark to confirm warm runs avoid the object download, and run the full VCS suite under pytest-xdist.

---

## Testing Strategy

### Unit Tests

All new tests live in `tests/test_vcs.py` and use a local repo exposed via a `file://` URL, so they exercise the cache path **without network access** (no `@pytest.mark.impure` needed). They redirect the cache to a temp dir via `COPIER_CACHE_DIR`.

- [x] `test_is_remote` — remote URLs are cached; on-disk paths and `_PathStr` are not.
- [x] `test_remote_clone_creates_and_reuses_mirror` — first use creates the mirror; second use reuses it (verified with a sentinel file inside the mirror that must survive).
- [x] `test_remote_clone_checks_out_ref` — a specific ref/tag is checked out correctly into the worktree.
- [x] `test_remote_clone_prunes_stale_worktree` — after the temp worktree dir is removed (simulating Copier's cleanup), the next use prunes the stale registration and succeeds.
- [x] `test_remote_clone_recovers_from_corrupt_mirror` — a partial/corrupt cache entry is discarded and rebuilt instead of crashing.

### Integration Tests

- [x] Existing `test_removes_temporary_clone` (a real `Worker.run_copy` against a remote template) still passes — the temporary worktree is cleaned up.
- [x] Existing `test_dont_remove_local_clone` and the dirty/shallow/tilde local tests still pass — local behavior is unchanged.
- [x] Full `tests/test_vcs.py` suite: **20 passed** under parallel pytest-xdist.

### Manual Testing

- Ran `copier copy --defaults --trust gh:copier-org/autopretty <dst>` twice. The first run built the mirror; the second reused it. Both produced identical, correct output, confirming the end-to-end path (not just `clone()`).
- Ran the benchmark across template sizes:
  - **autopretty (small):** remote warm runs ≈ 1.5s/run faster than no-caching (~26%).
  - **copier (larger):** remote warm runs ≈ 1.2–2.3s/run faster (the cold→warm gap widens with repo size).
  - **local templates:** unchanged within noise, as expected.

---

## Implementation Notes

### Progress Summary

I implemented the whole feature inside `copier/_vcs.py` and added tests in `tests/test_vcs.py`, keeping the change surface deliberately small (no edits to `_template.py`/`_main.py`).

What I built, in order:
1. Cache helpers (`get_cache_dir`, `_mirror_path`, `_is_remote`).
2. `get_or_create_mirror()` and `_clone_via_cache()`.
3. The remote branch in `clone()`.
4. Hardening discovered during testing (see decisions below).
5. New tests + a reproduction/benchmark script.

### Challenges faced and decisions made

- **Parallel-test race → atomic mirror creation.** Because the suite runs with `-n auto`, two tests cloning the same URL collided on the cache (`destination already exists`). I made mirror creation atomic: clone into a staging directory, then `rename()` it into place; if another process won the race, discard mine and reuse theirs. This also lightly mitigates the concurrency concern raised in the issue, without needing a full lock.
- **`safe.bareRepository=explicit` → explicit `--git-dir`.** Operating on the bare mirror via `cd`/`-C` failed under this setting. I pass `--git-dir <mirror>` for every mirror operation, which is portable regardless of the user's config. (Interestingly, the tests masked this because `pytest-gitconfig` provides an isolated config; only the real benchmark surfaced it.)
- **Worktree vs. pre-allocated dir (decision "A1").** `git worktree add` refuses an existing path, but the caller pre-creates an empty temp dir. I remove that empty placeholder right before adding the worktree, so `clone()`'s contract and all callers stay identical.
- **Cleanup of worktrees (decision "B1").** Rather than teach `Template._cleanup` about worktrees, I left it as-is (it still `rmtree`s the temp dir) and made `get_or_create_mirror()` run `git worktree prune` on every refresh. This is self-healing even if a process crashes mid-run.
- **Full mirror, not blobless.** For a *reused* mirror, `--filter=blob:none` would re-fetch blobs over the network on every checkout, defeating the purpose. A full `--mirror` pays once and is then fully offline for worktrees.
- **Freshness via `git remote update --prune`.** The mirror must stay current or Copier would resolve stale tags/branches and `copier update` would break. The refresh transfers almost nothing for an up-to-date mirror but keeps correctness.
- **Corrupt-cache recovery.** A partial/interrupted mirror could leave an `objects/` dir that isn't a valid repo. I added `_is_valid_mirror()` (`git rev-parse --is-bare-repository`) so such entries are discarded and rebuilt instead of crashing.

### Code Changes

- **Files modified:**
  - `copier/_vcs.py` — new imports (`os`, `hashlib.sha256`, `shutil.rmtree`, `platformdirs.user_cache_dir`, `handle_remove_readonly`); `CACHE_DIR_ENV_VAR`; `get_cache_dir`, `_mirror_path`, `_is_remote`, `_force_rmtree`, `_is_valid_mirror`, `get_or_create_mirror`, `_clone_via_cache`; remote branch added to `clone()`.
  - `tests/test_vcs.py` — `_make_remote_repo` helper and 5 new tests (listed above).
- **Approach decisions:** documented above (A1 + B1, atomic creation, `--git-dir`, full mirror, freshness, corrupt recovery).
- **Scope:** local mirror caching only. Full inter-process locking is intentionally deferred (and partially mitigated by atomic creation), as the issue allows.

---

## Pull Request

**PR Link:** _[to add when submitted]_

**PR Description (draft):**

> **Cache remote Git templates locally (#450)**
>
> Remote templates were re-cloned into a throwaway temp directory on every run, so repeated use of the same template re-downloaded everything. Following @yajo's suggestion, this caches a persistent `git --mirror` clone per template under the user cache dir and checks out temporary worktrees from it.
>
> - First use: `git clone --mirror` into the cache (created atomically via staging + rename), then `git worktree add` a temporary checkout.
> - Later uses: `git remote update --prune` (no full re-download) + a fresh worktree.
> - Local templates keep the original behavior (including dirty-change handling).
> - Cache dir via `platformdirs`, overridable with `COPIER_CACHE_DIR`. No new dependency.
> - Robustness: explicit `--git-dir` (works with `safe.bareRepository=explicit`), stale-worktree pruning, Windows read-only-safe cleanup, and corrupt-mirror recovery.
> - Existing VCS tests unchanged; 5 new network-free tests added (20 passing under pytest-xdist).
>
> Out of scope (deferred): full inter-process locking for concurrent mirror updates, mitigated here by atomic creation.

**Maintainer Feedback:**
- _[to fill in as feedback arrives]_

**Status:** _[Awaiting review / Iterating / Approved / Merged]_

---

## Learnings & Reflections

### Technical Skills Gained

- How Copier resolves and prepares templates: the `clone()` → `Template.local_abspath` → cleanup-hook lifecycle.
- Git internals for caching: `clone --mirror`, `worktree add/prune`, `remote update`, and the `safe.bareRepository` security setting.
- Writing concurrency-safe filesystem code (atomic staging + rename) and Windows-safe deletion of read-only Git files.
- Designing tests that exercise networked code paths offline using `file://` URLs.

### Challenges Overcome

- Diagnosing failures that only appeared under pytest-xdist (the cache race) and only with my real Git config (`safe.bareRepository`), but not in isolated test runs.
- Understanding *why* the benchmark looked "flat" at first (the cache wasn't cleared between runs, so there was never a cold baseline) and then quantifying where the time actually goes.

### What I'd Do Differently Next Time

- Clear/redirect the cache from the very first benchmark to make cold-vs-warm obvious immediately.
- Consider a freshness TTL (skip `git remote update` if the mirror was refreshed very recently) as a follow-up optimization, and propose full locking as a separate PR.

---

## Resources Used

- The issue and discussion: https://github.com/copier-org/copier/issues/450
- Prior optimization PR (#472, `--filter=blob:none`): https://github.com/copier-org/copier/pull/472
- Git docs: `git-clone(1)` (`--mirror`), `git-worktree(1)`, `git-remote(1)` (`update`), and `safe.bareRepository` in `git-config(1)`.
- `platformdirs` documentation for per-user cache directories.
- Forked Repo link :- https://github.com/DrKat0m/copier/tree/feat/cache-git-templates-450
