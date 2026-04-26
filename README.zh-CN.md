[English](README.md) · **简体中文**

# 面向 Codex 的 Multi-Phase Controller Skills

这个仓库是对 [`gsd-build/get-shit-done`](https://github.com/gsd-build/get-shit-done) 的一个小规模补充 skill 包。它提供了一套轻量的多阶段控制器工作流，用来协调一个主控会话和多个在同一仓库目录中并行推进的子任务对话。

控制核心围绕 `~/.mpc/{git仓库名}/mpc.md`、`~/.mpc/{git仓库名}/mpc_archive.md` 和 `~/.mpc/{git仓库名}/.lock.md` 展开。目录名取 Git 仓库名字，而不是当前 checkout 路径。目标是在多对话并行场景下，让子任务拆分、进度推进、回归确认和归档都更稳定、更可追踪，而不是替代 GSD 本身。

## 特点

- 优先面向 Codex 组织目录
- 支持通过 GitHub 路径使用 `/skill install`
- 以单目录执行为主流程
- 仓库文档提供中英文双版

## 包含的 skills

- `mpc-master-start`：把一个需求或计划文档拆成 MPC 子任务，并给出首轮可执行任务
- `mpc-master-progress`：只读查看活动任务或归档任务状态
- `mpc-master-regress`：对已完成的单个 MPC 子任务做主控回归，并在确认后推进到 `已回归`
- `mpc-master-archive`：把一个已回归任务从 `~/.mpc/{git仓库名}/mpc.md` 移动到 `~/.mpc/{git仓库名}/mpc_archive.md`
- `mpc-slave`：在共享仓库目录中按显式任务名推进一个子任务的执行状态

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

1. 如果要跑完整闭环，安装全部 5 个 skills。
2. 推送后尽量不要改 skill 文件夹名，因为安装 URL 会直接使用文件夹 basename 作为 skill 名。
3. 首次发布后至少验证一次 GitHub 安装 URL 是否可用。

## 使用说明

1. 在目标项目仓库里准备一个需求或计划文档。
2. 运行 `/mpc-master-start path/to/plan.md`，把子任务写入 `~/.mpc/{git仓库名}/mpc.md`。
3. 为首轮可执行任务分别打开独立对话，并运行 `/mpc-slave task_name`。
4. 每个任务都在自己的对话里持续推进；需要刷新进度或建议完成时，再次运行 `/mpc-slave task_name`。
5. 在主控会话中运行 `/mpc-master-progress`，查看总览或单任务详情。
6. 当某个子任务进入 `完成` 后，运行 `/mpc-master-regress task_name`。
7. 回归通过后，运行 `/mpc-master-archive task_name`。

## 运行时文件

下面这些文件属于“使用这些 skills 的目标项目”，不属于本 skill 仓库本身：

- `~/.mpc/{git仓库名}/mpc.md`
- `~/.mpc/{git仓库名}/mpc_archive.md`
- `~/.mpc/{git仓库名}/.lock.md`

## 其他建议

- 控制文件始终放在 `~/.mpc/{git仓库名}/` 下，不要在仓库里创建本地 `.mpc/`。
- 只让定义好的写入型 skills 修改 `~/.mpc/{git仓库名}/` 下的文件。
- 同一时刻只让一个对话推进一个任务。
- 任务拆分要保守：只要涉及同文件、同核心模块、路由或注册表、依赖清单、生成产物、共享导出面，或存在明显顺序关系，就建模为串行任务而不是并行任务。
- 把这个仓库当作 GSD 的补充，不要把它当成 GSD 的替代品。
- 不要把运行时生成的 `~/.mpc/{git仓库名}/` 状态文件拷回或提交到这个 skill 仓库。

## 发布说明

- 所有可安装的 skill 都保持在 `skills/<skill-name>/` 下。
- 每个 skill 目录内保留 `SKILL.md` 和 `agents/openai.yaml`。
- 仓库级文档放在根目录或 `docs/` 里，不要往每个 skill 子目录里继续塞 README。
- 如果发布后重命名 skill 文件夹，旧的安装链接会失效。

## 文档索引

- English README: [README.md](README.md)
- 中文规范文档: [docs/MPC_SPEC.zh-CN.md](docs/MPC_SPEC.zh-CN.md)
- English spec: [docs/MPC_SPEC.en.md](docs/MPC_SPEC.en.md)
