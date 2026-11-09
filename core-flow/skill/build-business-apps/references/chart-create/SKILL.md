---
name: chart-create
description: 图表创建 Skill - 按图表侧最新 ai-app-chart-create 规则，把《业务系统建设方案》第 8 节仪表盘设计与真实数据源字段转成 13 类组件 JSON，再组装成图表创建页可消费的 AI 建图配置（ai-create-chart-config），完成「AI 创建业务系统」阶段 4 中的**仪表盘**创建。当用户说"AI 创建图表"、"生成建图配置"、"创建仪表盘应用"、"把业务系统建设方案仪表盘转成 JSON"、"调用 chart-create"时调用。
---

# Skill：图表创建（chart-create）

> **定位**：本 skill 是「AI 创建业务系统」**阶段 4 应用包生成**里「应用创建 skill」系列之一，承担**仪表盘**（图表）的创建。它与 [表单规则](../form-approval.md)、[审批规则](../form-approval.md) **并列而非包含**：图表配置没有 `formConfig`、没有审批节点，只有 `dataSources` + `components`，且用**另一个配置消息 key**。
>
> **权威组件规则**：图表侧提供的最新 skill 已原样收录在
> [references/ai-app-chart-create.md](references/ai-app-chart-create.md)。本文件负责「怎么在本系统接入」，
> 组件白名单、维值组合、聚合 / 排序 / TopN 等生成规则以该文件为准；两者冲突时以官方副本为准。
>
> **职责边界**：本 skill 只生成配置与接入片段，不改创建页代码、不产 Demo 工程、不替用户选择数据源。
> 字段清单不自行采——auto-flow 内置链路经后端字段能力获取；生成组件 JSON 这一步不调任何工具。
>
> **路径约定**：`references/...` 为相对本 skill 目录（`skills/chart-create/`）的路径。`{...}` 为上下文占位符，须按实际情况替换，**不是文件名的字面组成**。

## 触发词

`/chart-create`

## 输入约定

组件生成使用官方 skill 的输入形状：**用户需求 + 数据源 `source-schema` 数组**（`source` + `fields`，
字段含 `fieldId` / `innerIndex` / `label` / `componentType` / `valueType` / `metricAggregations`）。
在本系统交接链路里，这两部分来源如下：

| 输入 | 必填 | 说明 |
| --- | --- | --- |
| 图表需求 | ✅ | 业务系统建设方案第 8 节仪表盘设计（受众、决策问题、区块×组件×数据源×维度/指标）；至少含**仪表盘名称 + 组件意图** |
| 仪表盘名称 | ✅ | 取自业务系统建设方案第 3 节首页看板名 / 第 8 节；**无法确定时先问用户，禁止编造** |
| 数据源清单 | ✅（完整配置时） | 每个数据源必须给**真实** `sourceId`（表单 / 审批 ID）+ `sourceType`（`"1"` 审批 / `"4"` 表单）+ `sourceName`；**由已发布应用提供，AI 不得编造** |
| 字段清单 | ✅（生成组件时） | 来自 auto-flow 后端 `GET /auto/flow/data/chart/sources/{sourceType}/{sourceId}/fields`（后端调办公端 `CHART_APP` 的 `source-schema`）；**不直连平台 `source/config`，也不编造** |
| 创建页地址 | ⬜ | 由平台统一入口提供；本系统不手拼创建页完整地址 |
| 是否立即保存 | ⬜ | 交接链路统一 `publishImmediately: true`，由创建页自动发布后回传 |
| 需求名称 | ⬜ | `--output={需求名称}`，用于落盘文件夹；缺省时从对话提取 |

> **输入充分性检查**：数据源标识缺失或字段清单为空时，**不得编造**。两条合法路径：
> ① 反问用户补齐数据源 / 字段；② 按「空配置」产出——`dataSources: []`、`components: []`，并如实说明
> 「创建页会按普通新建处理」。**空配置是如实结果，不是遗漏。**
>
> **auto-flow 内置链路的口径**：业务系统建设方案阶段表单尚未发布，拿不到真实 `form_id` 与字段 ID，所以 AI 建包阶段
> 只生成 `{name, summary, dataSources: [], components: []}` 骨架；应用包内入口创建图表时，数据源来自
> **已 claim 的表单 / 审批**，SPA 调后端字段接口拿到归一化字段（`fieldId + innerIndex + type`）后，
> 按本 skill 规则生成组件并交平台校验。浏览器与 skill 都**不得直连** `/manage/apps/chart/data/source/config`。

## 执行步骤

### 第 1 步 · 读输入，抽出仪表盘要素

1. 读取业务系统建设方案第 8 节（或用户口述需求）。
2. 抽出：**仪表盘名称**、**受众 / 决策问题**（→ `summary`）、**组件意图清单**（区块 / 组件 / 数据源 /
   维度与指标）、**全局设置**（是否首页、可见范围）。
