[English](README.md) · **简体中文**

# 面向 Codex 的 Multi-Phase Controller Skills

这个仓库是对 [`gsd-build/get-shit-done`](https://github.com/gsd-build/get-shit-done) 的一个补充 skill 包，定位为 GSD 项目的多 phase 管理辅助记录工具。它的核心职责不是替代 GSD 主工作流，而是把 GSD 已拆分的任务列表整理写入 `<repo-root>/.mpc/mpc.md`，并在多对话场景下持续同步任务进度。

控制核心围绕 `<repo-root>/.mpc/mpc.md`、`<repo-root>/.mpc/mpc_archive.md` 和 `<repo-root>/.mpc/.lock.md` 展开。控制目录固定放在目标项目的 Git 仓库根目录 `./.mpc/` 下；即使从子目录触发 skill，也要先解析到仓库根目录。`mpc-master-regress` 和 `mpc-master-archive` 仍然保留，但它们属于可选治理能力，不再是仓库的主叙事重点。

## 特点

- 面向 GSD 已拆分任务的整理导入与记录
- 用共享的 `mpc.md` 维护多会话协作中的进度真相
- 以单目录执行为主流程，适合并行推进多个任务
- 保留可选的回归确认与历史归档能力

## 包含的 skills

- `mpc-master-start`：整理导入 GSD 已拆分任务列表，写入 `<repo-root>/.mpc/mpc.md`，并罗列本次全部新增任务
- `mpc-master-progress`：只读查看共享任务记录板中的活动任务或归档任务状态
- `mpc-master-regress`：对已完成任务做可选的主控回归确认，并在确认后推进到 `已回归`
- `mpc-master-archive`：把一个已回归任务从 `<repo-root>/.mpc/mpc.md` 移动到 `<repo-root>/.mpc/mpc_archive.md`
- `mpc-slave`：在共享仓库目录中按显式任务名推进一个已记录任务的执行状态与进度同步

## 仓库结构

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

## 用 Codex 安装

这是一个多-skill 仓库。为了兼容 Codex 风格的安装方式，安装目标应当是 `skills/` 下的具体 skill 目录，而不是仓库根目录。

当前仓库地址：

```text
https://github.com/luobote55/multi-phase-controller
```

可直接使用的安装示例：

```text
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-start
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-progress
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-regress
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-master-archive
/skill install https://github.com/luobote55/multi-phase-controller/tree/main/skills/mpc-slave
```

推荐做法：

1. 如果要使用完整记录与治理能力，安装全部 5 个 skills。
2. 推送后尽量不要改 skill 文件夹名，因为安装 URL 会直接使用文件夹 basename 作为 skill 名。
3. 首次发布后至少验证一次 GitHub 安装 URL 是否可用。

## 使用说明

1. 在目标项目仓库里准备一份 GSD 计划文档或已经拆分好的任务列表。
2. 运行 `/mpc-master-start path/to/plan.md`，把任务整理写入 `<repo-root>/.mpc/mpc.md`。
3. 查看 `mpc-master-start` 的输出，它应按顺序给出：
   - `本次新增任务清单`
   - `最新任务树`
   - `首轮可执行任务与 /mpc-slave 命令`
4. 为首轮可执行任务分别打开独立对话，并运行 `/mpc-slave task_name`。
5. 每个任务都在自己的对话里持续推进；需要刷新进度或建议完成时，再次运行 `/mpc-slave task_name`。
6. 在主控会话中运行 `/mpc-master-progress`，查看总览或单任务详情。
7. 如果你需要做可选的主控回归确认，再运行 `/mpc-master-regress task_name`。
8. 如果任务已经回归通过且需要移出活动区，再运行 `/mpc-master-archive task_name`。

## 提示词约束

所有由 MPC skills 生成或展示给执行者的任务提示词，都应显式携带 GSD 能力前缀。

- `开始提示词` 首句必须包含：`利用gsd skills的能力，实现需求：<需求摘要>`
- `下一步提示词` 在自动生成或刷新后，也必须保留同样的前缀
- `完成提示词` 必须明确说明结论基于利用 GSD skills 推进后的产出

推荐模板：

```text
利用gsd skills的能力，实现需求：<需求摘要>。先确认边界与现状，再推进实现，并在过程中持续同步可用于更新 mpc.md 的进展信息。
```

```text
利用gsd skills的能力，实现需求：<需求摘要>。结合当前已完成内容、剩余工作与阻塞项继续推进下一步，并准备同步最新进度。
```

```text
基于利用gsd skills推进该任务后的产出，请确认是否已经满足需求：<需求摘要>；请给出完成依据、风险与是否可进入下一阶段的结论。
```

## 运行时文件

下面这些文件属于“使用这些 skills 的目标项目”的仓库根目录 `.mpc/`，除非当前仓库本身就是目标项目，否则不应出现在已发布的 skill 源码中：

- `<repo-root>/.mpc/mpc.md`
- `<repo-root>/.mpc/mpc_archive.md`
- `<repo-root>/.mpc/.lock.md`

## 其他建议

- 把这个仓库当作 GSD 的辅助记录工具，而不是替代品。
- 控制文件始终放在目标项目仓库根目录的 `.mpc/` 下。
- 只让定义好的写入型 skills 修改 `<repo-root>/.mpc/` 下的文件。
- 同一时刻只让一个对话推进一个任务。
- 当源文档已经是 GSD 已拆分任务列表时，优先保留其顺序、粒度和语义，只做规范化录入。
- 如果多个任务会改动同一文件、同一核心模块、依赖清单、生成产物、共享导出面，或存在明显顺序关系，就建模为串行任务而不是并行任务。
- 不要把运行时生成的 `.mpc/` 状态文件混入已发布的 skill 源码；本仓库默认通过 `.gitignore` 忽略它。

## 发布说明

- 所有可安装的 skill 都保持在 `skills/<skill-name>/` 下。
- 每个 skill 目录内保留 `SKILL.md` 和 `agents/openai.yaml`。
- 仓库级文档放在根目录或 `docs/` 里，不要往每个 skill 子目录里继续塞 README。
- 如果发布后重命名 skill 文件夹，旧的安装链接会失效。

## 文档索引

- English README: [README.md](README.md)
- 中文规范文档: [docs/MPC_SPEC.zh-CN.md](docs/MPC_SPEC.zh-CN.md)
- English spec: [docs/MPC_SPEC.en.md](docs/MPC_SPEC.en.md)
