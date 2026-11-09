# 应用搭建计划结构

application_build_plan是独立结构化对象，不是平台配置JSON。固定包含forms、approvals、dashboards、pending_items四个数组，允许空数组，不以缺失或null代替。对应槽位快照结构见 [槽位快照规范](requirements-schema.md)。

## 对象字段

- forms每项：id、name、requirement_ids、fields。字段每项：id、name、type、requirement_ids。
- approvals每项：id、name、requirement_ids、fields、nodes。字段结构同表单；节点每项：id、name、requirement_ids，按审批顺序排列。
- dashboards每项：id、name、requirement_ids、charts。组件每项：id、name、type、requirement_ids，按计划顺序排列。
- pending_items每项：target_id、requirement_ids、requirement、reason、impact。后三项为非空字符串；target_id指本计划的应用或子项ID，仅明确约束本计划全局时为null，不能掩盖目标不清。

所有应用及子项的id、name为非空字符串；id在整个计划内唯一。修改或改名保留ID，新增分配新ID，删除不复用。字段与组件的type为非空业务类型名称，具体平台支持范围按本包能力规则核对，不擅自更换语义。

requirement_ids为非空、去重的字符串数组，所有引用必须在同版槽位快照中存在；应用、子项和待确认事项均保留需求依据。审批办理身份、节点设置及图表逻辑数据源、维度、指标、聚合等执行细节不额外展示在简短卡片中，须能通过关联快照取得，不得只留在方案正文。

只列本次确认的自动搭建应用；影响本次计划的未决和人工补齐项进入pending_items，独立未来扩展不扩大本次执行范围。用户页面只展示业务名称、字段及类型、审批节点、组件名称及类型；内部ID和requirement_ids不写入平台config或用户下载正文。

## 简短示例（节选）

以下只展示一个审批，正式对象仍须保留四个数组；引用ID须存在于实际快照。

```json
{"approvals":[{"id":"a1","name":"领用申请","requirement_ids":["s01"],"fields":[{"id":"f1","name":"数量","type":"数字","requirement_ids":["s02"]}],"nodes":[{"id":"n1","name":"主管审批","requirement_ids":["s03"]}]}]}
```
