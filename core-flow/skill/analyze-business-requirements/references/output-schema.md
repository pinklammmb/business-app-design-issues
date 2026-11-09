# 输入与输出结构

所有调用返回一个 JSON 对象，不附说明或代码围栏。槽位名称保留中文，协议字段使用下述名称。

## requirements

```json
{
  "槽位分析":{
    "审批类型判定":[{
      "id":"s01",
      "定位":"资产使用申请",
      "已知信息":"普通用户可以申请使用资产",
      "状态":"partial",
      "依据":[{"来源":"用户输入","原文":"普通用户可以申请使用资产","推导理由":null}],
      "缺口或冲突":"是否必须审批通过才能使用",
      "是否需澄清":true
    }]
  }
}
```

示例只展开一个槽位；真实 requirements 的槽位分析保留槽位定义中的全部 28 个名称，每项值为数组。尚未适用、未激活的依赖记录为空数组，不建立假想对象。

定位、状态为字符串。已知信息及缺口或冲突为字符串或 null；missing 的已知信息为 null，resolved 的缺口为 null。依据为来源、原文、推导理由的对象数组，直接提取的推导理由为 null；missing 可为空数组。conflict 保留所有有效候选及各自依据。是否需澄清为布尔值，由矩阵计算。快照中的设计补全依据来源可为“AI设计补全”、原文=null、推导理由必填，不冒充用户输入；本阶段不凭空生成这些设计记录。

requirements 顶层仅含“槽位分析”，不包含附件处理状态。initialize 分析 user_input 与 materials 中的有效文本建立快照；submit_answers 使用已存快照，不重新处理附件。requirements 不作为 initialize 的外部输入。

## 响应外层

```json
{
  "status":"ask",
  "state":{
    "requirements":{},
    "rounds_issued":1,
    "items":[],
    "history":[],
    "active_batch":{"batch_id":"b1","round":1,"questions":[]}
  },
  "questions":[],
  "update_log":[],
  "deferred_content":[],
  "end_reason":null,
  "blocking_construction_plan":false,
  "unresolved":[],
  "requirements_input":null,
  "errors":[]
}
```

此处 {}、[] 是结构说明，不是首轮实际结果。requirements 为完整快照，所有字段必须出现。

- status：ask / finished / input_error。
- rounds_issued：已返回的问题批次数，初始化为 0，实际返回一批 ask 时增加 1，最多 3；回答及结束本身不增加轮次。
- items：持续清单。每项为 {item_id, links, gap, progress, depends_on, reason, asked_question_ids}。links 为 {slot, location} 数组；gap 为字符串；progress 枚举 pending / partial / resolved / skipped / unanswered / blocked / obsolete；depends_on 为 item_id 数组；reason 记录初建或增量触发原因；asked_question_ids 为问题 ID 数组。
- history：已提交批次数组，每项为 {batch_id, round, questions, answers}，完整保留问题与回答。
- active_batch：当前待提交批次对象；结束时为 null。其 questions 与响应 questions 完全相同，不同时存在两个有效批次。
- update_log：本次实际变更数组，每项为 {slot, location, action, reason}；action 为 add / update / remove。
- deferred_content：超出本题范围的新增需求，每项为 {question_id, text, reason}，不合并到槽位或清单；仅记录本次处理内容。
- end_reason：ask 时为 null；finished 时为 no_pending / round_limit / no_actionable。no_pending 表示无可问事项；第三轮回答提交并处理后使用 round_limit；no_actionable 仅表示无法识别业务意图、无可据此提出的问题，属于无可问事项的异常结果。
- blocking_construction_plan：ask 时暂为 false，判断以 finished 结果为准；finished 时仅当全部有效上下文无法识别业务意图才为 true；需求细节未定不设为 true。
- unresolved：结束时的未决事项数组，每项为 {item_id, links, gap, blocking_construction_plan, impact}。未要求追问的 B/C 类 missing 不自动加入本清单。
- requirements_input：ask 或 input_error 时为 null；finished 时与 state.requirements 完全一致，输出含稳定记录 ID 的完整槽位快照，见 [贯穿环节的结构化槽位信息](blueprint-input.md)。不另存独立业务摘要，不传问答历史给下游。
- errors：{field, message} 数组。输入错误时 status=input_error，原 state 可用则原样返回，不存在则为 null；questions、update_log、deferred_content、unresolved 为空，end_reason=null，不增轮次、不改需求。无任何有效输入文本时按 input_error 处理。

## questions

每题固定字段：

```json
{
  "question_id":"q1",
  "item_ids":["i1"],
  "links":[{"slot":"审批类型判定","location":"资产使用申请"}],
  "type":"single_choice",
  "text":"资产使用申请是否需要审批通过后才能领用？",
  "options":[
    {"value":"yes","label":"需要"},
    {"value":"no","label":"不需要"},
    {"value":"other","label":"其他"}
  ],
  "allow_explanation":true
}
```

type 枚举 single_choice / multiple_choice / text。text 题的 options 为空；other 或 none（都不是）选项要求填写说明。question_id 跨批唯一，item_id 稳定；同一事项的剩余缺口补问使用新 question_id 并保留原历史。一次 ask 的 questions 必须为 1–5 个独立决策。

## 状态转换检查

- initialize：校验 user_input 字符串与 materials 文本数组，按 analysis-schema.md 建立含全部 28 个槽位的 requirements，再生成清单；不发题则 rounds_issued=0。
- submit_answers：校验 state、batch_id、question_id、选项值和题型，布尔 skipped，不接受与所问题目无关的伪造 ID。单选恰好一个值，多选至少一个值；text 非空；跳过时可空，其他说明按规则必填。遗漏题允许但记 unanswered。
- 有效提交后，将旧 active_batch 和本次 answers 加入 history，再清空旧批次，更新需求与清单；第三轮处理完毕则 finished，否则有可问事项就 ask，无可问事项才 finished。返回第三轮问题时仍为 ask，不能在用户提交前结束。
- finished：questions=[]、active_batch=null、requirements_input 为完整业务结构对象；state.requirements 保留完整槽位快照，未变记录内容不重新生成。汇总须覆盖首轮及全部有效回答，仅排除失效要求；有业务意图则 blocking_construction_plan=false，未决事项保留待确认；仅无可识别业务意图时为 true。
- 任何响应均不得重复问题、恢复失效要求、把默认值当作回答，或超出三轮五题。