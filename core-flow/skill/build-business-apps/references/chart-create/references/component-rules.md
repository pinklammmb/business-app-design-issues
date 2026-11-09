# 图表组件规则（chart-create 参考）

> 权威来源：图表侧最新 skill [ai-app-chart-create.md](ai-app-chart-create.md)（2026-10-21 收录）。
> 本文件把规则压缩成自检口径；**不满足规则的组件会被创建页静默剔除**（不提示、不中断）。

## 1. 组件类型白名单（13 类）

| `componentType` | 中文名 | 典型用途 |
| --- | --- | --- |
| `baseDetail` | 明细表格 | 明细、列表、原始数据展示 |
| `zhibiao` | 指标表格 | 指标汇总 |
| `zhibiaoCard` | 指标卡片 | 单个核心数字 |
| `baseBar` | 柱状图 | 对比、比较 |
| `stackBar` | 堆叠柱状图 | 堆叠、构成比例 |
| `stripBar` | 条状图 | 排行、横向对比 |
| `stackStripBar` | 堆叠条状图 | 横向堆叠比例 |
| `baseLine` | 折线图 | 趋势、变化 |
| `basePie` | 饼状图 | 占比、份额 |
| `baseRadar` | 雷达图 | 多维综合评价 |
| `wordCloud` | 词云图 | 词频、热词 |
| `baseMap` | 地图 | 地域分布 |
| `rankList` | 排行榜 | 排名、TOP |

编辑器虽注册 35 类组件，AI 建图链路只放开以上 13 类；其余 22 类（柱线混合图、散点图、水波图、
进度条、时间轴、漏斗图、日历、天气、按钮、图片、智能文本、内嵌网页、筛选组件等）传入会被**直接剔除**。

## 2. 维度与数值组合

| 组合 | 允许的组件 |
| --- | --- |
| 0 维 1 值 | 仅 `zhibiaoCard`（指标表格不支持 0 维 1 值） |
| 1 维 N 值（N ≥ 1） | `zhibiao`、`baseBar`、`stackBar`、`stripBar`、`stackStripBar`、`baseLine`、`baseRadar`、`baseMap`、`rankList` |
| 2 维 1 值 | `baseBar`、`stackBar`、`stripBar`、`stackStripBar`、`baseLine` |
| 1 维 1 值 | `basePie`、`wordCloud`（这两类只支持 1 维 1 值） |
| 明细表格 | `baseDetail` 必须提供非空 `detailColumns`；不要给 `dimensions` / `metrics` |

字段语义硬校验：

- `dimensions` 只能绑定**非数值型**字段（`valueType≠2`）；绑定数值型字段 → 剔除组件。
- `baseMap` 的维度字段 `componentType` 必须是 `areamultiselect`；没有地址字段时改用 `baseBar` / `stripBar`。

## 3. 字段选择

- 只用输入中给出的 `fieldId + innerIndex`；子表 / 矩阵字段直接用自身键，不编造父子关系。
- 高基数字段默认不作为维度：姓名、手机号、身份证、编号、编码、描述、评语、备注、内容、详情等；
  用户明确要求时才可作为维度。
- 联系人、所属人字段默认不作为维度，用户明确要求时才可作为维度。
- 用户说法与字段名不完全一致时按语义同义词匹配：成绩/分数、及格率/通过率、日期/时间、
  班主任/负责老师/辅导员、学科/科目/课程、年级/学段。
- 用户明确要求某个统计指标时，必须存在可直接绑定的同义指标字段；不得用原始字段自行推导。

## 4. 聚合方式（`metrics.aggregateType`）

| 字段类型 | 允许值 |
| --- | --- |
| 数值型（`valueType=2`） | `总数`、`平均值`、`最大值`、`最小值`、`次数` |
| 文本型、日期型（`valueType=0/1`） | 只允许 `次数`；其他值自动重置为 `次数`，组件保留 |
| 未传 / 留空 | 按 `次数` 处理 |

- 优先从字段自带的 `metricAggregations` 里选择。
- `去重计次` 未在 AI 链路放开，**不要输出**（会被剔除组件）。

## 5. 排序（`sortType`）

- 只在用户明确要求排序时输出；只允许 `asc` / `desc`，`custom` 等非法值会被清除，组件保留。
- 最高、从高到低、TOP、前 N 用 `desc`；最低、从低到高用 `asc`。
- 不支持排序的组件：`wordCloud`、`baseMap`、`zhibiaoCard`。
- 排序必须写在被排序的字段项上：

