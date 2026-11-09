# chart-create 在应用搭建中的适配

仪表盘创建规范已全部改用用户指定的 skills/chart-create，随包收录于 [chart-create/SKILL.md](chart-create/SKILL.md)，其 [组件规则原文](chart-create/references/ai-app-chart-create.md)、[配置结构](chart-create/references/config-schema.md)、[组件自检](chart-create/references/component-rules.md) 和 [发送前校验](chart-create/references/validation-checklist.md) 按需加载。收录的主文件仅将同级表单、审批链接改指本包适配文件，不修改原目录。

## 本流程的调用边界

- 仪表盘与组件取自已确认应用搭建计划，关联槽位补齐逻辑数据源与统计口径，再使用结构化 task；chart-create 中按旧模板章节读取正文的约定在本流程不适用。
- prepare 保留计划中的仪表盘、组件及其稳定ID，按槽位引用补齐已明确的逻辑数据源、维度、指标和口径。chart-create 提到的空骨架可表示尚无真实数据源的准备状态，本流程不把空骨架当作可自动发布的完整配置；前置未完成返回 waiting。
- generate_config 必须先结束本计划表单、审批阶段；所引用的数据源须实际 published 且已 claim，并有后端获取的真实字段。前置尚在执行或缺 claim/字段调用结果时 waiting；引用源已失败或禁止发布则该仪表盘 blocked，不反问用户猜测数据源，不创建空图冒充完成。
- 不要求模型落盘、输出接入代码或回显多段内容；生成配置时只返回本包调用契约的 JSON。后端负责字段获取、握手、发送及实际发布；组件生成本身不调工具。

## 数据源与字段

后端通过 GET /auto/flow/data/chart/sources/{sourceType}/{sourceId}/fields 获取字段，由该接口在服务端衔接办公侧 CHART_APP source-schema。浏览器及 skill 不直连 /manage/apps/chart/data/source/config，也不再使用本包旧的独立字段接口文档作为调用入口。

提供给模型的 sources[].schema 统一为 {source,fields}，按 chart-create 内组件规则定义；接口外层信封、鉴权和其他返回形状由后端适配，不能让模型猜测 type 与 valueType 的映射。每个 ref 绑定到实际已发布且已 claim 的表单/审批，数字或字符串 ID 比对时统一转字符串，数据源名称取 schema.source.name。

## 两段配置

1. 按 chart-create/references/ai-app-chart-create.md 生成 {name,summary,components,warnings}。只使用真实 fieldId+innerIndex，13 类组件，最多 8 个，遵循维值、聚合、排序和 TopN 规则。
2. 按独立组件分别应用原组件规则，再合并合法结果；某个统计缺字段时不伪造指标，也不抹掉其他独立组件。未实现要求进 todos，诊断进 issues。超过 8 个时按已确认应用搭建计划组件顺序最多生成 8 个合法组件，超额组件逐项列待办，不自行拆新仪表盘、不静默截断后报全部完成。若会导致关键业务误判，则按 [发布控制](capability-policy.md) 禁止发布；所有组件均不可用才 blocked、config=null。
3. 从实际使用的数据源构造 dataSources，每项为 {sourceId,sourceType,sourceName}；sourceId/sourceType 均为字符串，sourceType 仅 "1" 或 "4"。组件的数据源必须命中此数组，不附字段清单或 sourceEnc。
4. 返回最终创建页配置 {name,summary,publishImmediately,dataSources,components}，仅允许这些顶层字段。publishImmediately 必须与外层 publish_allowed 一致：普通待办可为 true，关键待办为 false；本流程按发布控制覆盖 chart-create 中默认立即发布约定。warnings 不进入 config；不要返回 ChartConfig、chartPages 或表单协议字段。
5. 按 chart-create 的校验清单检查。空配置在通用创建入口可以按普通新建处理，但在本自动搭建流程不计 config_ready 或搭建成功。

## 页面衔接

仪表盘配置不写 sessionStorage，不复用表单键。由平台统一入口打开创建页，URL 带 ai=true；校验可信窗口和来源域后等待 ai:ready，再向约定的握手来源发送 {key:'ai-create-chart-config',data:JSON.stringify(config)}，targetOrigin 取经校验的握手 origin，不能用 *。不附 formType/productType，不硬编码域名。

chart-create 的配置结构文件注明接入链路尚未实现，这是该规范记录的实现状态，不等于本次已验证部署能力。后端需确认当前环境支持握手、覆盖、自动发布和回传；未支持则保留 waiting/unknown，不假造仪表盘发布事件或已完成状态。发布成功的消息体尚未在所提供包内定义，不照搬表单的 from/formId。
