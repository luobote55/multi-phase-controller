# MPC Skills Specification

## Purpose

This document defines the behavior contract for the following five skills:

- `mpc-master-start`
- `mpc-master-progress`
- `mpc-master-regress`
- `mpc-master-archive`
- `mpc-slave`

These skills form a small multi-phase controller layer on top of normal GSD-style work. The controller is centered on `~/.mpc/{project-name}/mpc.md` and `~/.mpc/{project-name}/mpc_archive.md`, so a main controller session and several worktree-bound worker sessions can coordinate without losing task state. `project-name` is always the Git repository name, not the current worktree directory name.

This document is a behavior spec, not a marketing page.

## Managed files

- Live controller file: `~/.mpc/{project-name}/mpc.md`
- Archive file: `~/.mpc/{project-name}/mpc_archive.md`
- Lock file: `~/.mpc/{project-name}/.lock.md`

Rules:

- These files live in a shared directory keyed by the Git repository name, and every worktree from the same repository uses the same controller files.
- The names `mpc.md`, `mpc_archive.md`, and `.lock.md` are fixed.
- `.lock.md` is a temporary write lock and is never created by read-only skills.
- If the files do not exist yet, `mpc-master-start` or the first archive action may create the minimal skeleton.

## Core principles

- The main controller is responsible for task splitting, status inspection, regression, and archiving. It does not do the worker task's day-to-day implementation work for it.
- The main-controller side is split into four skills with strict boundaries:
  - `mpc-master-start`
  - `mpc-master-progress`
  - `mpc-master-regress`
  - `mpc-master-archive`
- Every subtask must map to exactly one worktree.
- `task name`, `worktree name`, `worktree directory name`, and `worktree branch name` must be identical.
- Subtasks should be parallel by default. If there is a real dependency, declare it explicitly with `前序子任务` and `后序子任务`.
- If two subtasks obviously conflict in code, depend on strict ordering, or share the same core edit point, they must be serialized rather than run in parallel.
- `mpc.md` must use a fixed field order. No field may be omitted. Empty values must be recorded as `待填写` or `无`.
- Read-only actions must stay read-only. State transitions must be explicit.
- All timestamps must use `YYYY-MM-DD HH:mm:ss +08:00`.
- Every write action except `mpc-master-progress` must hold `~/.mpc/{project-name}/.lock.md`.

## Task state machine

Each subtask may only use these four states:

- `未开始`
- `进行中`
- `完成`
- `已回归`

The only normal transition path is:

`未开始 -> 进行中 -> 完成 -> 已回归`

Responsibility by skill:

- `mpc-master-start`: create new tasks in `未开始`
- `mpc-slave`: move `未开始 -> 进行中` and `进行中 -> 完成`
- `mpc-master-progress`: read only
- `mpc-master-regress`: move `完成 -> 已回归`
- `mpc-master-archive`: archive tasks that are already `已回归`

Archiving is not a state. It is an action allowed only after regression passes.

## Fixed structure of `mpc.md`

Every task block inside `mpc.md` must preserve this exact field order:

`任务标识 -> 状态 -> 来源计划文件 -> worktree名称 -> worktree目录 -> worktree分支 -> 前序子任务 -> 后序子任务 -> 开始时间 -> 完成时间 -> 回归时间 -> 归档时间 -> 最后更新时间 -> 累计耗时 -> 预计剩余时间 -> 当前进度概况 -> 阻塞事项 -> 下一步提示词 -> 开始提示词 -> 完成提示词 -> 完成摘要 -> 回归摘要 -> 归档状态`

Additional rules:

- `当前进度概况` is at most 5 lines.
- `阻塞事项` is at most 5 lines. Write `无` if there is no blocker.
- `累计耗时` and `预计剩余时间` use `Xd Xh Xm` style text such as `4h 30m`.
- `完成摘要` must cover completed work, remaining risk, self-check results, and what the main controller should focus on during regression.
- `回归摘要` must cover regression scope, evidence, result, and whether archiving is allowed.
- `归档状态` must be either `未归档` or `已归档`.

