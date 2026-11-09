# 分阶段调用契约

模型不依赖聊天记忆。后端提供 SKILL.md 及本阶段所需参考正文，或使用可读取本包的执行环境；单独传本地路径不代表普通模型 API 能读文件。以下 {} 是结构简写，实际调用须传完整对象。

## 1. prepare

```json
{"operation":"prepare","construction_plan_version":1,"confirmed":true,"application_build_plan":{"forms":[],"approvals":[],"dashboards":[],"pending_items":[]},"requirements_input":{"槽位分析":{}},"manual_todos":[]}
```

示例仅展示请求字段；真实 requirements_input 须包含完整28槽位记录。construction_plan_version 是已确认版本的正整数，confirmed 必须为 true。后端核对计划、快照和确认属于同一版本；未确认或跨版本不进入搭建。

application_build_plan 为调用方提供的已确认结构化搭建计划，完整结构见本包 [计划结构](application-build-plan-schema.md)。requirements_input 按本包 [槽位快照规范](requirements-schema.md) 校验，通过稳定槽位ID提供关联需求，不传问答历史，不传建设方案正文。manual_todos 可省略为 []，从同版计划和槽位约束提供；prepare 同时处理 pending_items，不要求人工待办重复提供。详见 [计划转执行任务](extraction.md)。本入口只接受以上字段，不兼容旧的章节参数；旧调用方须切换到已保存的结构化产物。

响应：

```json
{"status":"prepared","construction_plan_version":1,"tasks":[],"manual_todos":[],"issues":[],"scope_notes":["仅执行已确认应用搭建计划内的应用；人工事项单列"],"errors":[]}
```

任务 ref 沿用原应用计划 id，不重新编号。prepared 只表示完成校验及结构转换，不表示任务全部可执行或已创建。结构或引用错误 status=input_error，tasks=[]、manual_todos=[]、issues=[]、scope_notes=[]，errors 为 {field,message} 数组；版本无效时 construction_plan_version=null。业务缺口按 extraction.md 保留任务及问题，不能静默补造配置。

## 2. generate_config

一次处理一个 task；后端提供已保存的计划及同版状态。

```json
{
  "operation":"generate_config",
  "plan":{"status":"prepared","construction_plan_version":1,"tasks":[],"manual_todos":[],"issues":[],"scope_notes":[],"errors":[]},
  "target_ref":"app_1",
  "runtime":[],
  "organization":{"roles":[],"users":[],"supervisors":[]},
  "sources":[]
}
```

- plan 为 prepare 返回的完整执行任务计划，区别于请求输入的 application_build_plan；本阶段不再读取正文或章节。target_ref 须唯一匹配一个任务。后端保证仍对应已确认版本，不接受其他版本混合结果。
- runtime 为后端全部运行记录，格式见 runtime.md。表单/审批要求空白创建页已实际打开并就绪（page_ready），此时不要求已有应用ID，blank_app_id 可为 null；读取JSON文件并回显后为 configured，不代表已创建或发布。仪表盘仍按其适配约定要求 blank_created 及真实 blank_app_id。前置不满足则 waiting，不伪造页面状态或应用ID。目标已 published 时直接由后端复用配置，不应再次生成；误调返回 blocked 并说明已发布。
- organization.roles 每项 {business_role,role_name,verified}；users 每项 {business_identity,uid,n}；supervisors 每项 {business_identity,deptLevel,deptTypeId}。仅提供本任务必要项；角色映射 verified 必须为 true，用户身份及组织参数须由后端或用户提供，不猜测。无需组织信息的任务可用空数组。
- sources 每项 {ref,schema}；ref 指源应用任务，schema 为 auto-flow 字段接口经后端适配后的 {source,fields}，形状按 chart-create 的组件规范，不传接口外层信封。普通表单、审批可传 []。
- 仪表盘需等本计划全部表单、审批任务进入终态 published / failed / blocked；所引用源任务必须 published 且 claimed=true，所需 sources 须齐全，sources[].schema.source.sourceId/sourceType 与相应 runtime.source_id/source_type 转字符串后一致。前置仍在执行或接口字段尚未返回时 waiting；引用的源应用已失败或禁止发布时，将该仪表盘记 blocked 并列依赖待办，不能无限等待；已成功获取但 fields 为空则按 chart-create 返回诊断并 blocked。

响应固定为：

