**English** · [简体中文](README.zh-CN.md)

# Multi-Phase Controller Skills for Codex

This repository is a companion skill pack for [`gsd-build/get-shit-done`](https://github.com/gsd-build/get-shit-done). Its role is to help GSD projects record and synchronize multi-phase work: take GSD-split task lists, normalize them into `<repo-root>/.mpc/mpc.md`, and keep shared progress accurate across multiple conversations. It is not intended to replace the core GSD workflow.

The shared controller files are `<repo-root>/.mpc/mpc.md`, `<repo-root>/.mpc/mpc_archive.md`, and `<repo-root>/.mpc/.lock.md`. The `.mpc/` directory always lives under the target project's Git repository root, even when a skill is triggered from a subdirectory. `mpc-master-regress` and `mpc-master-archive` remain available, but they are optional governance features rather than the primary purpose of this repository.

## Highlights

- Import and normalize GSD-split task lists
- Keep a shared `mpc.md` as the source of truth for multi-conversation progress
- Favor single-directory execution for parallel task work
- Retain optional regression confirmation and archival workflows

## Included skills

- `mpc-master-start`: organize and import GSD-split task lists into `<repo-root>/.mpc/mpc.md`, then list every newly added task
- `mpc-master-progress`: read the shared task board without modifying it
- `mpc-master-regress`: optionally review one completed task and move it to `已回归` after explicit approval
- `mpc-master-archive`: move one regressed task from `<repo-root>/.mpc/mpc.md` to `<repo-root>/.mpc/mpc_archive.md`
- `mpc-slave`: advance one recorded task by explicit task name and synchronize its progress

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

1. Install all five skills if you want the full recording and governance toolkit.
2. Keep folder names stable after publishing, because install URLs use the folder basename as the installed skill name.
3. Test at least one GitHub install URL after the first push.

## How to use

1. Prepare a GSD plan document or an already split task list in the target repository.
2. Run `/mpc-master-start path/to/plan.md` to normalize those tasks into `<repo-root>/.mpc/mpc.md`.
3. Expect the output in this order:
   - `本次新增任务清单`
   - `最新任务树`
   - `首轮可执行任务与 /mpc-slave 命令`
4. Open one conversation per first-wave task and run `/mpc-slave task_name`.
5. Continue each task in its own conversation. Run `/mpc-slave task_name` again whenever you want to refresh progress or propose completion.
6. In the main coordination conversation, run `/mpc-master-progress` to inspect the whole board or one task.
7. If you want optional controller-side regression confirmation, run `/mpc-master-regress task_name`.
8. If a task has already passed regression and should leave the active board, run `/mpc-master-archive task_name`.

## Prompt requirements

All task prompts generated or surfaced by MPC skills must explicitly point the worker back to GSD skills.

- `开始提示词` must begin with: `利用gsd skills的能力，实现需求：<summary>`
- Auto-generated or refreshed `下一步提示词` must preserve the same prefix
- `完成提示词` must explicitly say the conclusion is based on outputs produced with GSD skills

Recommended templates:

```text
利用gsd skills的能力，实现需求：<summary>。先确认边界与现状，再推进实现，并在过程中持续同步可用于更新 mpc.md 的进展信息。
```

```text
利用gsd skills的能力，实现需求：<summary>。结合当前已完成内容、剩余工作与阻塞项继续推进下一步，并准备同步最新进度。
```

```text
基于利用gsd skills推进该任务后的产出，请确认是否已经满足需求：<summary>；请给出完成依据、风险与是否可进入下一阶段的结论。
```

## Runtime files

These files belong under the target project's repo-root `.mpc/` directory, not inside the published skill sources unless this repository itself is the target project:

- `<repo-root>/.mpc/mpc.md`
- `<repo-root>/.mpc/mpc_archive.md`
- `<repo-root>/.mpc/.lock.md`

## Recommendations

- Treat this repository as a GSD companion for recording and synchronization, not as a GSD replacement.
- Keep all controller files under the target project's repo-root `.mpc/` directory.
- Let only the defined writer skills modify files under `<repo-root>/.mpc/`.
- Use one active conversation per task at a time.
- When the source document already contains GSD-split tasks, preserve that order, granularity, and semantics whenever possible.
- If two tasks touch the same file, core module, dependency manifest, generated artifact, shared export surface, or otherwise require strict sequencing, model them as serial tasks instead of parallel tasks.
- Keep generated `.mpc/` runtime state out of published skill sources; this repository ignores `.mpc/` by default.

## Publishing notes

- Keep every installable skill under `skills/<skill-name>/`.
- Keep `SKILL.md` and `agents/openai.yaml` inside each skill folder.
- Add repository-level docs at the root or under `docs/`; avoid extra READMEs inside skill folders.
- If you rename a skill folder after publishing, the previous install URL will break.

## Documentation

- Chinese README: [README.zh-CN.md](README.zh-CN.md)
- English spec: [docs/MPC_SPEC.en.md](docs/MPC_SPEC.en.md)
- Chinese spec: [docs/MPC_SPEC.zh-CN.md](docs/MPC_SPEC.zh-CN.md)