## `mpc_archive.md`

- Each archived task keeps the same structure as in `mpc.md`.
- `任务总数` means the number of task blocks in the file.
- Archiving is a move, not a copy.
- After archiving:
  - the task is removed from `~/.mpc/{project-name}/mpc.md`
  - the task is appended to `~/.mpc/{project-name}/mpc_archive.md`
  - `状态` remains `已回归`
  - `归档状态` becomes `已归档`
  - `归档时间` is filled in
- Queries and dependency checks must search the union of `~/.mpc/{project-name}/mpc.md` and `~/.mpc/{project-name}/mpc_archive.md`, with the active file checked first.
- If the same task identifier exists in both files, that is a hard data error.

## Concurrent write rules

Even though real contention is usually low, all writes must still be serialized.

1. Every write action except `mpc-master-progress` must acquire `~/.mpc/{project-name}/.lock.md`.
2. The lock file must record at least:
   - owner skill name
   - target task or task list
   - current worktree or branch
   - lock timestamp
3. If the lock already exists, retry every 2 seconds for up to 60 seconds.
4. If the lock survives more than 10 minutes, treat it as stale and stop with an explicit error.
5. After taking the lock, reread the latest `~/.mpc/{project-name}/mpc.md`. Read `~/.mpc/{project-name}/mpc_archive.md` too when needed.
6. Except for `mpc-master-start`, each write transaction may modify only one task block plus file-level metadata.
7. `mpc-master-start` may append multiple new task blocks in one transaction and may minimally update the direct predecessor task's `后序子任务`.
8. Unrelated task content, order, and prompts must not be rewritten.
9. Writes must use temporary files plus atomic replacement.
10. After each successful write, update:
    - file-level `最后更新时间`
    - file-level `任务总数`
    - task-level `最后更新时间`
11. If a pending write is based on stale content, rebuild it against the latest snapshot rather than overwriting.
12. If a lock was held at any point, it must be released before exiting, whether the action succeeded or failed.

## Task splitting rules

`mpc-master-start` must follow these rules when splitting work:

- Task names must be globally unique.
- Reserved names such as `mpc`, `mpc_archive`, or `lock` are forbidden.
- Task names may only use lowercase letters, digits, and underscores.
- Split one requirement into roughly 3 to 10 executable steps by default.
- More than 10 steps is allowed only when strong atomicity requires it.
- Use serial subtasks, not parallel subtasks, when the work clearly conflicts or depends on strict sequencing.
- If a new task chain extends an existing chain, only the direct predecessor's `后序子任务` may be updated, and only with the smallest possible change.

## Skill definitions

### `mpc-master-start`

Manual invocation example:

```text
/mpc-master-start docs/example_plan.md
```

Responsibilities:

- Read a requirement or plan document.
- Analyze which parts can run in parallel and which must be serialized.
- Create new task blocks in `~/.mpc/{project-name}/mpc.md`.
- Generate:
  - task identifiers
  - predecessor and successor links
  - start prompts
  - completion prompts
- Create `~/.mpc/{project-name}/` and the minimal `mpc.md` skeleton when needed.
- Suggest `git worktree add` commands, but never execute them automatically.

Required output:

- total number of newly added tasks
- the first executable wave only
- suggested worktree commands only for that first executable wave

Additional constraints:

- A task is in the first executable wave when `前序子任务 = 无`, or when its predecessor tasks are already `已回归` in the existing controller files.
- Newly added tasks that are still blocked by predecessors must still be written into `~/.mpc/{project-name}/mpc.md`, but they should not be printed one by one in the command output.
- If there is no immediately executable task in the new batch, explicitly print `首轮可执行任务: 无` and briefly name the blocking predecessors without dumping the whole blocked chain.

