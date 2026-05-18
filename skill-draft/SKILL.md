---
name: unity-batchmode-worktree
description: "Use when collaborating on a Unity project where the user keeps the Editor open in one git worktree and you work in a second via batch mode. Triggers: running Unity batch-mode compile/test commands, triaging Editor errors, editing Assets files, merging between user's branch and yours, stray test-log.txt / TestResults*.xml / Unity logs in git status."
---

# Unity Batch Mode in a Git Worktree

## Overview

Unity locks `Library/` per-Editor, so two Editor processes can't share a project. The workaround: user keeps the Editor open on one worktree (`main`), you work in a sibling worktree (`dev`) and run Unity in `-batchmode` for compile/test. Work flows both directions via git.

**Core principle:** both worktrees are shared, but at any moment one side may be **locked** — mid-play-mode, mid-batch-run, or holding uncommitted edits. Writing to a locked worktree clobbers state or breaks a test. Your job: know the lock state before you write. **If unsure, ask.**

## When to Use

- Any Unity project with a sibling `<project>/` + `<project>2/` worktree layout
- Running compile checks, EditMode/PlayMode tests, or `-executeMethod` via batch mode
- Triaging an error the user saw in their live Editor
- Committing Unity-serialized assets (`.asset`, `.prefab`, `.unity`, `ProjectSettings/*.asset`)
- Syncing between branches

## Layout

```text
<ProjectRoot>/
├── <project>/     <- user's worktree, branch `main`, Editor lives here
│   └── <project>/ <- Unity project root
└── <project>2/    <- your worktree, branch `dev`, batch mode
    ├── <project>/ <- Unity project root (mirror)
    └── TestResults/  <- gitignored home for ALL batch-mode output
```

`dev` is a **scratch branch**, not long-lived. It gets squash-merged into `main` at sync points and immediately reset.

## Lock State

The Editor is open the entire session, so "Editor running" is not a signal. Lock state = git state + conversation.

**User's worktree is LOCKED if any of these:**

1. `git status` in their tree shows uncommitted changes that aren't yours
2. They said "hold on", "I'm testing", "don't touch anything", or equivalent
3. **Default when no handoff has been established this turn** — absence of permission is a lock

**Handed off (safe to write) only when all hold:**

- Explicit authorization *this turn* ("merge it", "go ahead", "I'm ready")
- Their `git status` is clean, or dirty state is approved-to-overwrite
- They haven't since said anything that re-locks

**Authorization is per-action, not a standing license.** "Yes, merge now" covers the merge, not the next write an hour later. **If unsure, ask** — a clarifying question is cheap, clobbering a test session is not.

**Your worktree:** yours by default, but before a long batch run tell the user *"batch run in my worktree, ~Nm, don't edit there until I report back."* Never start a second Unity process against a worktree while one is running — Library lock fails hard. Track background shell ids.

## Sync Workflow — Squash-Merge and Reset

1. **Commit freely on `dev` during a session.** Granularity doesn't matter, it all squashes. Small commits are a working tool for your own rollback/bisect, not history the user has to review.
2. **At a sync point, the user squash-merges in their worktree:** `git merge --squash dev && git commit -e`. Produces one commit on `main`, authored by them, with a message they approve.
3. **Immediately reset `dev`:** in your worktree, `git fetch && git reset --hard main`. Old agent commits are orphaned and GC'd. This is expected.
4. **User → you sync uses the same reset.** Never merge `main` into `dev` — always reset.

Mid-session rollback only works **before** the squash. If you need a pre-squash checkpoint to revisit later, tell the user so they can stash it (tag, branch, patch).

### Squash commit message

Lead with the user-facing change. Put agent-specific context in an `Agent notes:` trailer — design decisions, things tried and discarded, convention fix-ups, known follow-ups. Propose the message before they commit.

```text
Implement full combat system (melee, projectile, dash, block, heal, nuke)

Agent notes:
- Reworked vitals server-authoritative; damage/heal sync via ClientRpc
- Added Projectile NetworkBehaviour + prefab, registered in DefaultNetworkPrefabs
- Tuned Strength on all damaging ability SOs
- Known follow-up: invisibility toggles all renderers, should exclude local view
```

### Sync-point checklist

1. Confirm lock state on user's worktree
2. `git log --oneline main..dev` **and** `dev..main` — read subject lines. Overlap = user may have manually copied work, your commits may be redundant
3. Produce proposed squash message with `Agent notes:` trailer
4. User executes `git merge --squash dev && git commit -e`
5. Resolve asset conflicts per "Committing Unity Assets" below. Prefer user's on-disk prefab for `fileID` references
6. If `.cs` files changed, batch-mode compile in your worktree **before** the reset (while dev commits still exist to diff)
7. `git fetch && git reset --hard main` in your worktree. Verify `git log --oneline main..dev` is empty
8. Re-run pre-session divergence check before the next write

## Pre-Session Checklist

1. **Read `<user-worktree>/<project>/CLAUDE.md`** — project conventions (naming, architecture, test rules). Skipping it has caused convention-violation cleanup commits in past sessions.
2. **`git status` in BOTH worktrees.** Unstaged changes you didn't make this session = STOP and ask.
3. **`git log --oneline main..dev` and `dev..main`.** Know how the branches have diverged.
4. **Confirm `<your-worktree>/TestResults/` exists and is gitignored.**

## Batch Mode Commands

Route ALL output to `<your-worktree>/TestResults/` with descriptive timestamped names — never Unity's default log path, never the Unity project root. See [`batch-mode-commands.md`](batch-mode-commands.md) for copy-paste command templates (compile check, EditMode, PlayMode, executeMethod) and runtime expectations.

Rule of thumb: cold compile 30–120s, EditMode 60–180s, PlayMode 120–300s. Use `run_in_background` with a recorded shell id.

