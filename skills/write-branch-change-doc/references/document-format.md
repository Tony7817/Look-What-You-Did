# 文档输出规范

装配和复审前完整读取本文件。最终输出只包含本文件规定的 Markdown 文档。

## 通用规则

- 适用任意后端语言、框架和仓库；不得假设特定模块、表名或接口。
- 不确定的信息标记“待确认”，不得编造。
- 语言简洁，优先使用短句、列表和表格。
- 字段、枚举、参数、映射、状态流、依赖和测试场景优先使用表格。
- JSON 代码块必须使用多行格式，每个字段单独占一行。
- 修改内容章节必须逐文件使用独立标题，不得合并多个文件。
- “业务变化”“目前的逻辑”“之前的逻辑”只描述本次 diff 触及的行为差异；未变化的上游解析、下游调用链、事件分支和副作用不得复述，必要时仅用“继续既有处理”概括。
- 测试文件只描述本次新增或改变的 fixture、helper 和测试，不列出未变化的既有测试。
- 字段表一行一个字段。
- 不输出“注：”、测试执行说明、基线失败说明或迁移副本说明。
- 不输出“已知事项与后续”“后续计划”“背景介绍”或“总结”。

## 固定结构

```markdown
# [功能名]（[一句话限定]）分支变更说明

> 分支：`[branch]`
> 基线：`[base]`（merge base `[sha]`）

## 一、简介

## 二、接口文档

## 三、修改代码涉及入口

## 四、修改内容

## 五、表结构的修改

## 六、测试覆盖情况
```

与变更无关的章节跳过，后续章节顺延编号。

## 一、简介

只列事实，每条一行：

- 新增接口 / 能力 / 表 / 配置 / 任务。
- 修改接口 / 能力 / 表 / 配置 / 任务。
- 删除接口 / 能力 / 表 / 配置 / 任务。

无对应变更的类型不写。

## 二、接口文档

仅记录仓库本地接口文档或代码中实际存在的接口。不得输出 `operationId`、鉴权等级、tags、description 等元信息。

接口文档文件：

- `[api spec path]`

### 2.1 接口定义

按接口逐项列出全部新增接口和全部修改接口的当前最新完整定义。

字段变更标记：

- 🟩 新增：当前请求体或当前返回体中的新增字段。
- 🟧 修改：当前请求体或当前返回体中类型、结构或约束发生变化的字段。
- 🟥 删除：原始请求体或原始返回体中被删除的字段。

`[METHOD] [PATH]`

当前请求体：

```json
{
  "[field]": "[type / value]",
  "[new field]": "🟩 新增 / [type / value]",
  "[changed field]": "🟧 修改 / [type / value]"
}
```

当前返回体：

```json
{
  "[field]": "[type / value]",
  "[new field]": "🟩 新增 / [type / value]",
  "[changed field]": "🟧 修改 / [type / value]"
}
```

修改接口紧接着列出基线定义：

原始请求体：

```json
{
  "[field]": "[type / value]",
  "[deleted field]": "🟥 删除 / [type / value]"
}
```

原始返回体：

```json
{
  "[field]": "[type / value]",
  "[deleted field]": "🟥 删除 / [type / value]"
}
```

新增或修改接口组为空时写“无”。删除接口只列方法和路径：

`~~[METHOD] [PATH]~~`

规则：

- 修改接口的字段定义完全无变化时，接口下方写“接口无变动。”，仅列一次请求体和返回体，不输出当前和原始两套定义。
- 修改接口存在字段新增、修改或删除时，顺序固定为当前请求体、当前返回体、原请求体、原返回体。
- JSON 只展示真实请求体和返回体。
- 嵌套字段完整展开。
- 当前请求体和当前返回体中的新增字段必须在类型值前标记 `🟩 新增 /`，修改字段必须标记 `🟧 修改 /`。
- 原始请求体和原始返回体中被删除的字段必须在类型值前标记 `🟥 删除 /`；未删除字段不加标记。
- 仅业务行为变化但字段定义、类型、结构和约束未变化时，不得添加字段变更标记。
- 字段标记放在 JSON 字符串值内，不得使用 JSON 不支持的注释语法，确保代码块仍是合法 JSON。

## 三、修改代码涉及入口

以下列出全部修改代码涉及的入口。

图例：

```mermaid
flowchart LR
    added["新增"]:::added
    modified["修改"]:::modified
    unchanged["无变动"]:::unchanged
    deleted["删除"]:::deleted

    classDef added fill:#2b8a3e,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef modified fill:#f08c00,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef unchanged fill:#1864ab,stroke:#000000,stroke-width:2px,color:#ffffff
    classDef deleted fill:#c92a2a,stroke:#000000,stroke-width:3px,color:#ffffff
```

入口流程图示例：

```mermaid
flowchart LR
    entry["entry.ext"] --> service["service.ext"]

    classDef added fill:#2b8a3e,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef modified fill:#f08c00,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef unchanged fill:#1864ab,stroke:#000000,stroke-width:2px,color:#ffffff
    classDef deleted fill:#c92a2a,stroke:#000000,stroke-width:3px,color:#ffffff
    class service added
    class entry unchanged
```

规则：

