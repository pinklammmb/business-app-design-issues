# 节点结构化输出示例

以下为简短结构节选，不是完整接口响应。正式响应仍须补齐契约字段及全部28个槽位；需求ID仅供后台关联，不进入用户正文。

## 需求分析输出示例

```json
{"status":"ask","state":{"requirements":{"槽位分析":{"主要使用角色":[{"id":"s01","定位":"系统整体","已知信息":null,"状态":"missing","依据":[],"缺口或冲突":"使用角色未明确","是否需澄清":true}]}}},"questions":[{"question_id":"q1","type":"text","text":"哪些角色使用系统？"}]}
```

## 需求澄清输出示例

```json
{"status":"finished","requirements_input":{"槽位分析":{"审批人类型":[{"id":"s06","定位":"资产领用申请/审核节点","已知信息":"部门主管","状态":"resolved","依据":[{"来源":"q1回答","原文":"部门主管审批","推导理由":null}],"缺口或冲突":null,"是否需澄清":false}]}},"questions":[],"blocking_construction_plan":false}
```

finished 的 state.requirements 与 requirements_input 必须完全一致，此处省略重复快照。槽位记录ID只分配一次，后续修改沿用。