```json
{
  "status":"config_ready",
  "construction_plan_version":1,
  "ref":"app_1",
  "config":{},
  "publish_allowed":true,
  "todos":[],
  "issues":[],
  "errors":[]
}
```

- status 为 config_ready / waiting / blocked / input_error。
- config_ready 的 config 为结构完整、只包含当前已支持部分的创建页 JSON，不能加入 ref、状态、问题等包装字段。form/approval 见 form-approval.md；dashboard 按 dashboard-adapter.md 加载 chart-create，最终 config 含 dataSources、与 publish_allowed 相同的布尔 publishImmediately，不含 warnings；不是仅包装组件生成结果。
- waiting：缺运行前置结果，config=null，issues 说明缺项；不问用户业务问题，不模拟接口响应。
- blocked：不能生成结构合法且有实际用途的配置，或必需依赖已失败，config=null。仅缺部分能力不直接 blocked；按 capability-policy.md 生成可用部分并列 todos。图表不发布空图、不用错误指标替代缺失指标；可独立实现的其他组件继续生成。诊断进 issues，不写入 config.warnings。
- publish_allowed：布尔值。config_ready 且无关键缺项时为 true；其他情况为 false。config 非 null 时 publishImmediately 必须严格等于此值，不能沿用默认自动发布。
- todos：未实现要求数组，结构和分级见 [部分实现及发布控制](capability-policy.md)。普通待办与 config_ready 可同时存在；关键待办不丢弃可用配置，但禁止发布。waiting、blocked 仍保留已识别待办。input_error 时 todos=[]。
- input_error：请求结构错误、未知 ref、重复 ref/运行记录、版本不一致或数据源对应错误。config=null，issues=[]，errors 为 {field,message} 数组；无法识别版本或 ref 时相应值为 null。其他情况 errors=[]。
- 生成前同时检查 plan.manual_todos 中与目标相关的关键人工待办；不能等最终汇总时才发现权限等发布限制。准备阶段的 blocking issue 在生成前检查并按实际影响转为发布阻断或生成阻断；能力缺项进 todos，不一律阻止生成配置。运行结果可解除依赖问题，但不能代替业务决定。

## 3. summarize

```json
{
  "operation":"summarize",
  "plan":{"status":"prepared","construction_plan_version":1,"tasks":[],"manual_todos":[],"issues":[],"scope_notes":[],"errors":[]},
  "config_results":[],
  "runtime":[],
  "manual_todos":[]
}
```

config_results 为同版本保存的 generate_config 响应数组；每个 ref 仅传最新有效结果，不重新消费生成历史。runtime 为实际运行记录。汇总 plan.manual_todos、配置结果 todos 及调用方本次补充的 manual_todos，语义相同项去重；后者为同版已保存应用搭建计划及槽位约束中的人工事项，使用相同结构，可省略为 []，不要求 skill 读取全文。影响发布的人工项须在 prepare/生成前提供并执行门槛，不能到交付才首次检查。响应字段固定为 status（summarized / input_error）、construction_plan_version、overall、apps、todos、issues、scope_notes、errors；正常汇总按 runtime.md 判断，errors=[]。输入格式错误则 overall=null、apps=[]、todos=[]、issues=[]、scope_notes=[]，errors 给出错误。

调用方可用程序直接完成相同汇总，无须为此增加模型调用。任何操作都不调用真实创建、发布或字段接口；本包描述的是编排边界，不是这些 API 的可执行实现。

## 参数命名迁移

调用端、响应解析和已保存状态须同步迁移为当前参数名称，不混用旧名称。requirements_input 表示结构化需求输入；construction_plan_markdown 表示建设方案正文；current_construction_plan 表示当前方案正文；construction_plan_version 表示方案版本；blocking_construction_plan 表示是否阻塞方案生成。本契约只使用实际涉及的字段，不因此增加额外请求参数。

## 配置生成与结果汇总的交接

plan 直接使用 prepare 的完整已保存响应，尤其保留 manual_todos、issues、scope_notes，不能只截取 tasks 而丢失发布门槛。仅 status=prepared 才进入配置；prepare=input_error 不打开表单/审批创建页，也不创建仪表盘空白应用。generate_config 原响应按 ref 保存进 config_results；平台真实结果另写 runtime，不能把 config.status 当作 runtime.status。summarize 使用同一 construction_plan_version 的 plan、config_results 和 runtime。
