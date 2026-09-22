# 可续跑执行流程

本文件定义 `write-branch-change-doc-v2` 的详细阶段。每次开始或恢复任务时完整读取。

## 目录和状态

运行目录必须位于仓库外，避免污染工作区。目录键由仓库绝对路径的稳定哈希和分支名组成。默认路径见 `SKILL.md`。

`state.json` 字段：

| 字段 | 含义 |
| --- | --- |
| `version` | 固定为 `2` |
| `repo_root` | 仓库绝对路径 |
| `branch` | 当前分支 |
| `base` | 配置基线 |
| `head` | 本次审查的 HEAD |
| `merge_base` | `base` 与 `HEAD` 的 merge base |
| `phase` | 当前阶段 |
| `cursor` | 当前阶段下一批或下一对象 |
| `completed` | 各阶段或对象完成数量 |
| `artifacts` | 关键产物绝对路径 |
| `delivery.mode` | `full_markdown` 或 `file_path`，默认 `file_path` |
| `updated_at` | 最后更新时间 |

写入顺序固定为：

```text
生成或覆盖批次文件
-> 校验批次文件
-> 更新覆盖清单
-> 原子更新 state.json
```

如果在批次文件和状态更新之间中断，重跑同一批次并覆盖相同路径。

阶段切换必须使用同一提交顺序：

```text
校验当前阶段全部完成
-> 更新 completed 和 artifacts
-> 原子更新 state.json
-> 读取新 phase 的第一个批次
```

只有在当前阶段完成条件全部满足后才允许修改 `phase`。不得提前把后续阶段标记为完成。

批次完成、覆盖清单更新、`state.json` 更新和阶段切换都不是停止点。除用户明确暂停或外部依赖使当前批次无法安全完成外，必须连续处理到 `DONE`。

## INIT

1. 执行 `git rev-parse --show-toplevel` 获取仓库根目录。
2. 读取 `~/.look-what-you-did/config.yaml`，查找 `repositories.<repo-root>.base`。
3. 配置不存在时不得默认使用 `main`，先询问用户；确认后按原格式写入配置再继续。
   配置目录权限使用 `0700`，配置文件权限使用 `0600`；只保存仓库路径和基线。
4. 更新基线对应远端引用。不能安全更新时标记“待确认”并停止依赖旧引用继续。
5. 获取并冻结：

```bash
git rev-parse HEAD
git rev-parse <base>
git merge-base <base> HEAD
git branch --show-current
```

6. 创建运行目录和 `state.json`。
7. 检查是否存在相同 `repo_root`、`base`、`head`、`merge_base` 的未完成运行；存在则恢复，不新建。

完成条件：`state.json` 中四个指纹字段完整且互相一致。

完成后将 `phase` 更新为 `INVENTORY`。

## INVENTORY

执行：

```bash
git diff --name-status <merge-base>..HEAD
git diff --stat <merge-base>..HEAD
```

将每个路径登记到 `inventory/changed-files.tsv`：

```text
id	path	change	kind	test_file	fact_file	status
```

`change` 只能为 `A`、`M`、`D`。`kind` 按实际内容分类为代码、测试、接口文档、数据库迁移、配置、纯文档、生成产物或其他。

规则：

- 同一路径只能出现一次。
- 变更数必须等于文件行数。
- 子模块只登记 gitlink，并在接口或文档事实中读取实际子模块差异。
- 建立清单后冻结；后续个别文件漏项只能修正清单并重新核对覆盖。

完成条件：全部路径为 `pending` 或 `done`，文件数和 Git 输出一致。

完成后将 `phase` 更新为 `FACTS_API`。

## FACTS_API

先识别接口变化来源：

- 仓库本地 API spec 的实际差异。
- 路由、handler、请求模型和响应模型的实际差异。
- git submodule 中本次新增或更新的 API spec。

为每个新增、修改或删除接口建立一行 `inventory/apis.tsv`：

```text
id	method	path	change	source	fact_file	status
```

每个接口只读取必要范围：

1. 当前定义。
2. 基线定义。
3. 代码中的实际请求和响应字段。

事实文件必须记录完整请求体和返回体。接口字段定义无变化时记录一份请求体和返回体；字段定义有新增、修改或删除时按当前请求体、当前返回体、原请求体、原返回体排列。

不得输出 `operationId`、鉴权等级、tags、description 等元信息。无法确认的信息标记“待确认”。

没有接口变化时创建 `facts/apis/none.md`，内容为“无”，并将阶段标记为 N/A 完成。

完成后将 `phase` 更新为 `FACTS_DATABASE`。

## FACTS_DATABASE