3. 需求里没有任何图表 / 看板 / 仪表盘意图时，**停止生成并转 [表单规则](../form-approval.md) 或
   [审批规则](../form-approval.md)**——图表 skill 不产表单。

### 第 2 步 · 核对数据源（真实标识，不得编造）

1. 每个数据源必须有：`sourceId`（**字符串**）、`sourceType`（**字符串**，`"1"` 审批 / `"4"` 自定义表单）、
   `sourceName`。
2. `sourceId` 必须是平台已存在的表单 / 审批 ID（`form_id`）；**名称不能代替 ID**。
3. `sourceType` 传数字会被服务端判为不匹配；必须写成字符串。
4. 至少 1 个数据源；任一声明无效 / 无权限 / 无字段，**整份配置被拒绝**，创建页按普通新建处理。
5. **拿不到真实 ID 时不写占位值**，改走「空配置 + 如实说明」。

### 第 3 步 · 按官方规则生成组件

> 先加载 [references/ai-app-chart-create.md](references/ai-app-chart-create.md)；细节见
> [references/component-rules.md](references/component-rules.md)。

1. `componentType` 必须在 **13 类白名单**内：`baseDetail`、`zhibiao`、`zhibiaoCard`、`baseBar`、
   `stackBar`、`stripBar`、`stackStripBar`、`baseLine`、`basePie`、`baseRadar`、`wordCloud`、`baseMap`、
   `rankList`；其余组件会被创建页**直接剔除**。
2. 维值组合必须合法：明细表必须有非空 `detailColumns`；指标卡 0 维 1 值；`zhibiao` 1 维 N 值；
   柱 / 条 / 线支持 1 维 N 值或 2 维 1 值；雷达 / 地图 / 排行榜 1 维 N 值；饼图 / 词云只允许 1 维 1 值；
   `dimensions` 只能绑定非数值字段；地图维度必须是 `areamultiselect` 地址字段。
3. 字段选择：只用输入中的 `fieldId + innerIndex`；姓名、手机号、身份证、编号、编码、描述、备注等
   高基数字段默认不做维度；联系人 / 所属人字段默认不做维度；字段名按语义同义词匹配。
4. 聚合：数值字段只允许 `总数` / `平均值` / `最大值` / `最小值` / `次数`；文本 / 日期字段只允许 `次数`；
   优先使用字段自带的 `metricAggregations`；不输出「去重计次」。
5. 排序：只在用户明确要求时输出 `sortType`（`asc` / `desc`）；`wordCloud`、`baseMap`、`zhibiaoCard`
   不支持排序；排序位置按组件遵守官方规则（明细列 / rankList 指标 / 2 维只写维度 / 1 维优先指标，
   整个组件最多 1 个有效排序）。
6. TopN：只在用户明确要求时输出；`basePie` / `baseRadar` 取 1~100，其他组件 1~1000；不要自造 N。
7. 标题：业务名称、非空、同一结果内不重复；不要照抄「柱状图」等默认名。
8. 同效果去重：同一分析效果不要输出完全相同的配置，可在约束内换组件类型。
9. 组件数 **≤ 8**，第 9 个及以后会被忽略。
10. 字段存在但做不了聚合时输出 `baseDetail`；找不到可直接绑定的维度 / 指标时输出空 `components` 并在
    `warnings` 说明；数据源没有字段时同样输出空数组 + `warnings`。

### 第 4 步 · 组装配置并做发送前自检

1. 组件生成步骤的输出形状：顶层只允许 `name` / `summary` / `components` / `warnings`；
   每个组件只允许 `componentType` / `title` / `sourceId` / `sourceType` / `dimensions` / `metrics` /
   `detailColumns` / `topNum`；`sourceId` / `sourceType` / `fieldId` 用字符串，`innerIndex` 用整数。
2. 接入创建页时，把组件合并进图表接入配置：`{name, summary, publishImmediately, dataSources, components}`
   ——`warnings` 不写入创建页配置，只用于对话回显；`dataSources` 必须与组件引用一致。
3. 按 [references/validation-checklist.md](references/validation-checklist.md) 逐项自检；**创建页剔除组件是
   静默的**，不能依赖创建页反馈。
4. **自检不通过不落盘**——缺什么回显什么，补齐后再走第 5 步。

### 第 5 步 · 落盘与回显

1. 落盘 **1 个 JSON 配置文件**：`context/workspace/{需求名称}-yyyyMMdd/{仪表盘名称}-ai-create-chart-config-{日期}.json`；
   一个业务系统建设方案含多个仪表盘时每个一份。