## Reading the User's Editor Log

The user's live Editor log is **external** to the worktree:

- **Windows:** `%LOCALAPPDATA%/Unity/Editor/Editor.log` (current) + `Editor-prev.log`
- **macOS:** `~/Library/Logs/Unity/Editor.log`
- **Linux:** `~/.config/unity3d/Editor.log`

`tail -n 200` is usually enough — errors are near the end.

## Committing Unity Assets

Unity serializes as YAML with `guid`/`fileID` references. Pitfalls:

- **LF → CRLF churn.** ProjectSettings and `.asset` files may show as modified with only line-ending changes. `git diff -w <file>` — if empty, `git checkout --` it. Don't commit whitespace noise.
- **`fileID` conflicts on merges.** Both branches adding the same Resources-loaded SO with different `fileID`s → the fileIDs point to **different components inside the same prefab**. `grep -E '&[0-9]+|!u!' <prefab>` to see the prefab's internal IDs; match the fileID to the component type the field expects (e.g. a `MyProjectile` field needs the MonoBehaviour fileID, not the root GameObject).
- **Meta files.** Stage `<file>` + `<file>.meta` together. Never one without the other.
- **Never commit test artifacts.** `TestResults*.xml`, `test-log.txt`, anything under `TestResults/` stays gitignored.

## Parallel Agents via Persistent Warm Worktrees

When dispatching parallel subagents on a Unity project, **do not** use the `Agent` tool's `isolation: "worktree"` option — it spins up a fresh worktree per agent, which means each one pays a 5–30 minute cold `Library/` reimport before doing any work. The cost dwarfs any wall-clock savings from parallelism.

Instead, **pre-provision a small pool of long-lived sibling worktrees**, each with its own warm `Library/`, and have agents `git checkout` the branch they need:

```text
<ProjectRoot>/
├── liminal/      <- user's, branch main, warm Library
├── liminal2/     <- yours, branch dev, warm Library
├── liminal3/     <- agent pool slot, warm Library
└── liminal4/     <- agent pool slot, warm Library
```

Each pool slot is created once with `git worktree add`, opened in Unity once to populate `Library/`, and then reused across sessions. Agents do `git fetch && git checkout <branch>` inside their assigned slot — Unity reimports only the files that actually changed, which is fast.

**Rules for the pool:**

- One agent per slot at a time. The Unity Library lock still applies — never run two batch-mode processes against the same slot.
- Agents must reset/clean their slot before checking out a new branch (`git reset --hard && git clean -fd`) so leftover state from a previous session doesn't leak.
- Don't delete the pool slots between sessions. The whole point is keeping `Library/` warm.
- Plans that branch into truly independent tasks benefit most. Plans with hard sequential dependencies (Task N+1 imports symbols Task N defines) should still run sequentially in a single worktree — parallelism buys nothing there.

### Reserving a warm worktree

Pool slots are shared across sessions. Before doing anything in a slot, **claim it with a reservation marker** so a parallel session (or future-you) doesn't pick the same slot and stomp on in-progress work:

1. **Survey.** `git worktree list` to enumerate slots; for each candidate check `ls <slot>/.claude-reserved` and `git -C <slot> status -sb`. A slot is free only if there's no marker **and** the working tree is clean. Detached HEAD at an older commit is fine — that's the warm-pool resting state.
2. **Claim.** Write `<slot>/.claude-reserved` with: session date, branch about to be checked out, plan/task reference, and expected duration. One short file. Never stage or commit it (add `.claude-reserved` to `.git/info/exclude` if it isn't already globally ignored).
3. **Prep.** In the slot: `git fetch && git reset --hard && git clean -fd`, then `git checkout -B <branch> origin/main` (or the needed base). Unity will reimport only changed files on the next batch-mode run — `Library/` stays warm.
4. **Release.** When the work is merged (or abandoned), delete `.claude-reserved` and reset the slot back to a clean detached state at `main` so the next session finds it warm and obviously free. If handing a branch off mid-stream, leave the marker in place and update its contents to describe the handoff.

A reservation is advisory — a sticky note, not a lock. If you find a stale marker (older than ~24h with no activity on its branch), it's probably abandoned; flag it to the user before overwriting.

## Gotchas

- **`sed -i` on Git Bash for Windows silently empties files.** Use the Edit tool for in-place text replacement.
- **Never delete `Library/`, `Temp/`, or `UserSettings/`** — machine-local caches; deleting `Library/` forces a 30-minute reimport.
- **Two Unity batch processes against the same worktree** fail on the Library lock. Track shell ids; wait for exit before relaunch.

## Quick Reference

| Task | Do | Don't |
|---|---|---|
| Check user's worktree lock state | `git status` in their tree + recent conversation; ask if unsure | Rely on `Library/UnityLockfile` (Editor is always open) |
| Write to user's worktree | Only with explicit this-turn authorization | Treat prior authorization as standing |
| Run Unity batch mode | `-logFile <TestResults>/<name>.log`, record shell id | Default log path, or second Unity against same worktree |
| Read user's Editor error | `tail` the external `Editor.log` | Read files inside user's worktree |
| Sync dev ↔ main | Squash-merge then `git reset --hard main` on dev | Long-lived dev, rebases, force-pushes, `git merge main` into dev |
| In-place text replace | `Edit` tool | `sed -i` on Windows Git Bash |
| Commit LF/CRLF-only diff | `git checkout --` to discard | Commit the churn |
| Resolve `fileID` conflict | Match fileID → component type in prefab YAML | Pick one side blindly |
| New asset | Stage `<file>` + `<file>.meta` | Commit one without the other |
| Start a session | Read `CLAUDE.md`, `git status` both trees, check divergence | Start editing blind |
