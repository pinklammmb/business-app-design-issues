# 第三方 AI 建表页面接入协议

## 1. 适用场景

第三方页面负责通过 AI 生成表单配置，用户确认后跳转到表单创建页。第三方页面不需要调用创建页内部的字段初始化逻辑，只需要：

1. 按本文结构生成 JSON。
2. 将 JSON 写入当前浏览器 tab 的 `sessionStorage`。
3. 跳转表单创建页，并携带 `ai=true`。

创建页会自动读取 JSON，初始化表单名称和字段；审批模式还会初始化审批/抄送节点。

## 2. 必须满足的前提

### 2.1 `sessionStorage` 的同源要求

`sessionStorage` 按“协议 + 域名 + 端口 + 浏览器 tab”隔离。第三方页面必须和表单创建页处于同源环境，才能直接写入创建页可读取的 `sessionStorage`。

如果第三方页面与创建页不同源，不能通过第三方页面自身的 JavaScript 直接写入创建页的 `sessionStorage`。此时应由创建页所在源的父页面或桥接页面执行写入，再跳转创建页。

### 2.2 必须使用同一个 tab

写入配置和打开创建页必须在同一个浏览器 tab 中完成。使用 `window.open` 新开 tab 后，不能依赖原 tab 的 `sessionStorage` 传递配置。

## 3. 存储键

固定键名：

```text
ai-create-config
```

写入方式：

```js
sessionStorage.setItem('ai-create-config', JSON.stringify(config))
```

其中 `config` 必须符合本文第 4 节的 JSON 结构。

## 4. JSON 总体结构

最小可用结构：

```json
{
  "name": "高校学生信息表",
  "formConfig": [
    {
      "alias": "姓名",
      "compt": "editinput",
      "label": "姓名"
    }
  ]
}
```

完整结构：

```json
{
  "name": "学生请假审批表",
  "formConfig": [
    {
      "alias": "请假学生",
      "compt": "editinput",
      "label": "请假学生"
    },
    {
      "alias": "请假类型",
      "compt": "select",
      "label": "请假类型"
    }
  ],
  "summary": "用于收集学生请假信息并按流程审批",
  "publishImmediately": true,
  "flowConfigInfoArr": [
    {
      "id": 1,
      "title": "辅导员审批",
      "configType": "aprvUser",
      "aprvUserType": 3,
      "roleList": "[{\"roleName\":\"辅导员\"}]",
      "approveWay": 2
    }
  ]
}
```

## 5. 顶层字段定义

| 字段 | 类型 | 是否必填 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 创建页表单名称。建议非空，长度遵循创建页现有校验规则。 |
| `formConfig` | array | 是 | AI 生成的字段配置。没有字段时传空数组。 |
| `summary` | string | 否 | AI 生成摘要，主要用于审批模式展示。 |
| `flowConfigInfoArr` | array | 否 | 审批/抄送节点列表。普通建表、问卷建表可以省略。 |
| `publishImmediately` | boolean | 否 | 是否在创建页初始化完成后自动发布一次。仅严格等于 `true` 时生效，默认不自动发布。 |

创建页仅在 URL 带有 `ai=true` 时读取该配置。没有 `ai=true` 时，即使 `sessionStorage` 中存在该键，也不会应用 AI 配置。

### 5.1 立即发布参数

设置以下字段即可开启自动发布：

```json
{
  "publishImmediately": true
}
```

生效条件：

1. 创建页 URL 包含 `ai=true`。
2. `publishImmediately` 的值严格为布尔值 `true`，字符串 `"true"` 不生效。
3. 创建页已完成 AI 字段初始化；审批表单还会等待审批/抄送节点初始化完成。
4. 创建页复用现有“发布”入口执行一次完整发布校验和提交。

未传该字段、传 `false` 或传入其他类型时，创建页只初始化配置，不会自动发布。自动发布失败时保留创建页，按现有发布失败提示处理，第三方页面无需重复调用发布接口。

