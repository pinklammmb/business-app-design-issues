# 调用与交付契约

后端持久化需求、业务系统建设方案及版本；一次模型调用只完成一次状态转换。向模型提供 SKILL.md 和所需 reference 正文；只传本地路径不代表普通 API 能读取文件。

## 会话边界

本 skill 的对话修改入口仅用于尚未完成搭建的创建会话。调用方须依据已持久化的真实会话状态拦截已搭建完成会话的 revise 请求，持续禁用输入框与发送入口，刷新后也不恢复。模型不依靠聊天内容猜测是否已完成，不新增请求字段；不得因 ready、unchanged 或追问结束而重新开放已完成会话。

## 请求

首次生成：

```json
{
  "operation":"generate",
  "requirements_input":{}
}
```

对话修改或补充回答：

```json
{
  "operation":"revise",
  "base_version":1,
  "current_construction_plan":"当前版本完整 Markdown 正文",
  "current_application_build_plan":{},
  "requirements_input":{},
  "change_request":{"source_id":"m12","text":"完整原修改请求"},
  "answers":[]
}
```

- requirements_input 必须为 blueprint-input.md 定义的完整28槽位快照，字段、类型、状态与引用约束按本包 [槽位快照规范](requirements-schema.md) 校验，含稳定记录 id；{} 仅为简写。不重新运行需求分析，不把它改写成主题摘要。
- revise 必须有对应当前版本的完整业务系统建设方案、current_application_build_plan 和有效需求；计划结构见 [应用搭建计划输出](application-build-plan.md)，请求示例的 {} 仅为结构简写，不是有效计划。base_version 为后端提供的正整数。缺失版本或输入不成对时 input_error，不从聊天摘要补造。
- change_request 的 source_id、text 为非空字符串；来源标识由调用方提供。
- answers 默认为 []；每项为 {question_id,selected_values,text,skipped}，按本包 [追问设计](question-design.md) 校验单选、多选、文本及跳过答案，见 [对话修改追问规则](revision-questions.md)。补充调用须由后端同时提供上次返回的完整 questions 数组（字段名 prior_questions），以关联问题；仅有答案编号不代表模型知道原题。未知 ID、重复答案或不符合题型规则的回答按 input_error；跳过可空，不将缺答当成业务默认值。
- question_id 在同一 change_request.source_id 内跨批唯一；已解决题不再发题，继续补问使用新ID并保留原问题和回答。同一请求重试复用已存响应，不再次处理或增版。
- 多轮补充时，后端保留累计问题及有效回答，prior_questions 含全部已问问题；回答修正由后端保留当前有效答案，不重复同一 ID。仅问未解决的必要事项。
- 模板、生成规则、设计规则与示例为 skill 内置资源，不属于请求参数；用户和调用方都不能通过输入替换。规范变更通过维护 skill 完成。
- generate 仅接受 operation、requirements_input；revise 仅接受 operation、base_version、current_construction_plan、current_application_build_plan、requirements_input、change_request、answers，以及补充回答时的 prior_questions。提供外部模板、规则、示例或其他未定义参数时返回 input_error。
- 对话修改只改变业务需求与相应设计，不修改包内模板及规则；已有业务系统建设方案修改应由后端使用生成该业务系统建设方案时的同一 skill 发布版本。

## 统一响应

仅返回一个 JSON 对象，不加围栏或额外说明。字段均须存在：

```json
{
  "status":"ready",
  "requirements_input":{},
  "construction_plan_markdown":"完整业务系统建设方案 Markdown 正文",
  "application_build_plan":{"forms":[],"approvals":[],"dashboards":[],"pending_items":[]},
  "changes":[],
  "questions":[],
  "message":"本次处理摘要",
  "errors":[]
}
```