| 组件 | 排序写在哪 |
| --- | --- |
| `baseDetail` | `detailColumns`（`dimensions` / `metrics` 上的排序会被清除） |
| `rankList` | `metrics`（`dimensions` / `detailColumns` 上的排序会被清除） |
| 2 维组件 | 只写在 `dimensions`，最多 2 个维度字段同时排序 |
| 普通 1 维组件 | 优先保留 `metrics` 排序；没有数值排序时才保留 `dimensions` 排序；整个组件最多 1 个有效排序 |

- 维度排序只支持文本类、单选类、日期类；明细列排序支持文本类、单选类、数字类、日期类。
- 不满足时只清除排序配置，组件保留。

## 6. 取前 N 条（`topNum`）

```json
{ "topNum": { "enabled": true, "val": 10 } }
```

- 只在用户明确要求 TOP N / 前 N 条时输出；不要自造 N。
- `enabled` 必须是布尔 `true`；`val` 必须是整数。
- `basePie`、`baseRadar`：`1 ~ 100`；其余支持 TopN 的组件：`1 ~ 1000`。
- `val` 为空或越界 → **整个取前 N 配置被清除，不做修正**，组件保留。
- `zhibiaoCard` 没有取前 N 配置项。

## 7. 组件意图与去重

| 需求意图 | 优先组件 |
| --- | --- |
| 明细 / 列表 / 原始数据 | `baseDetail` |
| 指标 / 汇总 | `zhibiao`、`zhibiaoCard` |
| 对比 / 比较 | `baseBar` |
| 趋势 / 变化 | `baseLine` |
| 占比 / 份额 | `basePie` |
| 堆叠 / 构成比例 | `stackBar`、`stackStripBar` |
| 排行 / 横向对比 | `stripBar` |
| 排名 / TOP | `rankList`（可补 `stripBar`） |
| 综合评价 / 多维 | `baseRadar` |
| 词频 / 热词 | `wordCloud` |
| 地域分布 | `baseMap`（不满足地址字段约束时用 `baseBar` / `stripBar`） |

- 同一分析效果不要输出完全相同的配置；约束允许时分散使用不同组件类型
  （对比/排名：`baseBar` / `stripBar` / `rankList`；占比：`basePie` / `stackBar` / `stackStripBar`；
  分布：`wordCloud` / `stripBar`；综合评估：`baseRadar` / 多个 `zhibiaoCard`）。
- 每个组件 `title` 必须是业务名称，不能为空，不能照抄「柱状图」「指标卡片」等默认名；同一结果内不重复。

## 8. 无法匹配

- 字段存在但无法做聚合统计 → 输出 `baseDetail`。
- 用户有明确统计需求，但找不到可直接绑定的维度 / 指标字段 → `components` 输出空数组，
  `warnings` 输出“当前数据源缺少可分析的维度和数值，建议补充数值或分类字段”。
- 数据源没有字段 → `components` 输出空数组，`warnings` 输出“数据源为空，请补充数据或重新选择数据源”。

## 9. 组件静默剔除清单（写配置前逐项核对）

| 触发条件 | 结果 |
| --- | --- |
| 组件类型不在 13 类白名单 | 剔除组件 |
| 组件的数据源未命中 `dataSources` | 剔除组件 |
| 字段绑定查不到对应字段（`fieldId` / `innerIndex` 不对） | 剔除组件 |
| 维度绑定了数值型字段 | 剔除组件 |
| 地图组件的维度不是 `areamultiselect` 地址字段 | 剔除组件 |
| 维度与数值组合不合法 | 剔除组件 |
| 明细表格没有明细列 | 剔除组件 |
| 数值型字段的聚合方式不在 5 种范围内 | 剔除组件 |
| 文本 / 日期型字段聚合不是 `次数` | 重置为 `次数`，组件保留 |
| 组件数量超过 8 个 | 忽略第 9 个及后续 |
| 排序方式非法或排序字段类型不支持 | 清除排序，组件保留 |
| 取前 N 条数为空或越界 | 清除取前 N，组件保留 |

> 创建页**不会**反馈组件被剔除。典型症状是「传了 5 个组件、页面上只出现 3 个」；
> 「暂无图表数据」通常意味着字段绑定失败，优先核对 `fieldId` 与 `innerIndex`。
