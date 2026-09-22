---
name: write-branch-change-doc
description: 分批审查后端分支的接口、数据库、调用流程、文件变更和本分支新增测试，持续落盘并生成可续跑的固定结构 Markdown 变更说明；适用于大型分支或上下文受限环境。
---

# 后端分支变更说明

目标与 `write-branch-change-doc` 相同，但执行方式改为文件驱动的状态机。完整性来自持久文件和覆盖清单，不依赖模型把所有材料同时保存在上下文中。

开始或恢复任务前完整读取 [references/workflow.md](references/workflow.md)。装配文档和复审前完整读取 [references/document-format.md](references/document-format.md)。

## 执行约束

- 先创建持久运行目录和 `state.json`，再读取变更内容。
- 一个批次只读取一个边界明确的对象，例如一个接口、一个数据库对象、一个代码文件或 `8` 至 `15` 个测试。
- 每个批次先写事实或章节文件，再更新覆盖清单，最后原子更新 `state.json`；状态文件只是可续跑提交标记，不是本轮停止点。
- 最终文档必须在 `ASSEMBLE` 阶段写入仓库根目录的 `branch-change-content.md`，不得依赖压缩摘要临时重建。
- 已完成的阶段不得重复全量扫描；恢复时以 `state.json` 和已落盘文件为准。
- 上下文压缩摘要与持久文件冲突时，以持久文件为准。
- 默认连续执行 `INIT` 到 `DONE`，不得因批次完成、阶段切换或已更新 `state.json` 主动停止。
- 只有用户明确要求暂停，或外部依赖使当前批次无法安全完成时，才允许输出异常断点报告。
- 新增测试索引中的每个测试都必须有终态；不得只处理代表性测试。
- 不确定的信息标记“待确认”，不得编造。
- 文件事实和最终文档只描述本次 diff 触及的业务行为差异，不复述未变化的上游解析、下游调用链或既有业务分支。
- 不主动连接外部数据库。数据库结构以迁移、DDL、ORM schema 或仓库内定义为准。
- 不执行测试，除非用户另行明确要求；本技能只分析本分支新增测试。

## 运行目录

优先使用环境变量 `BRANCH_CHANGE_DOC_RUN_ROOT`。未设置时使用：

```text
${CODEX_HOME:-$HOME/.codex}/tmp/write-branch-change-doc/<repo-hash>/<branch-slug>/
```

目录结构：

```text
state.json
inventory/
  changed-files.tsv
  apis.tsv
  database.tsv
  flows.tsv
  tests.tsv
facts/
  apis/
  database/
  flows/
  files/
sections/
  01-intro.md
  02-api.md
  03-flows.md
  04-files.md
  05-database.md
  06-tests/
reviews/
  format.md
  content.md
  coverage.md
```

最终 Markdown 文件输出到仓库根目录下。

`state.json` 至少保存：

```json
{
  "version": 2,
  "repo_root": "/absolute/path",
  "branch": "branch-name",
  "base": "origin/main",
  "head": "sha",
  "merge_base": "sha",
  "phase": "INVENTORY",
  "cursor": {},
  "completed": {},
  "artifacts": {},
  "delivery": {
    "mode": "file_path"
  },
  "updated_at": "RFC3339"
}
```

## 状态机

| 阶段 | 产物 | 完成条件 |
| --- | --- | --- |
| `INIT` | `state.json` | 基线、HEAD、merge base 已冻结 |
| `INVENTORY` | `inventory/changed-files.tsv` | 全部新增、修改、删除路径已登记 |
| `FACTS_API` | `facts/apis/`、`inventory/apis.tsv` | 全部接口项为 `done` 或 N/A |
| `FACTS_DATABASE` | `facts/database/`、`inventory/database.tsv` | 全部数据库项为 `done` 或 N/A |
| `FACTS_FLOWS` | `facts/flows/`、`inventory/flows.tsv` | 每个入口流程已有文件，全部逻辑文件已入图或标记入口待确认 |
| `FACTS_FILES` | `facts/files/` | 每个变更文件已有文件事实 |
| `TESTS` | `inventory/tests.tsv`、`sections/06-tests/` | 全部新增测试项为 `done` 或 N/A |
| `ASSEMBLE` | `<repo_root>/branch-change-content.md` | 六个章节按规则装配完成 |
| `REVIEW_FORMAT` | `reviews/format.md` | 格式复审通过 |
| `REVIEW_CONTENT` | `reviews/content.md` | 内容复审通过 |
| `REVIEW_COVERAGE` | `reviews/coverage.md` | 覆盖复审通过 |
| `DONE` | 最终文档 | 三次复审全部通过 |

阶段可跳过的章节仍要建立明确的 N/A 状态或写“无”，不得用空文件隐式跳过。

执行连续性：

1. 未达到 `DONE` 且不存在外部阻塞时，必须继续处理下一步。
2. 批次完成和阶段推进只用于更新持久状态，不构成停止条件。
3. 复审发现问题时，修正产物后自动重新执行受影响复审，不等待用户推动。
4. 不得因为“担心后续上下文”而在安全的批次边界提前停止；只有当前批次确实无法安全完成时才进入异常断点。

阶段推进协议：

1. 只处理 `state.json` 的 `phase` 和 `cursor` 指向的当前批次。
2. 当前阶段全部完成条件满足且产物已校验后，才进入下一阶段。
3. 推进前先更新 `completed` 和 `artifacts`，再将 `phase` 改为下一阶段并清空不再适用的 `cursor`。
4. 不得为了预读后续阶段而加载整个仓库或全部 diff。
5. 阶段更新完成后立即读取并处理新 `phase` 的第一个批次。

## 恢复协议

每次启动或上下文压缩后，按以下顺序恢复：

1. 读取 `state.json`。
2. 校验 `repo_root`、`base`、`head`、`merge_base` 是否与当前仓库一致。
3. 不一致时创建新的带时间戳运行目录，不复用旧状态。
4. 一致时从 `phase` 和 `cursor` 继续。
5. 只读取当前批次及对应事实文件。
6. 不重新执行已标记完成且产物存在的阶段。
7. 如果状态为完成但产物缺失，将该阶段回退到未完成，不重新创建整个运行目录。
8. 完成当前阶段后按阶段推进协议更新 `state.json`，并立即处理新 `phase` 的第一个批次。恢复或压缩上下文后也不得因阶段切换主动停止。

## 异常断点报告

仅在用户明确要求暂停，或外部依赖使当前批次无法安全完成时，只输出：

```text
阶段：[phase]
已完成：[completed / cursor]
下一步：[只处理哪个未完成批次]
```

只有在上述硬阻塞条件下才输出。不得因批次数量、阶段数量、状态已更新、压缩恢复或希望先汇报进度而输出断点报告。异常断点报告不包含代码分析、文档正文或已完成阶段的复述。最终交付仍以 `DONE` 状态为准。

## 分批规则

- 测试默认每批 `8` 至 `15` 个。
- 单文件、单接口或单数据库对象过大时，继续按函数、字段组或调用分支拆分。
- 每批写入独立的确定性路径，重复处理时覆盖同一文件。
- 每批成功后更新覆盖清单，再更新 `state.json`。
- 同一批次不得同时追加到共享大文件，避免中断后重复或缺失。

## 交付

最终 Markdown 文件输出到仓库根目录下。`delivery.mode` 为 `full_markdown` 时，最后只读取已通过复审的 Markdown 文件作为回复；为 `file_path` 时，返回根目录下的 Markdown 文件和简短完成信息。

不得在最终阶段重新分析代码、重新生成测试场景或根据压缩摘要重写文档。
