# 审批与抄送节点（approval-nodes）

> 出处：《第三方 AI 建表页面接入协议》第 7 节。**仅审批建表场景**需要传 `flowConfigInfoArr`；**普通建表不要传无意义的审批节点**（普通建表见 [表单适配规则](form-approval.md)）。

## 1. 节点公共字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | number | 节点临时 ID，**保证列表内唯一即可**。 |
| `title` | string | 节点名称。 |
| `configType` | string | `aprvUser` 表示审批人；`ccUser` 表示抄送人。 |
| `approveWay` | number | 审批方式；**没有特殊要求时可使用创建页默认值**（示例中为 `2`）。 |

> 节点顺序即审批 / 抄送顺序，按本次任务中的审批流程链路依次排列。

## 2. 审批人节点（`configType: "aprvUser"`）

### 2.1 示例

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

### 2.2 `aprvUserType` 取值

| 值 | 含义 | 需要的字段 |
| --- | --- | --- |
| `2` | 主管 / 上级主管 | `aprvUsers`，JSON 字符串 |
| `3` | 角色 | `roleList`，JSON 字符串 |
| `6` | 其他用户类型 | **建议和创建页维护方确认**；无法识别时会降级为发起人本人 |

### 2.3 主管配置

```json
{
  "configType": "aprvUser",
  "aprvUserType": 2,
  "aprvUsers": "[{\"deptLevel\":\"1\",\"deptTypeId\":\"-1\"}]"
}
```

### 2.4 角色配置

```json
{
  "configType": "aprvUser",
  "aprvUserType": 3,
  "roleList": "[{\"roleName\":\"辅导员\"}]"
}
```

> **角色名称必须与组织权限中心中的角色名称一致。** 创建页会按角色名称搜索真实角色 ID；**搜索不到时会降级为发起人本人审批**。角色名称属真实组织数据，**由用户提供，AI 不得编造**。

## 3. 抄送节点（`configType: "ccUser"`）

### 3.1 指定用户抄送（`ccUserType: 1`）

```json
{
  "id": 2,
  "title": "抄送教务处",
  "configType": "ccUser",
  "ccUserType": 1,
  "ccUsers": "[{\"uid\":12345,\"n\":\"张三\"}]"
}
```

### 3.2 主管抄送（`ccUserType: 2`）

```json
{
  "id": 2,
  "title": "抄送直属主管",
  "configType": "ccUser",
  "ccUserType": 2,
  "ccUsers": "[{\"deptLevel\":\"1\",\"deptTypeId\":\"-1\",\"uid\":0,\"n\":\"直接主管\"}]"
}
```

### 3.3 角色抄送（`ccUserType: 3`）

```json
{
  "id": 2,
  "title": "抄送班主任",
  "configType": "ccUser",
  "ccUserType": 3,
  "roleList": "[{\"roleName\":\"班主任\"}]"
}
```

## 4. 抄送类型速查

| `ccUserType` | 含义 | 需要的字段 | 关键字段 |
| --- | --- | --- | --- |
| `1` | 指定用户 | `ccUsers`，JSON 字符串 | `uid`、`n`（成员真实 ID 与姓名，**由用户提供**） |
| `2` | 主管 / 直属主管 | `ccUsers`，JSON 字符串 | `deptLevel`、`deptTypeId`、`uid: 0`、`n` |
| `3` | 角色 | `roleList`，JSON 字符串 | `roleName`（**与组织权限中心一致**） |

## 5. 构造要点与常见坑

1. **字符串化**：`aprvUsers` / `ccUsers` / `roleList` 的值是**JSON 字符串**（对象序列化后的结果），**不是对象或数组本身**。写成对象 / 数组会导致创建页解析失败。
2. **`configType` 只有两种**：`aprvUser` / `ccUser`；**未识别的节点类型会被创建页直接跳过**。
3. **`id` 只需列表内唯一**，不是全局 ID；建议按 1、2、3 顺序递增。
4. **`summary` 与审批模式**：完整结构中的 `summary` 主要用于审批模式展示，审批建表建议补全。
5. **真实组织数据不编造**：`roleName`、指定抄送人的 `uid` / `n`、部门层级等均属真实组织数据，**必须由用户提供**；缺失时列入待确认，不得用占位值顶替。
6. **降级可预期**：
   - 角色搜索不到 → 审批 / 抄送角色节点降级；
   - `aprvUserType: 6` 无法识别 → 降级为发起人本人。
   生成时应**主动规避降级**，而不是依赖创建页兜底。
7. **不要给非审批表单塞节点**：普通建表省略 `flowConfigInfoArr`，应由 [表单适配规则](form-approval.md) 产出。
