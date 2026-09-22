# 安装说明

本仓库提供 `write-branch-change-doc`，用于生成后端分支变更说明。安装整个 `skills/write-branch-change-doc/` 目录，包括 `SKILL.md`、`references/` 和 `agents/`。

## 方式一：克隆后让 agent 安装

克隆仓库：

```bash
git clone https://github.com/Tony7817/Look-What-You-Did.git
```

把仓库的实际绝对路径告诉 Codex agent，例如：

```text
请阅读 /absolute/path/to/Look-What-You-Did/INSTALL.md，
将这个仓库中的 write-branch-change-doc skill 安装为 Codex 全局技能。
```

Agent 安装步骤：

1. 从给定仓库的 `skills/write-branch-change-doc/` 读取完整技能。
2. 全局安装目标为 `$CODEX_HOME/skills/write-branch-change-doc/`；未设置 `CODEX_HOME` 时使用 `~/.codex/skills/write-branch-change-doc/`。
3. 检查是否已有同名技能，包括项目 `.agents/skills/` 中的副本或链接。存在时先比较内容并确认实际加载位置；有本地修改时先备份，不盲目覆盖或创建重复副本。
4. 将完整技能目录复制到安装目标，不需要复制仓库根目录的文件。
5. 验证目标中的四个文件与源文件一致，且 `SKILL.md` 引用的文件存在。
6. 报告实际安装位置；技能将在下一轮对话中可用。

如果只希望在某个项目使用，可以改为请求：

```text
请将 /absolute/path/to/Look-What-You-Did 中的 write-branch-change-doc
安装到 /absolute/path/to/project/.agents/skills/，仅供该项目使用。
```

## 方式二：直接从 GitHub 安装

无需先克隆，向 Codex agent 发送：

```text
请使用 skill-installer，从 GitHub 仓库 Tony7817/Look-What-You-Did
的 main 分支安装 skills/write-branch-change-doc。
如果已安装同名技能，先比较现有版本并保留本地修改。
```

## 安装后使用

在需要审查的代码仓库中，发送：

```text
使用 $write-branch-change-doc 生成当前分支的变更说明。
```

首次使用时，如果该代码仓库尚未配置对比基线，agent 会询问基线分支或 commit。最终 Markdown 文件输出到该代码仓库根目录下。

## 更新

通过克隆方式安装时，先在本仓库执行 `git pull --ff-only`，再让 agent 按上述步骤比较并更新已安装副本。复制安装的技能不会随仓库拉取自动更新。

通过 GitHub 安装时，让 agent 获取 `main` 最新版本并比较更新。`skill-installer` 在目标目录已存在时会停止安装，因此需要先处理已有版本，不能将重复安装当成更新。