只依据迁移、DDL、ORM schema 或仓库内 schema 定义。

为每个新增、修改或删除的数据库对象建立事实文件，并登记 `inventory/database.tsv`：

```text
id	schema_object	change	source	fact_file	status
```

每个对象需要记录：

- 物理字段顺序。
- 字段类型、可空性、默认值。
- 新增、删除、不变状态。
- 约束和索引的名称、类型、之前状态和现在状态。
- 新增表和删除表的完整字段。

修改字段必须拆成删除行和新增行，不创建“字段修改”状态。

没有数据库变化时创建 `facts/database/none.md`，内容为“无”，并将阶段标记为 N/A 完成。

完成后将 `phase` 更新为 `FACTS_FLOWS`。

## FACTS_FLOWS

识别入口类型：HTTP、RPC、consumer、webhook、scheduler、启动装配、命令行或其他可执行入口。

为每个入口流程建立一行 `inventory/flows.tsv`：

```text
id	entry_type	entry_file	source	fact_file	status
```

逐流程处理，每个入口流程写入独立文件 `facts/flows/<id>.md`：

1. 从实际入口开始。
2. 按调用关系追踪到修改代码文件。
3. 同一文件出现在多个流程时重复列出。
4. 无法关联入口的修改文件写“入口待确认”。
5. 只包含参与代码或业务逻辑的文件；迁移、配置、纯文档和生成产物排除。

每个流程事实文件包含可直接装配的 Mermaid 片段，节点标签只使用文件名。

每完成一个流程，立即写事实文件，将对应清单项标记为 `done`，并更新 `cursor.next_flow_index` 和 `state.json`。处理完全部流程后，将 `inventory/changed-files.tsv` 中所有代码或业务逻辑变更文件与流程事实中的节点文件对照；未入图文件必须补入对应流程，无法关联入口时写入 `facts/flows/unlinked.md` 并标记“入口待确认”。

如果没有代码或业务逻辑变化，创建 `facts/flows/none.md` 并写“无”。

完成后将 `phase` 更新为 `FACTS_FILES`。

## FACTS_FILES

逐个处理 `inventory/changed-files.tsv`，不得一次读取全部文件。

每个路径建立 `facts/files/<id>.md`，内容结构固定：

```text
path
change
business_change
current_logic
previous_logic
```

处理规则：

- 只读取该文件相对 merge base 的 diff；必要时读取当前文件和基线版本的相关范围。
- 修改文件必须同时说明业务结果、状态变化、数据归属、权限边界或外部副作用。
- `current_logic` 和 `previous_logic` 只记录本次 diff 改变的业务行为；未变化的上游解析、下游调用、事件分支和副作用不得复述，必要时仅写“继续既有处理”。
- 测试文件的业务事实只说明本次新增或改变的 fixture、helper、测试覆盖对象和测试目的；不得列出未变化的既有测试，逐测试场景留到 `TESTS` 阶段。
- 新增文件说明业务职责和实现职责。
- 删除文件说明删除原因。
- 纯文档、配置、迁移和生成产物也必须在最终第四章再次列出。
- 每完成一个文件立即写事实文件并更新清单和状态。

完成全部文件后将 `phase` 更新为 `TESTS`。

## TESTS

### 建立测试索引

只处理相对 merge base 新增的测试用例。修改、重命名或未变化的测试不得进入清单。先以变更中新增或修改的测试文件为候选，再比较基线与 HEAD 的测试声明，只保留基线中不存在的新增测试。根据语言和框架选择稳定的声明提取方式；不得只依赖固定语言正则。

顶层测试和子测试都必须按完整测试路径比较。无法确认某个测试是新增、修改还是重命名时，标记“待确认”，不得当作新增测试写入。

`inventory/tests.tsv`：

```text
id	file	name	kind	start_line	end_line	status	fact_file
```

`kind` 分为顶层测试和子测试。子测试同样建立独立行。结束行由下一个声明前一行确定；无法可靠确定时标记“待确认”并按函数作用域人工定位。

索引数量必须与测试声明实际数量一致。建立索引后冻结，后续只修正无法可靠解析的项。

### 分批生成测试事实

1. 每批选择 `8` 至 `15` 个未完成测试。
2. 只读取本批测试的精确行范围。
3. 对每个测试提取业务输入、状态、权限、数据归属、外部调用、错误处理和最终断言。
4. 写入 `sections/06-tests/batch-<start>-<end>.md`。
5. 将每个测试标记为 `done`。
6. 更新 `cursor.next_test_index` 和 `state.json`。

表格行格式：

```markdown
| `TestName` | 实际业务场景和验证结果。 |
```

