**English** · [简体中文](README.zh-CN.md)

# Multi-Phase Controller Skills for Codex

This repository is a small add-on skill pack for [`gsd-build/get-shit-done`](https://github.com/gsd-build/get-shit-done). It adds a lightweight multi-phase controller workflow for Codex and similar agents that need to coordinate one main controller session with several task-specific worktrees.

The controller is built around `.mpc/mpc.md`, `.mpc/mpc-archive.md`, and `.mpc/mpc.lock`. The goal is to make subtask splitting, progress tracking, regression, and archiving predictable across parallel worktrees without turning this repo into a replacement for GSD itself.

## Highlights

- Codex-first skill layout
- GitHub-friendly installation via `/skill install`
- Small, focused extension to GSD instead of a fork or rewrite
- Bilingual repository docs in English and Chinese

## Included skills

- `mpc-master-start`: split one requirement or plan document into MPC subtasks and suggest worktree commands
- `mpc-master-progress`: show a read-only overview of active or archived task status
- `mpc-master-regress`: review one completed MPC task and move it to `已回归` after explicit approval
- `mpc-master-archive`: move one regressed task from `.mpc/mpc.md` to `.mpc/mpc-archive.md`
- `mpc-slave`: run inside one worktree and advance exactly one task through the worker-side state machine

## Repository layout

```text
.
├─ README.md
├─ README.zh-CN.md
├─ docs/
│  ├─ MPC_SPEC.en.md
│  └─ MPC_SPEC.zh-CN.md
└─ skills/
   ├─ mpc-master-start/
   ├─ mpc-master-progress/
   ├─ mpc-master-regress/
   ├─ mpc-master-archive/
   └─ mpc-slave/
```

## Install with Codex

This is a multi-skill repository. For Codex-style installation, target an individual skill folder under `skills/`, not the repository root.

Published repository:

```text
https://github.com/luobote55/multi-phase-controller
```

Install examples:

```text
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-start
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-progress
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-regress
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-archive
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-slave
```

Recommended installation set:

1. Install all five skills for the full controller loop.
2. Keep the folder names stable after publishing, because the install URL uses the folder basename as the installed skill name.
3. Test at least one GitHub install URL after the first push.

## How to use

1. Prepare a requirement or plan document in the target project repository.
2. Run `/mpc-master-start path/to/plan.md` to create subtasks inside `.mpc/mpc.md`.
3. Create the suggested git worktrees manually.
4. In each worktree, run `/mpc-slave` to update progress and propose completion when ready.
5. In the main controller session, run `/mpc-master-progress` to inspect the whole board or one task.
6. After a worker task reaches `完成`, run `/mpc-master-regress task_name`.
7. After regression passes, run `/mpc-master-archive task_name`.

## Runtime files

These files belong in the target project that uses the skills, not in this skill repository:

- `.mpc/mpc.md`
- `.mpc/mpc-archive.md`
- `.mpc/mpc.lock`

## Recommendations

- Keep task name, worktree directory name, and branch name identical.
- Let only the defined writer skills modify `.mpc/` files.
- Use one subtask per worktree.
- Treat this repo as a companion to GSD, not a replacement for GSD.
- Do not commit generated `.mpc/` runtime state into this repository.

## Publishing notes

- Keep every installable skill under `skills/<skill-name>/`.
- Keep `SKILL.md` and `agents/openai.yaml` inside each skill folder.
- Add repository-level docs at the root or under `docs/`; avoid adding extra docs inside each skill folder.
- If you rename a skill folder after publishing, the previous install URL will break.

## Documentation

- Chinese README: [README.zh-CN.md](README.zh-CN.md)
- English spec: [docs/MPC_SPEC.en.md](docs/MPC_SPEC.en.md)
- Chinese spec: [docs/MPC_SPEC.zh-CN.md](docs/MPC_SPEC.zh-CN.md)
