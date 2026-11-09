# 后端运行衔接

## 固定顺序

1. 后端绑定用户确认版本，传 application_build_plan、同版 requirements_input 及人工待办调用 prepare；保存转换后的 tasks 与 issues，ref 沿用计划应用 id。
2. 表单与审批按任务依赖调度，先打开空白创建页，前端实际加载就绪记为 page_ready；此时尚未要求创建应用或取得应用ID。随后调用 generate_config，后端保存返回 config 为应用配置JSON文件。
3. config_ready 后，后端校验配置、publish_allowed 与 todos；表单/审批创建页读取应用配置JSON文件并在页内回显，再完成创建，允许发布时自动发布并回传应用ID。关键缺项时仅回显并补齐，不发布或启用。具体文件读取通道见 [创建页流程](form-approval-entry.md)，不能将旧sessionStorage流程当成已实现的文件读取接口。
4. 表单、审批实际发布完成后，后端记录该 ref 的 published 状态及真实结果。本计划表单、审批全部进入 published / failed / blocked 终态后开始仪表盘阶段；仅实际引用的源应用必须已发布。非依赖应用失败不拖住独立仪表盘，必需源失败或禁止发布则该仪表盘 blocked 并列待办；前置仍执行时才等待。
5. 后端确认仪表盘所引用的表单、审批已 claim，按 [chart-create 适配](dashboard-adapter.md) 调用 GET /auto/flow/data/chart/sources/{sourceType}/{sourceId}/fields，验证业务成功并适配成 {source,fields} 后提供给模型。办公侧鉴权与签名留在服务端，不直连平台 source/config，不向模型传密钥。
6. 仪表盘亦先创建空白应用，再用任务和真实字段调用 generate_config。最终配置按 chart-create 包含 dataSources 与对应发布许可的 publishImmediately、不含 warnings；不写 sessionStorage。后端及 SPA 在 ai:ready 握手后通过 ai-create-chart-config 消息发送 JSON 字符串，按校验来源 origin 通信，交创建页转换和覆盖；仅允许发布时自动发布，关键缺项 false 时留待补齐。转换后的 ChartConfig 由创建页负责，模型不生成。仪表盘发布回传消息体未明确、接入未落地时保留未完成，不虚构 published 消息。

## 成功回传

表单和审批自动发布成功消息形状如下：

```json
{"action":"published","from":"form","data":{"formId":171354}}
```

审批 from 为 aprv。父页接收后需核对可信来源窗口、来源域及当前任务类型，避免其他应用消息串入；当前消息不含 task ref，后端须绑定当前正在发布的 ref。表单、审批同一通道串行处理，不能同时覆盖固定存储键。仪表盘按单独的 ai:ready / ai-create-chart-config 消息握手，不套用表单存储协议。

表单/审批打开创建页时 blank_app_id 可为 null；回传 formId 后保存 published_id，不虚构预创建ID。仪表盘如存在 blank_app_id，仍与发布ID分开记录，不能假定两者相同。sourceType 按 form=4、approval=1；source_id 由后端依据平台映射确认，再用 auto-flow 字段接口验证，不让模型把未知空白应用 ID 当作数据源 ID。

自动发布失败保留创建页，现协议没有失败消息。无回传只能记等待或待核实，超时不证明失败或成功；不得自动重复点击发布。重试前查询实际结果，表单/审批保留原创建页与JSON文件，先核验是否已实际创建，避免重复创建；仪表盘复用已存在的空白应用和配置。请求去重及重试策略由后端实现，本 skill 不自行循环重试。

## 运行记录与汇总

后端 runtime 每项固定为 {ref, status, blank_app_id, published_id, source_id, source_type, claimed, detail}，未知值为 null。status 为 pending / page_ready / blank_created / configured / published / failed / blocked / unknown。page_ready 用于表单/审批空白创建页已加载；blank_created 保留给仪表盘既有空白应用流程。只有平台真实发布证据支持 published。claimed 为 true / false / null，由后端依据 claim 实际结果填写，不由 AI 推断；仪表盘 source_id、source_type、claimed 为 null。

summarize 接收任务、config_results、runtime 和可选 manual_todos，不修改实际状态。返回每项 {ref,name,type,status,published_id,detail}，以及聚合去重的 todos 和 issues。按 [部分实现与交付](capability-policy.md) 判断：全部任务发布且待办及阻塞均解决为 complete；存在实际已发布或已应用的部分成果但仍有待办/未发布项为 partial；无实际应用成果为 incomplete。已发布应用的普通待办仍须保留，计划外人工事项由 manual_todos 补充；未提供时保留范围限制。

模型输入 plan.status=prepared、配置输出 config_results[].status=config_ready、实际运行 runtime[].status=published 属于不同阶段，不可互相复制。

表单/审批 configured 只说明JSON在页内回显，没有实际应用创建证据时不算应用成果，不据此判 partial。

config_ready 不等于 published；缺失 runtime 记录按 pending 保留，后端应补齐。发布证据或 ID 不完整却标 published 时记录问题，不报 complete。无任务时 incomplete 并说明无可执行对象。配置已应用但存在关键缺项时，runtime 记 blocked、detail 写清已配置及禁止发布原因，不虚报 published；仅空白创建页、回显预览或空白壳本身均不算部分成果。搭建开始时锁定本次执行版本；后续修改是新版本，不能热替换正在执行的plan，也不把旧配置和旧真实应用ID自动套到新版。本次创建会话搭建完成后，不再通过对话新增或修改需求、计划及方案，不恢复普通输入框或追问卡片；前述版本隔离不表示允许完成后继续对话。是否对已创建应用执行更新需另有平台更新契约，本流程不擅自重建或覆盖。

## 搭建完成后的交互

后端将本次创建会话保存为搭建已完成，前端持续禁用对话输入框、发送按钮和对话修改入口，提示“搭建已完成，无法继续通过对话修改”。刷新或重新打开时仍按已保存状态禁用，后端同时拒绝该会话的新修改请求；不重新调用方案修改或需求澄清。查看结果、下载建设方案及进入业务系统的入口保留。完成状态由后端依据实际运行记录判定，不以模型生成配置或页面回显代替。此限制由会话状态和调用入口执行，不新增模型响应字段。

交付页面的建设方案下载读取同版后端归档；应用访问地址与业务系统跳转入口由后端从平台真实返回值组装，不由模型推导。没有可核验入口时按钮不可用，不宣称整个业务系统已可访问；模型汇总只评价实际应用成果。
