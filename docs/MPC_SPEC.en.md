# MPC Skills Specification

## Purpose

This document defines the behavior contract for the following five skills:

- `mpc-master-start`
- `mpc-master-progress`
- `mpc-master-regress`
- `mpc-master-archive`
- `mpc-slave`

These skills are designed as a multi-phase management companion for GSD projects. Their primary purpose is to take GSD-split task lists, normalize them into `<repo-root>/.mpc/mpc.md`, and keep a shared progress record accurate across multiple conversations. They complement GSD rather than replace its main workflow.

`mpc-master-regress` and `mpc-master-archive` are still part of the toolkit, but they should be treated as optional governance features rather than the main story of the repository.

## Managed files

- Live controller file: `<repo-root>/.mpc/mpc.md`
- Archive file: `<repo-root>/.mpc/mpc_archive.md`
- Lock file: `<repo-root>/.mpc/.lock.md`

Rules:

- These files live under the current project's repo-root `.mpc/` directory, and all conversations for the same repository use the same files.
- The names `mpc.md`, `mpc_archive.md`, and `.lock.md` are fixed.
- `.lock.md` is a temporary write lock and is never created by read-only skills.
- If the files do not exist yet, `mpc-master-start` or the first archive action may create the minimal skeleton.
- Always resolve the current project's Git repository root first, then use that root's `.mpc/` directory.

## Core principles

- The main value of `mpc.md` is to record GSD-split tasks and preserve a shared source of truth for progress.
- The controller-side skills handle task intake, read-only inspection, optional regression, and optional archiving; they do not replace day-to-day implementation work.
- Subtasks are driven by explicit task identifiers rather than directory- or branch-based auto-detection.
- Single-directory execution is the primary mode: multiple subtasks progress through separate conversations in the same repository directory.
- The default rule is "one active conversation per task at a time". This spec does not add session-claiming machinery.
- When the source document already contains a GSD-split task list, preserve its order, granularity, and meaning whenever possible, and only normalize it.
- If there is a real dependency or conflict, declare it explicitly with `前序子任务` and `后序子任务`.
- If two subtasks touch the same file, core module, route or registry, dependency manifest, generated artifact, shared export surface, or otherwise require strict sequencing, they must be serialized rather than run in parallel.
- `mpc.md` must use a fixed field order. No field may be omitted. Empty values must be recorded as `待填写` or `无`.
- Read-only actions must stay read-only. State transitions must be explicit.
- All timestamps must use `YYYY-MM-DD HH:mm:ss +08:00`.
- Every write action except `mpc-master-progress` must hold `<repo-root>/.mpc/.lock.md`.

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
- `mpc-master-regress`: optionally move `完成 -> 已回归`
- `mpc-master-archive`: optionally archive tasks that are already `已回归`

Archiving is not a state. It is an action allowed only after regression passes.

## Fixed structure of `mpc.md`

Every task block inside `mpc.md` must preserve this exact field order:

`任务标识 -> 状态 -> 来源计划文件 -> 执行模式 -> 执行目录 -> 前序子任务 -> 后序子任务 -> 子任务目标 -> 开始时间 -> 完成时间 -> 回归时间 -> 归档时间 -> 最后更新时间 -> 累计耗时 -> 预计剩余时间 -> 当前进度概况 -> 阻塞事项 -> 下一步提示词 -> 开始提示词 -> 完成提示词 -> 完成摘要 -> 回归摘要 -> 归档状态`

Additional rules:

- `执行模式` is currently fixed to `single_dir`.
- `执行目录` is currently fixed to `.`.
- `当前进度概况` is at most 5 lines.
- `阻塞事项` is at most 5 lines. Write `无` if there is no blocker.
- `累计耗时` and `预计剩余时间` use `Xd Xh Xm` style text such as `4h 30m`.
- `完成摘要` must cover completed work, affected files or modules, self-check commands and results, remaining risk, and what the controller should focus on during regression.
- `回归摘要` must cover regression scope, evidence, result, and whether archiving is allowed.
- `归档状态` must be either `未归档` or `已归档`.

## Prompt template rules

All task prompts generated or surfaced by MPC skills must explicitly point the worker back to GSD skills.

- `开始提示词` must begin with `利用gsd skills的能力，实现需求：<subtask goal summary>`.
- Auto-generated or refreshed `下一步提示词` must preserve the same leading phrase rather than dropping or rewriting it.
- `完成提示词` must explicitly say the decision is based on outputs produced while using GSD skills.
- Old hard-coded phase-command examples are no longer allowed; use reusable GSD-oriented prompt templates instead.

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

