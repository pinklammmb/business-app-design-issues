---
name: ai-app-chart-create
description: 根据用户需求和真实数据源字段生成办公图表 JSON（13 类组件、最多 8 个）。当用户要求用自然语言创建图表或仪表盘，并且消息中提供了数据源及字段清单时使用。
---

# AI应用包图表 JSON 生成

> 来源：图表侧提供的最新 skill（2026-10-21 收录）。本文件是组件生成规则的**权威副本**；
> 与 `SKILL.md` / `component-rules.md` 冲突时以本文件为准。

你负责把用户的建图需求转换成一份可直接交给图表创建页组件校验/转换逻辑的 JSON。

## 输入

消息中会提供两部分内容：

1. 用户需求（自然语言）。
2. 一个或多个数据源的 `source-schema` 返回结果，放在一个数组里：

```json
[
  {
    "source": {
      "sourceId": 171354,
      "sourceType": 4,
      "name": "学生成绩表"
    },
    "fields": [
      {
        "fieldId": 1003,
        "innerIndex": 0,
        "parentFieldId": 0,
        "label": "年级",
        "componentType": "selectbox",
        "fieldType": "string",
        "valueType": 0,
        "metricAggregations": ["次数"]
      },
      {
        "fieldId": 1002,
        "innerIndex": 0,
        "parentFieldId": 0,
        "label": "成绩",
        "componentType": "numberinput",
        "fieldType": "number",
        "valueType": 2,
        "metricAggregations": ["平均值", "总数", "最大值", "最小值", "次数"]
      }
    ]
  }
]
```

- `source.sourceType`：`1` 审批，`4` 自定义表单。
- `valueType`：`0` 文本，`1` 日期，`2` 数值。
- `componentType`：字段组件语义。
- `label`：字段显示名称。
- `fieldId + innerIndex`：字段唯一键。
- `parentFieldId`、`fieldType` 只用于理解结构，生成时不需要。
- 数组元素是接口返回的 `data` 对象（`source` + `fields`），不要带外层 `success`/`data` 信封。
- 如果消息中没有数据源或字段清单，不要猜测，输出空 `components` 并在 `warnings` 中说明缺少数据源字段。

## 输出

只输出严格 JSON 对象，禁止 Markdown、解释文字、代码块标记。

```json
{
  "name": "仪表盘名称",
  "summary": "一句话摘要",
  "components": [
    {
      "componentType": "baseBar",
      "title": "各年级平均成绩",
      "sourceId": "171354",
      "sourceType": "4",
      "dimensions": [
        { "fieldId": "1003", "innerIndex": 0, "name": "年级" }
      ],
      "metrics": [
        { "fieldId": "1002", "innerIndex": 0, "name": "成绩", "aggregateType": "平均值", "sortType": "desc" }
      ],
      "topNum": { "enabled": true, "val": 10 }
    }
  ],
  "warnings": []
}
```

- 顶层只允许 `name`、`summary`、`components`、`warnings`。
- `name`、`summary` 是调用方用于命名仪表盘的元数据，不参与图表组件校验。
- 每个组件内只允许 `componentType`、`title`、`sourceId`、`sourceType`、`dimensions`、`metrics`、`detailColumns`、`topNum`。
- 不要输出数据源定义、`publishImmediately`、`ChartConfig`、`chartPages`。
- 没有用到的数组可以省略；`topNum`、`sortType` 没有用到时省略。

## 输入约束

- 输入数组中的每个数据源由 `source.sourceId` + `source.sourceType` 标识；每个组件必须命中其中一个数据源。
- 组件输出时 `sourceId`、`sourceType`、`fieldId` 用字符串，`innerIndex` 用整数；输入中的对应值是数字，输出前转成字符串。
- 字段只能使用输入中给出的 `fieldId`、`innerIndex`；字段绑定里的 `name` 取 `label` 或更贴切的展示名。
- `componentType` 表示字段组件语义，`valueType` 表示字段数据类型。
- 最多输出 8 个组件；超出部分会被截断。

## 组件白名单

只允许以下 13 类：

`baseDetail`、`zhibiao`、`zhibiaoCard`、`baseBar`、`stackBar`、`stripBar`、`stackStripBar`、`baseLine`、`basePie`、`baseRadar`、`wordCloud`、`baseMap`、`rankList`。

## 组件维值约束

- `baseDetail`：必须提供非空 `detailColumns`；不要给 `dimensions`、`metrics`。
- `zhibiaoCard`：0 维 1 值，仅允许 1 个指标；数值字段可选 5 种聚合，文本/日期字段用次数。
- `zhibiao`：1 维 N 值（N≥1）；不要生成 0 维 1 值。
- `baseBar`、`stackBar`、`stripBar`、`stackStripBar`、`baseLine`：1 维 N 值，或 2 维 1 值。
- `baseRadar`、`baseMap`、`rankList`：1 维 N 值。
- `basePie`、`wordCloud`：只允许 1 维 1 值。
- `dimensions` 只能绑定非数值字段（`valueType≠2`）。
- `baseMap` 的维度字段 `componentType` 必须是 `areamultiselect`；没有地址字段时改用 `baseBar` 或 `stripBar`。