Suggested command format:

```bash
git worktree add ../task_name -b task_name base_branch
```

### `mpc-master-progress`

Manual invocation examples:

```text
/mpc-master-progress
/mpc-master-progress phone_1_requirement_1
```

Responsibilities:

- Read only.
- Show a summary of all active tasks, or show one task in detail.
- Search `~/.mpc/{project-name}/mpc.md` first and `~/.mpc/{project-name}/mpc_archive.md` second.

Display rules:

- `未开始`: show `开始提示词`
- `进行中`: show elapsed time, progress summary, blockers, remaining estimate, next prompt
- `完成`: show completion time, total effort, completion summary, completion prompt
- `已回归`: show start time, completion time, regression time, total effort, regression summary, archive status

### `mpc-master-regress`

Manual invocation example:

```text
/mpc-master-regress phone_1_requirement_1
```

Responsibilities:

- Work on one task that is currently `完成`.
- Collect regression evidence from code changes, self-check results, remaining risks, and task goals.
- Present a recommendation first.
- Only move the task to `已回归` after explicit approval from the current caller.

When approved, update:

- `状态 = 已回归`
- `回归时间`
- `回归摘要`
- `最后更新时间`

If regression does not pass, keep the task in `完成`.

### `mpc-master-archive`

Manual invocation example:

```text
/mpc-master-archive phone_1_requirement_1
```

Responsibilities:

- Work on one task that is currently `已回归`.
- Move that task from `~/.mpc/{project-name}/mpc.md` to `~/.mpc/{project-name}/mpc_archive.md`.
- Create a minimal archive file if necessary.

When archiving, update:

- `归档状态 = 已归档`
- `归档时间`
- `最后更新时间`

Do not archive tasks in `未开始`, `进行中`, or `完成`.

### `mpc-slave`

Manual invocation example:

```text
/mpc-slave
```

Responsibilities:

- Run inside the current task worktree.
- Identify the task from the current directory name and git branch name.
- Refuse to continue if they do not match exactly.
- Use `~/.mpc/{project-name}/mpc.md` as the controller file.
- Only handle `未开始 -> 进行中 -> 完成`.

Behavior by state:

- `未开始`
  - check all predecessor tasks across both active and archive files
  - only start when all predecessors are `已回归`
  - initialize start time, remaining estimate, and next prompt
- `进行中`
  - update progress summary, blockers, remaining estimate, next prompt, and last updated time
  - prefer the newest relevant file under `./planning/`
  - if there is enough evidence, recommend completion first
  - only move to `完成` after explicit approval
- `完成`
  - show completion information only
  - never move directly to `已回归`
- `已回归`
  - show regression and archive information only
  - tell the caller to use `mpc-master-archive`

## Minimum working loop

The design is considered valid when all of the following work reliably:

1. `mpc-master-start` creates correctly structured subtasks.
2. `mpc-master-progress` shows a stable read-only overview.
3. `mpc-slave` identifies its own task from the current worktree and can move it to `完成` after approval.
4. `mpc-master-regress` can evaluate one completed task and safely move it to `已回归` after approval.
5. `mpc-master-archive` can safely move one regressed task into the archive file.
6. Parallel worktrees do not lose task state because of overwrite races.

## Explicitly forbidden behavior

- Do not make "view status" implicitly change status.
- Do not let `mpc-slave` bypass predecessor dependencies.
- Do not let `mpc-slave` jump from `进行中` straight to `已回归`.
- Do not let `mpc-master-progress` perform regression or archive actions.
- Do not archive tasks that have not already passed regression.
- Do not write controller files without holding `~/.mpc/{project-name}/.lock.md`.
- Do not decide that a predecessor task is missing by checking only `~/.mpc/{project-name}/mpc.md`.
- Do not rewrite unrelated task blocks while updating one task.
- Do not omit fixed fields and break downstream parsing.
