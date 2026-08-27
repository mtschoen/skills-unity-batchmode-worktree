> [!NOTE]
> **This skill has been retired and its content reaped.** The persistent warm
> worktree pool protocol lives in
> [skills-orchestration](https://github.com/mtschoen/skills-orchestration) under
> `fleet-orchestration/`, genericized to any project with costly first-build
> state. The lock-state material lives in the same repo under `project-lock/`.
> The Unity-specific half (batch-mode invocation, NUnit result parsing, Editor
> log paths, and asset commit hygiene) moved into the Unity project that used
> it, as its own `docs/BatchMode.md`.
> See [skills-dev#25](https://github.com/mtschoen/skills-dev) for the rationale.
> This repository is archived and read-only.

# unity-batchmode-worktree

A skill for collaborating on a Unity project where the user keeps the Editor open in one git worktree and the agent works in a second via batch mode - working around Unity's per-Editor `Library/` lock without two Editors fighting over one project.

## When it fires

Running Unity batch-mode compile/test commands, triaging Editor errors, editing Assets files, merging between the user's branch and yours, or seeing stray `test-log.txt` / `TestResults*.xml` / Unity logs in `git status`.

## What it does

Tracks worktree lock state (git state + conversation), routes all batch-mode output to a gitignored `TestResults/`, runs the squash-merge-and-reset sync workflow, handles Unity YAML asset/meta/`fileID` commit pitfalls, and provides a warm-worktree pool pattern for parallel agents (avoiding cold `Library/` reimports).

The authoritative spec is [`SKILL.md`](SKILL.md).

**Repo:** <https://github.com/mtschoen/skills-unity-batchmode-worktree>
