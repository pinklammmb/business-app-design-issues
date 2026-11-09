# 节点结构化输出示例

以下为简短结构节选，不是完整接口响应。正式响应仍须补齐契约字段及全部28个槽位；需求ID仅供后台关联，不进入用户正文。

## 应用搭建计划输出示例

```json
{"application_build_plan":{"forms":[],"approvals":[{"id":"a1","name":"资产领用申请","requirement_ids":["s06"],"fields":[{"id":"f1","name":"领用数量","type":"数字","requirement_ids":["s02","s03"]}],"nodes":[{"id":"n1","name":"部门主管审批","requirement_ids":["s06"]}]}],"dashboards":[],"pending_items":[]}}
```

## 方案修改输出示例

```json
{"status":"ready","requirements_input":{"槽位分析":{"审批人类型":[{"id":"s06","定位":"资产领用申请/审核节点","已知信息":"资产管理员","状态":"resolved","依据":[{"来源":"本次修改","原文":"改成资产管理员审批","推导理由":null}],"缺口或冲突":null,"是否需澄清":false}]}},"application_build_plan":{"forms":[],"approvals":[{"id":"a1","name":"资产领用申请","requirement_ids":["s06"],"fields":[{"id":"f1","name":"领用数量","type":"数字","requirement_ids":["s02","s03"]}],"nodes":[{"id":"n1","name":"资产管理员审批","requirement_ids":["s06"]}]}],"dashboards":[],"pending_items":[]},"changes":[{"target":"资产领用申请/审核节点","summary":"审批人及对应节点同步更新，完整方案跟随调整"}]}
```

示例省略节点名称等其他槽位记录，展示 n1 通过 requirement_ids 关联 s06 的机制。计划可以同时关联多条记录。修改例保留 s06、a1、n1 的ID，实际响应还须返回完整 construction_plan_markdown；完整方案不逐段绑定ID，正文示例见 [建设方案示例](blueprint-example.md)。
