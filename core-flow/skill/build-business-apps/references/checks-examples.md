# 检查与行为示例

## 输出前检查

- prepare 只转换已确认应用搭建计划；同版槽位引用有效，任务 ref 沿用计划应用ID，审批不重复建普通表单，无任务或有问题均未虚报完成。
- 字段、审批、统计要求未丢失；名称、ref 与 source_ref 一致，依赖存在；未提到的业务不新增。
- 表单与审批满足当前接入协议，publishImmediately 是与 publish_allowed 一致的布尔值；无自造字段键、人员或角色，无关键能力静默降级。
- 仪表盘在表单审批阶段结束后生成，实际引用的源应用已发布，字段来自对应成功接口结果；源应用已 claim，ID 和类型转换符合 chart-create，字段身份用 fieldId+innerIndex，不以 label 代替。
- 单仪表盘最多 8 个组件，组件类型、维度、聚合、排序、TopN 均符合 chart-create；超过 8 个时只生成支持的最多 8 个，超额组件逐项保留待办，不报全量完成、不自行拆新仪表盘；关键影响则禁止发布。
- 单组件不能跨数据源直接连接，过滤或计算要求无协议表达时保留问题，不私造 filter/公式字段。图表可适配展示形式，但不得用明细表冒充明确统计结果。
- 表单/审批先打开空白创建页，不要求预创建应用ID；读取JSON文件并回显后完成创建，允许时自动发布并回传ID。
- 配置就绪与实际发布分开；不把空字段列表当作无数据记录，不编造仪表盘发布消息。
- 仪表盘最终 config 含 dataSources、与发布许可一致的 publishImmediately，不含 warnings；消息 key 为 ai-create-chart-config，在 ai:ready 后发字符串，不使用 sessionStorage。
- JSON 严格可解析，config 不含过程元数据；本次不生成动画、页面代码或其他文件。

## 示例：普通表单与依赖仪表盘

已确认计划中含表单 app_1“资产台账”（字段：资产名称，单行文本）和仪表盘 app_2“资产概览”（组件：资产记录数）。组件关联槽位 s03 已明确以资产台账统计记录次数，s01、s02 分别记录字段名和类型。此例仅演示结构，不是默认业务要求。

prepare 的任务片段：

```json
[
  {"ref":"app_1","type":"form","name":"资产台账","requirement_ids":["s01","s02"],"requirements":{"fields":[{"名称":"资产名称","类型":"单行文本"}]},"depends_on":[]},
  {"ref":"app_2","type":"dashboard","name":"资产概览","requirement_ids":["s03"],"requirements":{"components":[{"名称":"资产记录数","source_ref":"app_1","指标":"记录次数"}]},"depends_on":["app_1"]}
]
```

app_1 的表单空白创建页已打开并就绪（page_ready）、无其他缺口时生成以下config；后端保存为JSON文件，由创建页读取回显、完成创建后自动发布并回传ID：

```json
{"name":"资产台账","formConfig":[{"alias":"资产名称","compt":"editinput","label":"资产名称"}],"publishImmediately":true}
```

仅 config_ready 或 configured 时，app_2 必须 waiting；app_1 实际 published、已 claim 且 auto-flow 接口返回对应真实字段后，才可生成 app_2 配置。本例无缺项，仪表盘最终配置须含 dataSources、布尔 publishImmediately=true，组件生成结果的 warnings 转外层 issues。

假设后端确已返回资产台账 sourceId=171354、sourceType=4，以及文本字段“资产名称”fieldId=1001、innerIndex=0，最终仪表盘 config 示例为（ID 仅为本例假设，不用于实际调用）：

```json
{
  "name":"资产概览",
  "summary":"统计资产登记记录次数",
  "publishImmediately":true,
  "dataSources":[{"sourceId":"171354","sourceType":"4","sourceName":"资产台账"}],
  "components":[{
    "componentType":"zhibiaoCard",
    "title":"资产记录数",
    "sourceId":"171354",
    "sourceType":"4",
    "metrics":[{"fieldId":"1001","innerIndex":0,"name":"资产名称","aggregateType":"次数"}]
  }]
}
```