- 只覆盖参与代码或业务逻辑的新增、修改、删除文件。
- 纯文档、生成产物、迁移文件和配置文件排除。
- 列出全部修改代码涉及的入口，从实际入口文件开始展示对应调用流程。
- 每个入口标题下必须明确写出 `协议是否变更：是` 或 `协议是否变更：否`。
- 同一文件参与多个流程时重复体现。
- 无法关联入口的代码或业务逻辑文件不得省略，单独标记“入口待确认”。
- “入口待确认”按未关联入口分组；同一文件参与多个流程时分别放入每个流程。
- 节点标签只写文件名，不写路径、方法、接口地址或说明。
- 图例必须使用 Mermaid 色块示例，不直接以颜色名称或 emoji 代替；图例和流程图的 `classDef` 名称、颜色、边框及文字样式必须完全一致。
- 绘图后核对每个代码或逻辑变更文件均已入图。

## 四、修改内容

逐项列出全部变更文件，按新增、修改、删除分组。代码、业务逻辑、配置、迁移、测试、纯文档和生成产物均不得省略。

文件即使已出现在接口、流程、表结构或测试章节，也必须在本章再次列出。

任一分组为空时，该分组标题下写“无”。

### 4.1 新增文件

#### `[file path]`

- 业务职责：[业务能力、业务对象或用户动作]。
- 实现职责：[技术职责以及如何支撑业务]。
- 核心用例：主流程用 `[A] → [B] → [C]`。
- 按实际维度列出入参校验、状态校验、幂等、外部调用、失败处理、事务边界、提交后副作用等。

### 4.2 修改的文件

#### `[file path]`

业务变化：

- [业务场景和业务规则现在如何工作]。
- [之前的业务场景和业务规则如何工作]。

目前的逻辑：

- [本次修改后的行为差异及其直接业务结果、状态变化或副作用]。

之前的逻辑：

- [同一行为差异在基线中的逻辑]。

规则：

- 每个修改文件必须同时写清业务描述、之前逻辑和目前逻辑。
- 只写本次 diff 改变的行为，不复述未变化的上游解析、下游调用链、事件分支或副作用；需要说明继续既有处理时，仅写“继续既有处理”。
- 只描述新增依赖、注册接口或声明实现等技术事实不算业务描述。
- 无法确认业务语义时标记“待确认”。

### 4.3 删除的文件

#### `[file path]`

- [为什么删除]。

## 五、表结构的修改

图例：

```mermaid
flowchart LR
    added["新增"]:::added
    modified["修改"]:::modified
    unchanged["无变动"]:::unchanged
    deleted["删除"]:::deleted

    classDef added fill:#2b8a3e,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef modified fill:#f08c00,stroke:#000000,stroke-width:3px,color:#ffffff
    classDef unchanged fill:#1864ab,stroke:#000000,stroke-width:2px,color:#ffffff
    classDef deleted fill:#c92a2a,stroke:#000000,stroke-width:3px,color:#ffffff
```

迁移 / Schema 来源：

- `[migration or schema file]`

### 5.1 修改表：`[schema.table]`

全字段按物理顺序列出：

| 字段 | type\|null\|default | 变更 |
| --- | --- | --- |
| `[field]` | `[type] / [null] / [default]` | 无变动 |
| `[field]` | `[old type] / [old null] / [old default]` | 删除 |
| `[field]` | `[new type] / [new null] / [new default]` | 新增 |

约束与索引变更：

| 名称 | 类型 | 之前 | 现在 |
| --- | --- | --- | --- |
| `[name]` | `[constraint / index type]` | `[before]` | `[after]` |

规则：

- 变更列只写 `新增`、`无变动` 或 `删除`，不写说明；表格使用图例对应文字，不依赖 emoji 或颜色渲染。
- 不存在“修改字段”状态；修改必须拆成删除行和新增行。
- 字段、约束、索引必须列全。

### 5.2 新增表：`[schema.table]`

全字段：

| 字段 | type\|null\|default | 变更 |
| --- | --- | --- |
| `[field]` | `[type] / [null] / [default]` | 新增 |

索引：

| 名称 | 类型 | 内容 |
| --- | --- | --- |
| `[name]` | `[index type]` | `[columns and conditions]` |

### 5.3 删除表：`[schema.table]`

- `[why removed]`

## 六、测试覆盖情况

只逐项列出本分支相对基线新增的测试用例。修改、重命名或未变化的测试文件及测试用例不得列出。测试文件本身有变更但没有新增测试用例时，不列该文件。没有新增测试用例时只写“无”。

测试场景必须说明业务输入、状态、权限、数据隔离、外部调用、错误处理、事务结果或具体技术断言。不得使用“覆盖对应方法的行为”或“验证方法正常工作”等泛化表述。

### 6.1 [测试类型]：`[test file path]`

| 测试 | 覆盖场景 |
| --- | --- |
| `[test name]` | `[scenario]` |

## 最终回复

- `delivery.mode=full_markdown`：最终回复只包含已复审的完整 Markdown 文档。
- `delivery.mode=file_path`：最终回复说明 Markdown 文件已输出到仓库根目录下，并附简短完成信息。

不得附加技能说明、测试执行说明、进度说明或后续计划。