自动发布成功后，创建页不展示“发布成功”提示，也不执行普通发布成功跳转；创建页会向父窗口发送一次消息：

```js
{
  action: 'published',
  from: appStore.isApproveForm ? 'aprv' : 'form',
  data: { formId: formId }
}
```

第三方页面应监听 `message` 事件，并根据 `action === 'published'` 和 `data.formId` 处理后续逻辑。

## 6. `formConfig` 字段结构

每个字段对象结构如下：

| 字段 | 类型 | 是否必填 | 说明 |
| --- | --- | --- | --- |
| `alias` | string | 是 | 字段别名。创建页会写入最终字段对象。没有单独别名时可与 `label` 相同。 |
| `compt` | string | 是 | 创建页字段组件类型。必须使用第 7 节中的组件标识。 |
| `label` | string | 是 | 字段显示名称。 |

示例：

```json
{
  "alias": "联系电话",
  "compt": "editinput",
  "label": "联系电话"
}
```

### 6.1 组件类型处理规则

创建页会对 `compt` 做组件映射。无法识别的类型会自动降级为 `editinput`，但第三方页面仍应优先使用约定的类型，避免字段能力丢失。

建议第三方页面至少支持以下常用类型：

| `compt` | 适用字段 |
| --- | --- |
| `editinput` | 单行文本、姓名、学号、联系电话 |
| `textarea` | 长文本、备注、申请理由 |
| `select` | 单选下拉 |
| `multiple` | 多选 |
| `radio` | 单选按钮 |
| `checkbox` | 多选框 |
| `date` | 日期 |
| `datetime` | 日期时间 |
| `datetimerange` | 日期时间范围 |
| `number` | 数字 |
| `rate` | 评分 |
| `upload` | 附件或文件 |
| `belonger` | 所属人 |
| `relateddata` | 关联数据 |
| `detailcombox` | 子表单/明细表 |

实际可用类型以创建页 `supportFieldMap` 为准。第三方新增或使用其他类型前，应先和创建页维护方确认映射关系。

## 7. 审批流结构（可选）

只有审批建表场景需要传 `flowConfigInfoArr`。普通建表和问卷建表不要传无意义的审批节点。

节点公共字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 节点临时 ID，保证列表内唯一即可。 |
| `title` | string | 节点名称。 |
| `configType` | string | `aprvUser` 表示审批人；`ccUser` 表示抄送人。 |
| `approveWay` | number | 审批方式；没有特殊要求时可使用创建页默认值。 |

### 7.1 审批人节点

```json
{
  "id": 1,
  "title": "部门负责人审批",
  "configType": "aprvUser",
  "aprvUserType": 2,
  "aprvUsers": "[{\"deptLevel\":\"1\",\"deptTypeId\":\"-1\"}]",
  "approveWay": 2
}
```

`aprvUserType`：

| 值 | 含义 | 需要的字段 |
| --- | --- | --- |
| `2` | 主管/上级主管 | `aprvUsers`，JSON 字符串 |
| `3` | 角色 | `roleList`，JSON 字符串 |
| `6` | 其他用户类型 | 建议和创建页维护方确认；无法识别时会降级为发起人本人 |

主管配置示例：

```json
{
  "configType": "aprvUser",
  "aprvUserType": 2,
  "aprvUsers": "[{\"deptLevel\":\"1\",\"deptTypeId\":\"-1\"}]"
}
```

角色配置示例：

```json
{
  "configType": "aprvUser",
  "aprvUserType": 3,
  "roleList": "[{\"roleName\":\"辅导员\"}]"
}
```

角色名称必须与组织权限中心中的角色名称一致。创建页会按角色名称搜索真实角色 ID；搜索不到时会降级为发起人本人审批。

### 7.2 抄送节点

指定用户抄送：

```json
{
  "id": 2,
  "title": "抄送教务处",
  "configType": "ccUser",
  "ccUserType": 1,
  "ccUsers": "[{\"uid\":12345,\"n\":\"张三\"}]"
}
```

