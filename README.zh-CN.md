# Maintainer Skills（维护技能）

[English](README.md)

用于维护上游项目长期 GitLab Fork 的 Opencode 技能集。包含两个配合使用的技能：**init-fork** 负责初始化 Fork 基础设施，**upstream-sync** 负责持续同步上游发布版本。

## 技能列表

### `/upstream-sync`

将 GitLab Fork 与上游仓库的最新标签发布版本同步。

- 从上游获取最新的语义化版本标签
- 通过隔离的 git worktree 合并到开发分支
- 智能解决合并冲突，以 `DOWNSTREAM_CHANGES.md` 作为 Fork 特定行为的真实性来源
- 检测上游是否已取代某个下游变更，并提议更新账本
- 打开一个 GitLab 合并请求并附带上游发布说明
- 在 `.upstream-version` 中追踪状态——无需外部数据库

支持在 CI 中无人值守运行（`opencode -p "/upstream-sync [URL]"`）或交互式使用。

### `/init-fork`

初始化一个用于长期下游维护的软 Fork。克隆 Fork 后运行一次即可。

- 配置 `upstream` git 远程仓库
- 创建 `.upstream-version`，记录发现的基线标签
- 创建 `DOWNSTREAM_CHANGES.md`——结构化的 Fork 独有修改账本
- 在 `AGENTS.md` 中追加 Fork 维护章节
- 检测已有的 Fork 独有提交，提供预填充账本选项

与 `/upstream-sync` 配合用于持续维护。

## 快速开始

```bash
# 1. 克隆 Fork
git clone https://gitlab.example.com/your-group/your-fork.git
cd your-fork

# 2. 初始化 Fork（一次性）
opencode -p "/init-fork https://github.com/upstream/project.git"

# 3. 与上游同步（定期运行或在 CI 中运行）
opencode -p "/upstream-sync"
```

## 文件结构

```
skills/
  upstream-sync/SKILL.md    # 同步技能定义与工作流
  upstream-sync/evals/       # 评估测试用例
  init-fork/SKILL.md         # 初始化技能定义与工作流
```

## 维护的产物文件

当这些技能在 Fork 上运行时，会创建和管理以下文件：

| 文件 | 用途 |
|------|------|
| `.upstream-version` | 追踪上游仓库 URL 和最后合并的标签 |
| `DOWNSTREAM_CHANGES.md` | 所有 Fork 独有修改的账本 |
| `AGENTS.md`（Fork 维护章节） | 声明仓库为 Fork 并记录账本格式 |

## 许可证

内部使用。
