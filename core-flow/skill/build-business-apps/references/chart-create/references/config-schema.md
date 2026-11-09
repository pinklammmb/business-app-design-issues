# 图表建图配置 Schema（chart-create 参考）

> 真源：图表接入协议第 5~8、14、19 节。字段名与取值**不得改名**；新增能力一律走可选字段。
>
> **两段形状不要混淆**：组件生成（[ai-app-chart-create.md](ai-app-chart-create.md)）只输出
> `{name, summary, components, warnings}`；发往创建页的配置还要补 `dataSources` / `publishImmediately`，
> 并去掉 `warnings`。

## 1. 顶层结构

```json
{
  "name": "学生成绩分析仪表盘",
  "summary": "按年级和班级分析学生成绩分布",
  "publishImmediately": false,
  "dataSources": [
    { "sourceId": "171354", "sourceType": "4", "sourceName": "学生成绩表" }
  ],
  "components": [
    {
      "componentType": "baseDetail",
      "title": "成绩明细",
      "sourceId": "171354",
      "sourceType": "4",
      "detailColumns": [
        { "fieldId": "1001", "innerIndex": 0, "name": "姓名" }
      ]
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 仪表盘名称，非空 |
| `dataSources` | array | 是 | 数据源标识列表；**完整配置至少 1 个**（空数组合法但会被创建页按普通新建处理） |
| `components` | array | 是 | 组件列表，可为空数组 |
| `summary` | string | 否 | AI 生成摘要，用于展示 |
| `publishImmediately` | boolean | 否 | 仅严格布尔 `true` 生效；默认不自动保存 |

**顶层只允许以上五个字段。** 不要写 `formConfig`、`flowConfigInfoArr`、`formType`、`productType`
——那是表单 / 审批协议的形状。

## 2. `dataSources` 结构

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `sourceId` | string | 是 | 表单类数据源即表单 ID（`form_id`） |
| `sourceType` | string | 是 | `"1"` 审批 / `"4"` 自定义表单；**字符串形式的数字** |
| `sourceName` | string | 是 | 数据源显示名称 |

- 至少 1 个，支持多个；`sourceType` 传数字会被判为不匹配。
- 第三方**不提供字段清单**；`sourceEnc` 等附加字段由创建页加载时填充。
- 任一声明无效 / 无权限 / 无字段 → **整份配置被拒绝**，创建页按普通新建处理。

## 3. `components` 结构

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `componentType` | string | 是 | 13 类白名单之一（见 `component-rules.md`） |
| `title` | string | 是 | 组件标题 |
| `sourceId` | string | 是 | 必须命中 `dataSources` 中某一项 |
| `sourceType` | string | 是 | 必须与该数据源的 `sourceType` 一致 |
| `dimensions` | array | 否 | 维度字段绑定列表 |
| `metrics` | array | 否 | 数值字段绑定列表 |
| `detailColumns` | array | 否 | 明细列，仅 `baseDetail` 使用且必须非空 |
| `topNum` | object | 否 | 取前 N 条，`{ "enabled": true, "val": 10 }` |

字段绑定对象：

| 字段 | 类型 | 适用位置 | 说明 |
| --- | --- | --- | --- |
| `fieldId` | string | 全部 | 数据源真实字段 ID |
| `innerIndex` | number | 全部 | 子表内序号；非子表字段传 `0` |
| `name` | string | 全部 | 显示名称，不参与定位 |
| `aggregateType` | string | 仅 `metrics` | 聚合方式中文名（见 `component-rules.md`） |
| `sortType` | string | 全部 | 仅 `asc` / `desc` |

- **`fieldId` + `innerIndex` 二者组合唯一定位字段，缺一不可。**
- 组件数 > 8 时，第 9 个及后续被忽略。

## 4. 配置消息与创建页（postMessage 握手）

1. 创建页由平台统一入口在 iframe 里打开，URL 必须带 `ai=true`（创建页据此发 `ai:ready`）；
2. 收到 `{ type: 'ai:ready' }` 后，向握手来源 postMessage `{ key: 'ai-create-chart-config', data }`
   （**不是表单键**；`data` 是配置的 `JSON.stringify` 字符串）；
3. **不追加** `formType` / `productType`；
4. 实际地址以部署环境为准，**不要硬编码完整域名**。

```js
// 只在收到创建页 { type: 'ai:ready' } 后发送；
// data 必须是字符串，targetOrigin 用握手消息的 origin，不要用 '*'。
window.parent.postMessage(
  { key: 'ai-create-chart-config', data: JSON.stringify(config) },
  event.origin
)
```

约束：

- 配置**不再写 `sessionStorage`**（不落浏览器存储）；跨源同站的 iframe 也能收到配置，跨站不能内嵌
  ——平台会话 Cookie 是 `SameSite=Lax`，跨站 iframe 里带不过去，会落到登录页。
- 父子页面 10 秒内没完成握手时，父页提示「创建页没有回应配置握手，本次按空白新建打开」，按空白新建处理。
- 图表接入链路（协议第 4 / 5 / 14 / 15 节）**尚未实现**：本 skill 先把键、配置形状与 `ai=true` 对齐，
  待图表侧落地后即可联调。

## 5. 配置读取失败时创建页的表现

| 问题 | 创建页表现 |
| --- | --- |
| 没有 `ai=true` / 没有收到 `ai-create-chart-config` 配置消息 | 按普通新建仪表盘处理 |
| JSON 无法解析 | 无法应用配置，需重新发送合法 JSON 字符串 |
| 数据源标识无效 / 无权限 / 没有字段 | **整份配置拒绝**，按普通新建处理 |
| 组件类型不在 13 类白名单 / 未命中数据源 / 字段查不到 | **静默剔除该组件** |
| 维度绑定数值型字段 / 地图维度不是地址类 / 维值组合非法 | 静默剔除该组件 |
| 明细表格没有明细列 | 静默剔除该组件 |
| 数值聚合不在 5 种范围内 | 静默剔除该组件 |
| 文本 / 日期字段聚合不是 `次数` | 重置为 `次数`，组件保留 |
| 组件数 > 8 | 忽略第 9 个及后续 |
| 排序非法 / 排序字段类型不支持 | 清除排序配置，组件保留 |
| 取前 N 条数为空或越界 | 清除取前 N 配置，组件保留 |

> **组件剔除是静默的**：不弹提示、不中断建图。所以必须按 `validation-checklist.md` 先自检，
> 不能依赖创建页反馈。

## 6. 版本兼容约定

- `ai-create-chart-config` 是当前协议版本的固定配置消息 key（postMessage 的 `key`，不是浏览器存储键），不得改名。
- 顶层字段、组件项与字段绑定项名称保持兼容，新增能力走可选字段。
- 本协议的组件契约与站内 AI 建图、服务端校验规则同源；第三方生成的配置与站内 AI 配置走同一条校验路径。
