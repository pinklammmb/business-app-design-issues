# 图表配置发送前校验（chart-create 参考）

> 真源：图表接入协议第 16~18 节。发送配置消息前必须完成本文件的自检；
> **组件被剔除是静默的**，不能依赖创建页反馈。
>
> 组件生成阶段若按 [ai-app-chart-create.md](ai-app-chart-create.md) 输出 `warnings`，发往创建页前
> 必须先把 `warnings` 里的缺口处理掉或如实回显；`warnings` 本身不进入发往创建页的配置。

## 1. 校验函数

```js
function validateAiCreateChartConfig(config) {
  if (!config || typeof config !== 'object') return false
  if (typeof config.name !== 'string' || !config.name.trim()) return false
  if (!Array.isArray(config.dataSources) || config.dataSources.length === 0) return false
  if (!Array.isArray(config.components)) return false
  if (config.components.length > 8) return false

  const ALLOWED_TYPES = [
    'baseDetail', 'zhibiao', 'zhibiaoCard', 'baseBar', 'stackBar',
    'stripBar', 'stackStripBar', 'baseLine', 'basePie', 'baseRadar',
    'wordCloud', 'baseMap', 'rankList'
  ]
  const ALLOWED_SOURCE_TYPES = ['1', '4']
  const AGGREGATE_TYPES = ['总数', '平均值', '最大值', '最小值', '次数']

  const sourceKeys = new Set()
  for (const source of config.dataSources) {
    if (!source || !source.sourceId || !source.sourceType || !source.sourceName) return false
    if (typeof source.sourceId !== 'string' || typeof source.sourceType !== 'string') return false
    if (!ALLOWED_SOURCE_TYPES.includes(source.sourceType)) return false
    sourceKeys.add(`${source.sourceType}_${source.sourceId}`)
  }

  return config.components.every((comp) => {
    if (!comp || !ALLOWED_TYPES.includes(comp.componentType)) return false
    if (!sourceKeys.has(`${comp.sourceType}_${comp.sourceId}`)) return false

    const dimensions = comp.dimensions || []
    const metrics = comp.metrics || []
    const detailColumns = comp.detailColumns || []

    if (comp.componentType === 'baseDetail' && detailColumns.length === 0) return false

    const bindings = [...dimensions, ...metrics, ...detailColumns]
    const fieldsValid = bindings.every(
      (f) => f && f.fieldId !== undefined && f.fieldId !== null &&
        typeof f.innerIndex === 'number' &&
        (f.sortType === undefined || f.sortType === null || f.sortType === '' ||
          f.sortType === 'asc' || f.sortType === 'desc')
    )
    if (!fieldsValid) return false

    return metrics.every(
      (m) => m.aggregateType === undefined || m.aggregateType === null ||
        m.aggregateType === '' || AGGREGATE_TYPES.includes(m.aggregateType)
    )
  })
}
```

## 2. 检查清单

| # | 检查项 | 结果 |
| --- | --- | --- |
| 1 | `name` 非空 | ✅ / ⚠️ |
| 2 | `dataSources` 为数组；完整配置至少 1 个真实数据源 | ✅ / ⚠️（空 = 如实说明） |
| 3 | 每个数据源 `sourceId` / `sourceType` / `sourceName` 齐全 | ✅ / ⚠️ |
| 4 | `sourceId` / `sourceType` 是**字符串**，`sourceType` ∈ `{"1","4"}` | ✅ / ⚠️ |
| 5 | `components` 为数组，数量 ≤ 8 | ✅ / ⚠️ |
| 6 | 每个组件 `componentType` ∈ 13 类白名单 | ✅ / ⚠️ |
| 7 | 每个组件命中 `dataSources`（`sourceType + sourceId` 一致） | ✅ / ⚠️ |
| 8 | 字段绑定同时给了 `fieldId` 与 `innerIndex` | ✅ / ⚠️ |
| 9 | 维值组合合法（0 维 1 值 / 1 维 N 值 / 2 维 1 值 / 饼图 1 维 1 值 / 明细列非空） | ✅ / ⚠️ |
| 10 | `metrics.aggregateType` ∈ 5 种；文本 / 日期只用 `次数` | ✅ / ⚠️ |
| 11 | `sortType` 只用 `asc` / `desc`，且写在允许的位置 | ✅ / ⚠️ |
| 12 | `topNum.val` 在组件范围内，`enabled` 为布尔 `true` | ✅ / ⚠️ |
| 13 | 组件标题非空、不重复 | ✅ / ⚠️ |
| 14 | 顶层没有多余字段（无 `formConfig` / `flowConfigInfoArr` / `formType`） | ✅ / ⚠️ |
| 15 | 配置消息 key 是 `ai-create-chart-config`（不是 `ai-create-config`） | ✅ / ⚠️ |
| 16 | 跳转 URL 带 `ai=true`，且没有 `formType` / `productType` | ✅ / ⚠️ |
| 17 | 只在收到 `{ type: 'ai:ready' }` 后发送，`targetOrigin` 用握手 `origin`（不用 `*`） | ✅ / ⚠️ |