主管抄送：

```json
{
  "id": 2,
  "title": "抄送直属主管",
  "configType": "ccUser",
  "ccUserType": 2,
  "ccUsers": "[{\"deptLevel\":\"1\",\"deptTypeId\":\"-1\",\"uid\":0,\"n\":\"直接主管\"}]"
}
```

角色抄送：

```json
{
  "id": 2,
  "title": "抄送班主任",
  "configType": "ccUser",
  "ccUserType": 3,
  "roleList": "[{\"roleName\":\"班主任\"}]"
}
```

## 8. 跳转创建页

写入配置后，在同一个 tab 跳转创建页，并追加：

```text
ai=true
```

审批建表建议追加：

```text
formType=0
```

问卷建表追加：

```text
productType=1
comptControl=button,relateddata,belonger,livevideo
limitTabNum=false
```

示例：

```js
const config = {
  name: '高校学生信息表',
  formConfig: [
    { alias: '姓名', compt: 'editinput', label: '姓名' },
    { alias: '学号', compt: 'editinput', label: '学号' }
  ]
}

sessionStorage.setItem('ai-create-config', JSON.stringify(config))

const createPageUrl = '/manage/apps/forms/create?ai=true'
window.location.href = createPageUrl
```

实际创建页地址以前端环境配置中的 `VUE_APP_FORM_CREATE_PATH` 为准，不要在第三方页面硬编码完整域名。

## 9. 写入前校验

第三方页面提交前建议执行以下校验：

```js
function validateAiCreateConfig(config) {
  if (!config || typeof config !== 'object') return false
  if (typeof config.name !== 'string' || !config.name.trim()) return false
  if (!Array.isArray(config.formConfig)) return false

  return config.formConfig.every((field) => {
    return field && typeof field.label === 'string' && typeof field.compt === 'string'
  })
}
```

建议同时检查：

- `formConfig` 中没有重复或空的 `label`。
- `compt` 使用创建页已支持的组件类型。
- 审批节点的 `configType` 只使用 `aprvUser` 或 `ccUser`。
- `aprvUsers`、`ccUsers`、`roleList` 是合法 JSON 字符串，而不是对象或数组本身。
- 角色名称和组织权限中心中的名称一致。

## 10. 配置读取失败时的表现

| 问题 | 创建页表现 |
| --- | --- |
| 没有 `ai=true` | 按普通新建表单处理 |
| 没有 `ai-create-config` | 按普通新建表单处理 |
| JSON 无法解析 | 无法应用 AI 字段，需重新写入合法 JSON |
| `compt` 不支持 | 自动使用 `editinput` |
| 角色搜索不到 | 审批/抄送角色节点降级处理 |
| 未识别节点类型 | 跳过该节点 |

## 11. 联调最小用例

先使用以下最小配置确认链路可用：

```js
sessionStorage.setItem(
  'ai-create-config',
  JSON.stringify({
    name: 'AI 联调测试表',
    formConfig: [
      { alias: '姓名', compt: 'editinput', label: '姓名' },
      { alias: '申请日期', compt: 'date', label: '申请日期' }
    ]
  })
)
```

然后在同一个 tab 打开带 `ai=true` 的创建页，确认：

- 表单名称为“AI 联调测试表”。
- 创建页包含“姓名”和“申请日期”两个字段。
- 字段类型分别为文本和日期。

确认普通建表链路后，再增加 `flowConfigInfoArr` 验证审批人、角色和抄送节点。

## 12. 版本兼容约定

- `ai-create-config` 是当前协议版本的固定存储键，不要自行改名。
- 顶层字段和字段项名称保持兼容，新增字段应采用可选字段方式。
- 组件类型或审批节点类型发生变化前，需要同步更新本文档和创建页映射。
- 配置写入成功后应尽快跳转创建页，避免被同一 tab 中其他流程覆盖。