此配置在可信 ai:ready 握手完成后以 ai-create-chart-config 消息发送，data 为 JSON 字符串；warnings 若存在只保留到外层 issues。只有实际发布结果才能使仪表盘任务变为 published。

## 示例：原文要求协议无法表达

字段明确必填且枚举“请假、出差”，选项还决定审批路径。prepare 完整保留要求；generate_config 能生成合法的已支持部分时返回 config_ready、publish_allowed=false，配置内 publishImmediately=false，并将选项及必填等关键缺项写入 todos。不可生成合法审批结构时才 blocked、config=null；不能发布改变审批路径的替代方案。

## 示例：审批角色与发布

“管理员审批”没有真实角色映射：列关键待办，publish_allowed=false，不把管理员字符串当作已核实角色；若唯一必需审批节点因缺真实身份而无法构成合法配置，则 blocked、config=null。明确映射 verified=true 且其他要求均可表达时，按角色节点生成 JSON 字符串 roleList；发起节点不加入 aprvUser 列表。

后端收到 action=published、from=aprv 和真实 formId，且确认对应当前窗口任务后，才将该审批任务设为 published。无消息时 unknown/waiting，不凭配置成功推断已发布。

## 示例：失败与结果

两张表单中一张已发布、一张发布失败，表单审批阶段结束后，若仪表盘只引用成功且已认领的一张，可继续生成；若引用失败的一张，记录依赖阻塞待办，不无限等待。汇总 overall=partial，保留成功 ID 和未完成原因，不重新创建成功表单。

仪表盘要求“通过率”，接口只有“审批结果”而无可直接绑定的指标字段：按 chart-create 返回不可满足的诊断，不自己计算比率，不用记录次数代替通过率。

## 发布许可与待办示例

下面是假设任务“资产台账”的局部响应示例；实际版本、标识和依据由调用方提供。

普通缺项：说明字段期望默认展示“未填写”，不参与校验、审批、计算或权限，当前协议无法设置。已支持字段照常生成并允许发布，普通待办在交付中保留：

```json
{
  "status":"config_ready","construction_plan_version":1,"ref":"app_1",
  "config":{"name":"资产台账","formConfig":[{"alias":"说明","compt":"textarea","label":"说明"}],"publishImmediately":true},
  "publish_allowed":true,
  "todos":[{"target":"app_1 / 说明","requirement":"默认展示未填写","reason":"当前字段协议未定义默认值参数","impact":"不影响现有业务办理，仅缺少默认展示文字","follow_up":"在平台对应字段配置入口补齐默认值","blocks_publish":false}],
  "issues":[],"errors":[]
}
```

关键缺项：库存数量必须满足业务规定的非负校验，但当前协议无法传该校验。可生成数字字段供后续补齐，不能发布或启用：

```json
{
  "status":"config_ready","construction_plan_version":1,"ref":"app_1",
  "config":{"name":"库存台账","formConfig":[{"alias":"库存数量","compt":"number","label":"库存数量"}],"publishImmediately":false},
  "publish_allowed":false,
  "todos":[{"target":"app_1 / 库存数量","requirement":"库存数量不得小于零","reason":"当前字段协议未定义校验配置","impact":"不能保证库存数据正确","follow_up":"在平台对应校验配置入口设置非负校验并核验后再发布","blocks_publish":true}],
  "issues":[],"errors":[]
}
```

检查：配置都合法且可供初始化，但第二份配置不能进入发布；两者均不能把原要求从业务系统建设方案删掉。其他独立应用继续，交付保留完整业务系统建设方案及待办。所有应用已发布但仍有普通待办时整体仍为 partial，不能记 complete。