## 3. 第三方自查的能力边界

本地能校验：名称、数据源字段齐备与类型、组件白名单、组件与数据源匹配、字段绑定齐全、聚合方式取值、
排序取值、组件数量。

**本地无法校验**（只有创建页加载字段清单后才能判定）：

- 字段是否真实存在、是数值型还是文本型、是否为地址类字段；
- 维度与数值组合是否满足第 10 节全部规则（建议本地实现同等判断）；
- 数据源是否无效 / 无权限 / 没有字段。

建议同时检查：组件标题不重复不空；`baseDetail` 以外不依赖 `detailColumns`；2 维组件只用 5 类；
`basePie` / `wordCloud` 只用 1 维 1 值；排序与取前 N 只写在允许的位置与范围。

## 4. 联调最小用例

```js
// 父页 / 打开者：只在收到创建页的 { type: 'ai:ready' } 后发送配置
window.addEventListener('message', (event) => {
  if (event.data?.type !== 'ai:ready') return
  event.source.postMessage({
    key: 'ai-create-chart-config',
    data: JSON.stringify({
      name: 'AI 联调测试仪表盘',
      dataSources: [
        { sourceId: '替换为真实表单ID', sourceType: '4', sourceName: '替换为真实表单名' }
      ],
      components: [
        {
          componentType: 'baseDetail',
          title: '数据明细',
          sourceId: '替换为真实表单ID',
          sourceType: '4',
          detailColumns: [
            { fieldId: '替换为真实字段ID', innerIndex: 0, name: '替换为真实字段名' }
          ]
        }
      ]
    })
  }, event.origin)
})
```

在 iframe（或新窗口）里打开 `/manage/apps/chart/create?ai=true`，确认：

- 仪表盘名称为「AI 联调测试仪表盘」；
- 数据源已挂载为指定表单；
- 页面出现明细表格组件并列出指定字段；
- 组件有数据，不是「暂无图表数据」。

明细表格通了之后，再依次验证：指标卡片（0 维 1 值）→ 指标表格（1 维 N 值）→ 柱状图（含聚合 / 排序）
→ 饼状图（含取前 N）→ 地图（地址类字段）。

> 管理侧「空配置」口径：`dataSources: []` 或 `sourceId` 是占位文案时，创建页会整份拒绝、按普通新建处理。
> 这不是失败路径，而是本系统不伪造数据源时的如实退化。

## 5. 读取失败时的创建页表现

| 问题 | 创建页表现 |
| --- | --- |
| 没有 `ai=true` 或没有收到 `ai-create-chart-config` 配置消息 | 按普通新建仪表盘处理 |
| JSON 无法解析 | 无法应用配置，需重新发送合法 JSON 字符串 |
| 数据源无效 / 无权限 / 无字段 | 整份配置拒绝，按普通新建处理 |
| 组件被剔除（类型 / 数据源 / 字段 / 维值组合 / 聚合） | 静默剔除该组件 |
| 组件数 > 8 | 忽略第 9 个及后续 |
| 排序 / 取前 N 非法 | 清除对应配置，组件保留 |
