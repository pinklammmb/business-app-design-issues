# 应用搭建计划转执行任务

prepare 直接接收用户已确认的 application_build_plan 和同版 requirements_input 槽位快照，不接收建设方案正文或章节。应用搭建计划是范围与应用结构的唯一入口；快照只补充其关联的业务细节及必须遵守的约束，不从快照扩出计划外应用。该步骤是校验和结构转换，明确映射时可由后端程序完成，无需额外调用模型。

完整输入字段与类型见本包 [计划结构](application-build-plan-schema.md) 和 [槽位快照规范](requirements-schema.md)。

## 输入校验

- application_build_plan 包含 forms、approvals、dashboards、pending_items 四个数组，允许空数组，但不能将缺失或 null 当成无该类需求。每个应用、字段、节点、组件都有稳定 id 及 requirement_ids；同计划ID唯一。表单和审批同时为空时仍可接收仅仪表盘计划，不再要求存在表单章节。
- requirements_input 与确认版本完全一致，包含28槽位快照；记录 id 全局唯一，全部 requirement_ids 必须存在。版本绑定由后端核对；跨版、重复ID或悬空引用返回 input_error，不按名字猜测换绑。
- 计划与快照中的已明确业务含义冲突时保留定位问题并阻止相关任务，不静默选择一方；返回问题，由调用方安排修正并重新确认计划。模型不能借 prepare 更改已确认计划。
- 空计划不伪造应用；可返回 prepared、tasks=[] 及范围说明，交付仍按 incomplete 判断，不记已完成。

## 结构转换

forms 每个对象生成一个 form 任务；approvals 每个对象生成一个 approval 任务，内含其字段和节点，不再创建同名普通表单；dashboards 每个对象生成一个 dashboard 任务。同名不同ID不擅合并，同ID重复是输入错误。

每项 task 固定为 {ref,type,name,requirement_ids,requirements,depends_on}：

- ref 直接沿用应用计划项 id，是逻辑ID，不是实际平台ID，不另分配新编号；type 为 form / approval / dashboard；name 沿用已确认名称。
- requirement_ids 合并应用及子项已有的槽位引用，去重，保留追溯依据。字段、节点、组件保留计划内子项 id 和 requirement_ids；这些元数据不写入平台 config。
- requirements：form 为 {fields:[]}，approval 为 {fields:[],nodes:[],branches:[]}，dashboard 为 {components:[]}。按计划保留字段和节点顺序；业务属性采用中文命名，已有字段名称与类型、节点名称、组件名称与类型直接映射，不根据名称重新设计。
- 逐条读取 requirement_ids 指向的槽位记录，补齐卡片未展示的已明确要求：审批办理身份、主管层级、抄送、分支、字段约束，以及图表逻辑数据源、维度、指标、聚合、过滤和口径。不得丢失已知细节；partial/conflict 如实记录 issues，不能补默认审批人或统计口径。
- AI补全项可能只关联支撑其设计的业务需求，其字段类型或组件类型直接以已确认计划为准，不能因为快照没有逐字重复计划而删掉它；也不能将AI补全写成用户明确要求。
- components 每项另有 source_ref，引用源应用的计划 id；明确需要多个源时用 source_refs 保留全部并记录当前协议限制。按关联槽位中的业务对象定位到计划项；多义或找不到则 source_ref=null 并记录 dependency/ambiguous，不任选相似名称。真实 sourceId、fieldId 等仍由后端后续接口获取。
- depends_on 为本次任务 ref 数组，去重且全部必须存在。当前契约只支持引用本次计划内的表单和审批；计划外既有源尚无统一运行记录契约，保留为 unsupported/dependency，不承诺本流程自动使用，也不能凭空加建应用。循环引用记录问题，由后端处理，不任意排序或无限等待。

## 未完成要求和范围

pending_items 不可原样当作 todos 透传：target_id 命中应用时直接使用该应用ref；命中字段/节点/组件时由其父应用取得ref；只有明确全局约束时 target_id=null 才作用于本次计划全局。目标找不到则记录 ambiguous，不静默丢弃。每项转换为 {target,requirement,reason,impact,follow_up,blocks_publish}，其中 follow_up 根据已知缺项写明补齐动作（不编造接口或页面），blocks_publish 按 capability-policy.md 判定，不能因用户确认计划变为false。可确定的人工项进入 manual_todos，业务未决同时记录 issues。关联的 C 类要求与明确全局约束一并保留。manual_todos 由调用端从已保存计划及槽位约束提供，不从建设方案章节再次提取。不得因精简卡片未展示权限或校验，就忽略已有的关键发布限制；不确定影响时按关键缺项处理。

issues 每项 {target,code,detail,blocking}；target 是任务 ref 或 application_build_plan；code 为 input / ambiguous / unresolved / unsupported / dependency。结构错误返回 input_error；业务缺口保留任务和issues，只有受影响应用及必要依赖受限，其余独立任务继续。

scope_notes 明确“仅执行已确认 application_build_plan 内应用；人工事项单列”，不宣称完整建设方案全部自动实现。缺项分级、todos 和发布门槛沿用 capability-policy.md；仪表盘仍在表单审批阶段结束、所引用源已发布认领且字段真实可用后生成。