2. **在对话中回显四段**（缺一不可）：
   - ① 完整 JSON 配置（代码块）；
   - ② 握手 + 发送配置代码片段（键 `ai-create-chart-config`，由 SPA 在 `ai:ready` 后 postMessage 发出）；
   - ③ **配置还原表**：数据源 / 组件逐项列出 类型 / 标题 / 数据源 / 维度 / 指标；
   - ④ 发送前自检结果表（逐项 ✅ / ⚠️，含空配置与 `warnings` 的如实说明）。
3. 本系统实际入口由平台统一入口 + SPA 内嵌 iframe 完成，**不手拼创建页完整域名**，也不在聊天里让用户
   自己贴配置到浏览器。

## 输出契约

1. **组件生成输出**：严格 JSON `{name, summary, components, warnings}`；除生成组件 JSON 外不输出其他文字。
2. **发给创建页的配置**：顶层 `name`（必填）+ `dataSources`（必填）+ `components`（必填，可空）
   + 可选 `summary` / `publishImmediately`；`warnings` 不落入配置。
3. **配置消息 key**：固定 `ai-create-chart-config`，**不得改名，不得复用 `ai-create-config`**（它是 postMessage 消息里的 `key`，不是浏览器存储键）。
4. **空值处理**：数据源 / 字段 ID 未确认时**不编造**，输出空数组并如实说明。
5. **不做的事**：不产出 Demo 工程、不修改创建页代码、不直连平台取字段、不伪造字段清单、不替用户选择数据源。

## 硬约束

**必须**

- ✅ **组件规则以 [references/ai-app-chart-create.md](references/ai-app-chart-create.md) 为准**；
  13 类白名单、维值组合、聚合、排序、TopN、标题与去重要求逐条遵守。
- ✅ **配置消息 key 固定 `ai-create-chart-config`**：收到 `{ type: 'ai:ready' }` 后 `window.parent.postMessage({ key: 'ai-create-chart-config', data: JSON.stringify(config) }, event.origin)`，`data` 必须是字符串（平台只解析字符串）。
- ✅ **先握手再发配置**：只在收到 `ai:ready` 后发送，`targetOrigin` 用握手消息的 `event.origin`，**绝不用 `*`**；跨源同站可用，跨站不能内嵌（平台会话 Cookie 为 `SameSite=Lax`，会落到登录页）。
- ✅ **`sourceId` / `sourceType` 必须是字符串**；`sourceType` 只用 `"1"`（审批）/ `"4"`（表单）。
- ✅ **每个组件必须命中 `dataSources`**（`sourceType + sourceId` 组合一致）。
- ✅ **`fieldId` + `innerIndex` 组合定位字段**；只用后端字段能力返回的真实字段。
- ✅ **组件数 ≤ 8**；数值聚合只用 5 种；`sortType` 只用 `asc` / `desc`；`topNum` 按范围取值。
- ✅ **名称非空**；组件标题不空、不重复；空数据源 / 空字段如实输出空 `components` + `warnings`。

**禁止**

- ❌ 复用表单键 `ai-create-config`、自造配置消息 key。
- ❌ 编造数据源 ID、字段 ID、字段类型、地址类字段等真实数据。
- ❌ 把 `sourceType` 写成数字、把 `fieldId` / `innerIndex` 只写一个。
- ❌ 使用 13 类之外的组件类型，或超过 8 个组件还假装全部生效。
- ❌ 给没有仪表盘意图的需求硬产图表配置（应转 `form-create` / `approval-create`）。
- ❌ 生成组件 JSON 时调用工具、取字段、发配置消息或跳转（那是接入链路的职责）。

## 参考文件（按需加载，渐进披露）

| 文件 | 内容 | 何时加载 |
| --- | --- | --- |
| `references/ai-app-chart-create.md` | **图表侧最新 skill 全文**：输入形状、输出形状、13 类组件、维值 / 字段 / 聚合 / 排序 / TopN / 去重 / 标题 / 边界 | 第 3 步生成组件前必读 |
| `references/config-schema.md` | 顶层字段、`dataSources` / `components` / 字段绑定结构、配置消息 key、postMessage 握手前提、创建页 URL 参数、读取失败时创建页表现 | 第 4 步组装配置时 |
| `references/component-rules.md` | 13 类组件白名单、维度与数值组合规则、聚合方式、排序、取前 N、语义建议与静默剔除清单 | 第 3 步映射组件时 |
| `references/validation-checklist.md` | 发送前校验函数与检查清单、联调最小用例、创建页读取失败表现 | 第 4 步自检与联调时 |

> **渐进披露原则**：本文件只放**每次都要用的**触发条件、输入契约、流程、输出契约与硬约束；
> 组件细节放 `references/ai-app-chart-create.md` 与 `component-rules.md`，按需加载。
