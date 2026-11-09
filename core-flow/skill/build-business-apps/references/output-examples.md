# 节点结构化输出示例

以下假设请求中的应用搭建计划包含资产领用申请：a1为该计划内的逻辑应用ID，字段f1和节点n1映射为平台配置，不把内部ID写入config。示例假设后端已提供verified=true的真实资产管理员角色映射；交付例另假设已收到实际发布证据。ID为演示值，正式响应遵循调用契约。

## 应用配置输出示例

```json
{"status":"config_ready","construction_plan_version":2,"ref":"a1","config":{"name":"资产领用申请","formConfig":[{"alias":"领用数量","label":"领用数量","compt":"number"}],"flowConfigInfoArr":[{"id":1,"title":"资产管理员审批","configType":"aprvUser","aprvUserType":3,"roleList":"[{\"roleName\":\"资产管理员\"}]"}],"publishImmediately":true},"publish_allowed":true,"todos":[],"issues":[],"errors":[]}
```

## 交付结果输出示例

```json
{"status":"summarized","construction_plan_version":2,"overall":"complete","apps":[{"ref":"a1","name":"资产领用申请","type":"approval","status":"published","published_id":171354,"detail":"假设后端已核验发布成功"}],"todos":[],"issues":[],"scope_notes":["仅本次确认计划，无必要待办；独立未来扩展不在本次完成范围"],"errors":[]}
```