## `mpc_archive.md`

- Each archived task keeps the same structure as in `mpc.md`.
- `任务总数` means the number of task blocks in the file.
- Archiving is a move, not a copy.
- After archiving:
  - the task is removed from `<repo-root>/.mpc/mpc.md`
  - the task is appended to `<repo-root>/.mpc/mpc_archive.md`
  - `状态` remains `已回归`
  - `归档状态` becomes `已归档`
  - `归档时间` is filled in
- Queries and dependency checks must search the union of `<repo-root>/.mpc/mpc.md` and `<repo-root>/.mpc/mpc_archive.md`, with the active file checked first.
- If the same task identifier exists in both files, that is a hard data error.

## Concurrent write rules

Even though real contention is usually low, all writes must still be serialized.

1. Every write action except `mpc-master-progress` must acquire `<repo-root>/.mpc/.lock.md`.
2. The lock file must record at least:
   - owner skill name
   - target task or task list
   - current repository directory or branch
   - lock timestamp
3. If the lock already exists, retry every 2 seconds for up to 60 seconds.
4. If the lock survives more than 10 minutes, treat it as stale and stop with an explicit error.
5. After taking the lock, reread the latest `<repo-root>/.mpc/mpc.md`. Read `<repo-root>/.mpc/mpc_archive.md` too when needed.
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

## Task intake and light splitting rules

`mpc-master-start` must follow these rules when organizing work:

- The input remains a plan or requirement file path.
- By default, treat that file as the source of a GSD-split task list.
- If the source already contains executable tasks, preserve their order, granularity, and meaning whenever possible, and only normalize them into MPC records.
- Only perform light splitting when an item is too coarse, mixes multiple separable work items, or cannot be tracked clearly for progress.
- Task names must be globally unique.
- Reserved names such as `mpc`, `mpc_archive`, or `lock` are forbidden.
- Task names may only use lowercase letters, digits, and underscores.
- If light splitting is needed, organize one requirement into roughly 3 to 10 executable steps by default.
- Use serial subtasks, not parallel subtasks, when the work clearly conflicts or depends on strict sequencing.
- Also default to serial subtasks whenever the work touches the same file, core module, route or registry, dependency manifest, generated artifact, or shared export surface.
- If a new task chain extends an existing chain, only the direct predecessor's `后序子任务` may be updated, and only with the smallest possible change.
- Do not add path-ownership checks or session-claiming logic in this version. Conflict control relies on task splitting, dependency links, and calling discipline.

## Task tree and new-task listing output

`mpc-master-start`, `mpc-master-progress`, `mpc-master-regress`, and `mpc-master-archive` must all print the latest task tree at the end of each command.

Task tree rules:

- Build the tree from the union of `<repo-root>/.mpc/mpc.md` and `<repo-root>/.mpc/mpc_archive.md`; ignore the archive file if it does not exist.
- When there are multiple values, split `前序子任务` and `后序子任务` by `、`. Tasks with `前序子任务 = 无` are roots.
- Child order must follow the parent's `后序子任务` order exactly. Root order should prefer the original order in `mpc.md`.
- Use `├─`, `└─`, `│  `, and three-space indentation consistently.
- `mpc-master-progress`, `mpc-master-regress`, and `mpc-master-archive` should append state labels by default. Archived tasks must display as `[已归档]`.
- If the same task exists in both active and archive files, or if the tree contains cycles, missing child tasks, or one child is directly claimed by multiple parents, stop and report a data error.

For `mpc-master-start`, the output order is fixed:

1. `本次新增任务清单`
2. `最新任务树`
3. `首轮可执行任务与 /mpc-slave 命令`

`本次新增任务清单` must include every newly added task in write order and show at least:

- `任务标识`
- a one-line `子任务目标` summary
- `前序子任务`
- `状态`

Do not list only first-wave tasks. Blocked tasks must appear in the list too.

## Skill definitions

### `mpc-master-start`

Manual invocation example:

```text
/mpc-master-start docs/example_plan.md
```

Responsibilities:

- Read a GSD plan document or an already split task list.
- Decide which items can be imported directly, which need only light normalization, and which require light splitting.
- Create new task blocks in `<repo-root>/.mpc/mpc.md`.
- Generate:
  - task identifiers
  - predecessor and successor links
  - subtask goals
  - start prompts
  - completion prompts