禁止：

- “覆盖对应函数的新增或回归行为”。
- “验证该方法正常工作”。
- 把场景写成前置条件、执行动作或测试步骤。
- 只写测试名称，不复述断言结果。

如果一个测试本身超过上下文预算，按逻辑断言拆成多个读取步骤，最后合并为一个表格行。

没有新增测试用例时，创建 `sections/06-tests/none.md`，内容为“无”。

全部新增测试完成且 `tests.tsv` 无 `pending` 项后，将 `phase` 更新为 `ASSEMBLE`。

## ASSEMBLE

装配前检查：

- `changed-files.tsv` 全部为 `done`。
- `apis.tsv`、`database.tsv` 全部为 `done` 或 N/A。
- `flows.tsv` 全部为 `done` 或 N/A，且全部代码或业务逻辑变更文件已入图或标记“入口待确认”。
- `tests.tsv` 全部为 `done` 或 N/A。
- 每个变更文件都有 `facts/files/` 产物。
- 每个接口、数据库对象和测试都有确定性事实文件。

按 [document-format.md](document-format.md) 装配，最终交付文件固定写入 `<repo_root>/branch-change-content.md`：

```text
01-intro.md
02-api.md
03-flows.md
04-files.md
05-database.md
06-tests.md
<repo_root>/branch-change-content.md
```

章节没有内容时跳过，后续章节顺延编号。

不得在装配阶段重新阅读原始大文件。装配只读取已确认的事实文件和批次文件。

`<repo_root>/branch-change-content.md` 写入并校验后，将 `phase` 更新为 `REVIEW_FORMAT`，并立即执行格式复审。

## REVIEW_FORMAT

完整读取 `<repo_root>/branch-change-content.md` 和 [document-format.md](document-format.md)，逐项检查：

- 标题、分支和基线头部。
- 章节顺序、标题层级和空分组。
- JSON 多行格式、嵌套字段和接口当前/原始顺序。
- 第三章每个入口是否明确标注协议是否变更。
- 图例与流程图使用完全相同且语义一致的 Mermaid `classDef` 色块，入口和节点标签正确。
- 字段表、约束索引表、测试表。
- 不存在 skill 禁止的元信息、注记或额外章节。

将结论、问题和修正写入 `reviews/format.md`。发现问题时修正对应章节文件并重新装配，然后重新执行本复审。

格式复审通过后，将 `phase` 更新为 `REVIEW_CONTENT`，并立即执行内容复审。

## REVIEW_CONTENT

完整读取 `<repo_root>/branch-change-content.md` 和相关事实文件，逐章核对：

- 接口字段类型和值。
- 数据库字段、默认值、约束和索引。
- 代码流程调用关系和文件分类。
- 每个修改文件的业务变化、当前逻辑和之前逻辑。
- 修改文件的内容仅聚焦本次 diff 的行为差异，没有复述未变化的上游解析、下游调用链或既有业务分支。
- 测试场景与实际断言一致。

不得只抽查重点章节。将结论、问题和修正写入 `reviews/content.md`。发现问题时回到对应事实阶段或章节批次修正，再重新装配并重新执行格式和内容复审。

内容复审通过后，将 `phase` 更新为 `REVIEW_COVERAGE`，并立即执行覆盖复审。

## REVIEW_COVERAGE

以以下清单为基准逐项比较：

- `inventory/changed-files.tsv` 与第四章文件标题。
- `inventory/apis.tsv` 与接口章节。
- `inventory/database.tsv` 与表结构章节。
- `inventory/flows.tsv` 和流程章节中全部代码或业务逻辑变更文件。
- `inventory/tests.tsv` 与测试章节中每个新增测试名。

检查：

- 每项至少出现一次。
- 不存在额外或占位文件。
- 没有重复行导致虚增覆盖。
- 所有状态均为 `done` 或 N/A。
- 文件数、接口数、数据库对象数和新增测试数与索引一致。

将结论、问题和修正写入 `reviews/coverage.md`。发现问题时修正对应产物并重新执行受影响的复审；结构变化必须重新执行全部三次复审。

三次复审全部通过后，将状态改为 `DONE`。

`DONE` 是唯一允许交付最终文档的状态。

## 上下文压缩恢复

收到压缩摘要后禁止先扫描仓库。固定执行：

```text
读取 state.json
-> 校验指纹
-> 读取当前 cursor
-> 读取当前批次的已有产物
-> 只处理未完成项
-> 覆盖批次文件
-> 更新 state.json
```

不得因为摘要缺少某个细节而重做已经标记完成的阶段。缺失细节只能从该阶段对应的事实文件补齐。