- application_build_plan：按 [应用搭建计划输出](application-build-plan.md) 返回独立结构化计划；前端用于展示简短卡片，不从完整正文临时摘要。
- status：ready / unchanged / needs_input / blocked / input_error。
- ready：生成或修改成功，返回完整新有效需求、完整业务系统建设方案和完整应用搭建计划；允许存在明确标注的待确认项，不代表应用可直接执行或发布。正常业务系统建设方案标题为“# 系统名称·业务系统建设方案”。changes 为 {"target":"章节或对象","summary":"实际修改及影响"} 数组；首版可用一项概述生成范围。questions=[]，errors=[]。
- unchanged：无须改动、仅咨询，或用户跳过必要问题后本次请求仍无法完整落实（message 必须明确“本次修改未落实”，不得宣称修改成功）。返回原有效需求、原业务系统建设方案及原应用搭建计划；changes=[]，questions=[]，errors=[]；message 回答咨询或说明无变化。仅用于 revise。
- needs_input：仅用于 revise；本次存在必须由用户决定的歧义、冲突或依赖。先按 slot-policy.md 区分分类：C 类配置细节及其冲突默认不追问，保留待人工确认，不单独触发 needs_input；无法理解修改对象或操作意图时才针对性确认。原有效需求、原业务系统建设方案及原应用搭建计划原样返回，不提交局部修改；changes=[]，errors=[]。questions 每项为 {question_id,type,text,options,allow_explanation,requirement_ids,reason}，题型及校验按 revision-questions.md。本项修改内问题 ID 稳定且不重复，只问未解决事项。后端保存问题和原请求，展示后发起新的 revise。
- blocked：全部有效需求仍无法识别业务意图，无法形成有依据的建设方案；提示重新描述。已有业务意图但审批路径、办理人、权限等未确定不属于 blocked，generate 生成可确定部分并标待确认；revise 按本次修改的必要性决定提问或保留待确认。requirements_input 返回候选完整槽位快照；construction_plan_markdown 为“# 业务系统建设方案生成受阻”及阻塞事项、依据、影响、所需处理。application_build_plan=null，changes=[]、questions=[]、errors=[]。候选结果是失败诊断，后端不得覆盖原正式需求和业务系统建设方案。
- input_error：请求格式、关联或必需字段错误。requirements_input、construction_plan_markdown 和 application_build_plan 为 null；changes=[]、questions=[]，errors 为 {field,message} 数组，message 简述错误。不修改保存状态。

## 保存、版本与确认

版本参数映射：generate 无外部版本输入；后端首次保存 ready 后分配 construction_plan_version=1。revise 的 base_version 是当前已保存 construction_plan_version 的请求别名，不是独立版本计数。只有 ready 且实际内容发生变化时递增；unchanged/needs_input/blocked/input_error 不增版。调用方交付搭建时的 construction_plan_version、application_build_plan 和 requirements_input 从同一确认记录一次读取，不混入旧快照。

1. 后端串行处理同一业务系统建设方案修改，提交前核对 base_version 仍为最新；旧版本请求拒绝提交。通过请求标识去重，重试复用已存结果，不重复增版。
2. 校验槽位 ID 全局唯一，计划的 requirement_ids 全部存在于同版本快照，删除或改名没有悬空引用；用户页面和导出不含内部 ID。仅 ready 通过结构及一致性校验后，将新 requirements_input、业务系统建设方案正文与 application_build_plan 一起保存为一个版本。首次为 1，实际修改由后端递增。流式片段不作为正式结果。
3. unchanged、needs_input、blocked、input_error 都不替换当前正式业务系统建设方案及应用搭建计划。blocked 的诊断和候选需求另存为本次结果；原版本仍保留。
4. 用户确认由后端绑定具体版本。生成或实际修改后的新版本均为待确认，旧版确认不能授权新版搭建。未经确认或仍有阻塞执行的配置，不进入相应搭建操作。
5. 确认动作无需模型调用，由后端绑定当前最新版本并触发搭建编排；此 skill 只负责方案内容，不直接调用搭建接口。对话修改不设三轮上限，不接收其他流程的轮次状态。ready 或 unchanged 后前端恢复输入框并询问是否“按计划搭建”；needs_input 时改为问题卡片。处理或追问期间禁止确认旧卡片，不把未提交修改当成已确认方案。
6. 用户只看业务系统建设方案、修改摘要或问题，不需要看协议字段。文件默认由后端将 construction_plan_markdown 保存为“系统名称-业务系统建设方案.md”；归档位置由调用方指定。无需模型拥有文件系统权限。

旧 JSON 调用方须新增读取和保存 application_build_plan；revise 须传同版本 current_application_build_plan，旧版本没有计划时先完成配套计划生成及校验后再进入本契约的修改流程，不以空对象冒充已有计划。

与旧版迁移：旧生成 skill 直接返回 Markdown；本版统一返回 JSON，调用方须改为读取 construction_plan_markdown，并保存同步返回的 requirements_input，不能把整段 JSON 当作业务系统建设方案文件。

## 参数命名迁移

调用端、响应解析和已保存状态须同步迁移为当前参数名称，不混用旧名称。requirements_input 表示结构化需求输入；construction_plan_markdown 表示建设方案正文；current_construction_plan 表示当前方案正文；construction_plan_version 表示方案版本；blocking_construction_plan 表示是否阻塞方案生成。本契约只使用实际涉及的字段，不因此增加额外请求参数。