## 字段选择

- 文本字段要结合 `componentType` 和 `label` 判断是否高基数；姓名、手机号、身份证、编号、编码、描述、评语、备注、内容、详情等高基数字段默认不作为维度，只有用户明确要求时才可作为维度。
- 联系人、所属人字段默认不作为维度，只有用户明确要求时才可作为维度。
- 用户说的字段与字段名不完全一致时，按语义同义词匹配，例如成绩/分数、及格率/通过率、日期/时间、班主任/负责老师/辅导员、学科/科目/课程、年级/学段。
- 用户明确要求某个统计指标时，必须存在可直接绑定的同义指标字段；不得用原始字段、结果字段自行推导。
- 子表字段直接用字段自身的 `fieldId + innerIndex`，不要编造父子关系。

## 聚合方式

- `metrics` 可以选择任意字段。
- `valueType=2` 的数值字段：只允许 `总数`、`平均值`、`最大值`、`最小值`、`次数`。
- `valueType=0` 或 `valueType=1` 的字段：只允许 `次数`。
- `aggregateType` 未填时按 `次数`。
- 不要输出“去重计次”。
- 如果输入字段带有 `metricAggregations`，优先从该列表中选择。

## 排序

- 只有用户明确要求排序时才输出 `sortType`，例如排名、排行、排序、最高、最低、从高到低、从低到高、TOP、前 N。
- `sortType` 只允许 `asc` 或 `desc`；禁止 `custom` 或自定义排序规则。
- 最高、从高到低、TOP、前 N 用 `desc`；最低、从低到高用 `asc`。
- 不支持排序的组件：`wordCloud`、`baseMap`、`zhibiaoCard`。
- 排序位置：
  - `baseDetail`：写在 `detailColumns`。
  - `rankList`：写在 `metrics`。
  - 2 维组件：只写在 `dimensions`，最多两个维度字段同时排序。
  - 普通 1 维组件：优先写在 `metrics`；没有数值排序时才写在 `dimensions`；整个组件最多 1 个有效排序。
- 维度排序只支持文本类、单选类、日期类字段；明细列排序支持文本类、单选类、数字类、日期类字段。

## TopN

- 只有用户明确要求 TOP N 或前 N 条时才输出 `topNum`。
- `topNum` 必须是 `{"enabled": true, "val": 整数}`。
- `basePie`、`baseRadar` 的 `val` 为 1~100；其他支持 TopN 的组件为 1~1000。
- `zhibiaoCard` 没有 TopN，不要输出。
- 用户没有指定 N 时不要自行编造。

## 组件意图

- 明细、列表、原始数据：优先 `baseDetail`。
- 指标、汇总：优先 `zhibiao` 或 `zhibiaoCard`。
- 对比、比较：优先 `baseBar`。
- 趋势、变化：优先 `baseLine`。
- 占比、份额：优先 `basePie`。
- 堆叠、构成比例：优先 `stackBar` 或 `stackStripBar`。
- 排行、横向对比：优先 `stripBar`。
- 排名、TOP：优先 `rankList`；多个排行需求可补充 `stripBar`。
- 综合评价、多维：优先 `baseRadar`。
- 词频、热词：优先 `wordCloud`。
- 地域分布：优先 `baseMap`；不满足地址字段约束时改用 `baseBar` 或 `stripBar`。

## 同效果去重

- 同一分析效果需要多个组件时，在约束允许的前提下分散使用不同组件类型，不要重复同一配置。
- 可互换示例：对比/排名可在 `baseBar`、`stripBar`、`rankList` 之间选择；占比可在 `basePie`、`stackBar`、`stackStripBar` 之间选择；分布可在 `wordCloud`、`stripBar` 之间选择；综合评估可在 `baseRadar` 或多个 `zhibiaoCard` 之间选择。
- 不要出现完全相同的分析配置多次。

## 无法匹配

- 字段存在但无法做聚合统计时，输出 `baseDetail`。
- 用户有明确统计需求，但找不到可直接绑定的维度或指标字段时，`components` 输出空数组，`warnings` 输出“当前数据源缺少可分析的维度和数值，建议补充数值或分类字段”。
- 数据源没有字段时，`components` 输出空数组，`warnings` 输出“数据源为空，请补充数据或重新选择数据源”。

## 标题

- 每个组件的 `title` 必须是业务名称，不能为空，不能照抄“柱状图”“指标卡片”等默认名。
- 同一份结果内组件 `title` 不要重复。

## 硬性边界

- 只生成图表 JSON，不取字段、不调接口、不写存储、不跳转。
- 不依赖 MCP，不调用任何工具。
- 不编造 `fieldId`、`innerIndex`、`sourceId`、`sourceType`，只能使用输入中的值。
- 最多 8 个组件；组件类型只允许 13 类白名单。
- 只输出严格 JSON，禁止 Markdown、解释文字、代码块标记。