- Write `执行模式 = single_dir` and `执行目录 = .` for every new task.
- Create `<repo-root>/.mpc/` and the minimal `mpc.md` skeleton when needed.
- Suggest the first executable wave, but never execute anything automatically.

Additional requirements:

- Generated `开始提示词` must begin with `利用gsd skills的能力，实现需求：...`.
- The downstream `mpc-slave` flow must preserve the same prefix when it refreshes `下一步提示词`.
- Output must list every newly added task before showing the task tree.

### `mpc-master-progress`

Manual invocation examples:

```text
/mpc-master-progress
/mpc-master-progress phone_1_requirement_1
```

Responsibilities:

- Read only.
- Show a summary of all active tasks, or show one task in detail.
- Search `<repo-root>/.mpc/mpc.md` first and `<repo-root>/.mpc/mpc_archive.md` second.
- When displaying `开始提示词`, `下一步提示词`, or `完成提示词`, keep the original wording intact rather than stripping the GSD prefix.

### `mpc-master-regress`

Manual invocation example:

```text
/mpc-master-regress phone_1_requirement_1
```

Responsibilities:

- Work on one task that is currently `完成`.
- This is an optional controller-side regression confirmation capability, not the only core workflow.
- Build regression evidence primarily from:
  - `完成摘要`
  - self-check commands and results
  - the current overall repository state
  - `子任务目标`
- Interpret the review target as output produced while using GSD skills to drive the task.
- Present a recommendation first.
- Only move the task to `已回归` after explicit approval from the current caller.

### `mpc-master-archive`

Manual invocation example:

```text
/mpc-master-archive phone_1_requirement_1
```

Responsibilities:

- Work on one task that is currently `已回归`.
- This is an optional archival capability for historical cleanup.
- Move that task from `<repo-root>/.mpc/mpc.md` to `<repo-root>/.mpc/mpc_archive.md`.
- Create a minimal archive file if necessary.

### `mpc-slave`

Manual invocation example:

```text
/mpc-slave phone_1_requirement_1
```

Responsibilities:

- Require one explicit `任务标识` argument.
- Never auto-detect the task from the current directory, Git branch, or recent changes.
- Use `<repo-root>/.mpc/mpc.md` as the controller file.
- Advance a recorded task and synchronize its progress.

Behavior by state:

- `未开始`
  - check all predecessor tasks across both active and archive files
  - only start when all predecessors are `已回归`
  - initialize start time, remaining estimate, and set `下一步提示词 = 开始提示词`
- `进行中`
  - update progress summary, blockers, remaining estimate, next prompt, and last updated time
  - prefer the newest relevant file under `./planning/` in the shared repository directory
  - whenever `下一步提示词` is auto-generated or refreshed, preserve the `利用gsd skills的能力，实现需求：...` prefix
  - if there is enough evidence, recommend completion first
  - completion evidence must include completed work, affected files or modules, self-check commands and results, remaining risk, and recommended regression focus points
  - only move to `完成` after explicit approval
- `完成`
  - show completion information only
  - never move directly to `已回归`
- `已回归`
  - show regression and archive information only
  - tell the caller to use `mpc-master-archive`

## Minimum working loop

The design is considered valid when all of the following work reliably:

1. `mpc-master-start` reliably imports and normalizes GSD task lists into correct subtask records.
2. `mpc-master-progress` shows a stable read-only overview and task tree.
3. `mpc-slave` identifies its task from an explicit `任务标识` and can move it to `完成` after approval.
4. `mpc-master-regress` can optionally evaluate one completed task from completion summaries, self-checks, and the current repository state, then safely move it to `已回归` after approval.
5. `mpc-master-archive` can optionally move one regressed task into the archive file.
6. Parallel conversations do not lose task state because of overwrite races.

## Explicitly forbidden behavior

- Do not make "view status" implicitly change status.
- Do not let `mpc-slave` bypass predecessor dependencies.
- Do not let `mpc-slave` jump from `进行中` straight to `已回归`.
- Do not let `mpc-master-progress` perform regression or archive actions.
- Do not archive tasks that have not already passed regression.
- Do not write controller files without holding `<repo-root>/.mpc/.lock.md`.
- Do not decide that a predecessor task is missing by checking only `<repo-root>/.mpc/mpc.md`.
- Do not rewrite unrelated task blocks while updating one task.
- Do not omit fixed fields and break downstream parsing.
- Do not allow `mpc-slave` to run without an explicit task name or to guess one automatically.
