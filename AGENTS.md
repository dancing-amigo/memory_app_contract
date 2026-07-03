# AGENTS.md

This repository is the shared contract between the Memory System repository and application repositories such as Chat.

## Required operating rule

Before reading or editing this repository, run:

```sh
git pull --ff-only
```

After editing this repository, commit and push the contract changes before you finish the task:

```sh
git status --short
git add <changed-files>
git commit -m "<concise contract change>"
git push
```

If push is blocked by authentication or remote configuration, report that explicitly and leave the local commit ready to push.

## Contract authority

The Memory repository owns the Memory API, MemorySpace, MemoryView, Source, Policy, membership, and lifecycle semantics.

Application repositories own their app UI, app-local sessions, app-local message models, batching, formatting, and how returned memory context is used inside the app.

This repository does not replace the Memory repository's ADRs or product documentation. It records the cross-repository contract that application agents must read before integrating with Memory.

## Editing rules

- Do not redefine Memory internals here.
- Do not add app-specific behavior that requires Memory to branch structurally by app name.
- Keep unresolved integration questions in the relevant `memory/README.md` or `apps/<app_id>/README.md` only when they affect the current contract; otherwise track them outside this repo.
- Keep examples aligned with the current Memory API.
- If a contract change requires Memory API, data model, policy, evaluation, or membership changes, update the Memory repository docs or ADRs in the same work.
