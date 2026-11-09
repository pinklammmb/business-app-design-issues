# 表单与审批配置适配

本文件已将 form-create、approval-create 的适用规则整理在本包：配置形状与校验见本文，字段映射见 [字段类型](field-types.md)，审批结构见 [节点规范](approval-nodes.md)，发布约束见 [发布控制](capability-policy.md)。生成时只读取本包文件，不要求外部安装原 skill 或读取原位置。

## 本流程与原 skill 的差异

1. 以已确认应用搭建计划为入口，槽位引用补齐审批人和字段约束；任务已结构化，不按任何建设方案章节扫描文档。
2. 模型返回 JSON 配置给后端，由后端校验并保存为应用配置JSON文件；打开空白创建页后由页内读取并回显，创建后自动发布并回传ID，见 [创建页流程](form-approval-entry.md)。模型不自行落盘、打开页面或向用户追问文件名。缺信息用 issues 返回。
3. 新 [创建页协议](session-storage-contract.md) 已增加 publishImmediately，本流程表单与审批配置须按 [发布控制](capability-policy.md) 设置布尔值，允许发布为 true，关键缺项未补齐为 false，覆盖旧 skill 顶层白名单限制。普通表单允许 name、formConfig、summary、publishImmediately；审批另加 flowConfigInfoArr。
4. 不依赖创建页静默降级：不能映射的要求保留为 todos，生成合法的已支持部分；关键缺项设 publish_allowed=false。无法形成有效配置才 blocked、config=null，不把审批角色搜不到后改为发起人视为成功。

## 生成

- name 取任务名称，字段保留顺序，formConfig 非空；每项 alias、compt、label 为非空字符串，label 不重名。alias 有设计值则沿用，无则与 label 相同。
- 按 [字段白名单](field-types.md) 映射类型，数值、日期精度等语义不能改变。可依据明确语义选择控件，不能补造字段或选项。
- 普通表单不含 flowConfigInfoArr。审批配置同时包含字段与流程；发起、结束等展示节点不冒充审批人或抄送节点，审批节点顺序与设计一致。
- 按 [节点规范](approval-nodes.md) 生成 aprvUser、ccUser；节点 id 在列表内唯一。aprvUsers、ccUsers、roleList 为合法 JSON 字符串，不是数组本身。
- 真实角色、部门、成员由输入 organization 提供。业务“管理员”不自动等于组织权限中心同名角色；organization.roles 明确业务身份与真实角色名称的映射及 verified=true 后才采用。指定用户使用输入的真实 uid、n。仅有“主管”不能擅定缺失的部门层级参数。
- approveWay 未有特殊要求时可省略由创建页默认处理；明确多人审批规则却无编码依据时列关键待办并禁止相关应用发布，不能从示例的 2 猜测含义。aprvUserType=6 未有明确适配约定不得采用。

## 配置支持范围

字段选项值按 slot-policy.md 归 C 类，默认不追问，用户已给出的枚举仍须保留为未实现要求。现有协议字段项只定义 alias、compt、label，尚未定义必填、默认值、选项、公式、关联目标和子表结构如何传输；审批协议尚未定义条件分支、节点字段权限的完整编码。

提取时完整保留这些要求；生成时按 capability-policy.md 区分普通与关键缺项，均列入 todos，继续生成协议支持的字段与节点。可形成合法配置则 config_ready；普通缺项不阻止发布，关键缺项设 publish_allowed=false、publishImmediately=false。不能形成合法有效配置才 blocked、config=null。不写自造键，不用错误替代实现，不断言平台本身不支持。

以后有经过确认的扩展协议，应更新本参考再使用；本版不以任意 JSON 扩展替代明确契约。

## 自检

检查顶层白名单、字段与组件类型、流程顺序、角色映射、JSON 字符串、真实人员及协议覆盖。仍影响关键业务行为的缺项或待确认项禁止相关应用发布。config_ready 表示已支持部分JSON可交后端保存为文件并由创建页读取回显；发布必须另核对 publish_allowed，未实现内容保留在 todos。
