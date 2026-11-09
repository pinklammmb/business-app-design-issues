# 研发调用契约

## 运行方式

这是有状态业务流程中的单步模型调用；状态由后端持久化，模型不依赖聊天记忆。调用时向模型提供 SKILL.md 和所需 reference 内容；运行环境支持读取本地 skill 时可直接载入。仅传一个磁盘路径给普通模型 API 不代表它能读到文件。

不要求实现特定 HTTP 地址。后端可以通过现有模型服务调用，采用以下 JSON 协议。

## 1. 解析原始需求并推送第一轮

```json
{
  "operation": "initialize",
  "user_input":"用户原始需求文本，可在附件承载全部需求时为空",
  "materials":["文档附件提取的有效文本"]
}
```

user_input 为字符串，materials 为文档附件提取文本组成的字符串数组，无附件时为 []。只支持文档类上传，后端先提取有效文本再调用模型；不传附件来源、完整性、未读范围或读取问题。模型不打开附件文件。至少 user_input 或一个 materials 元素有非空有效文本；空字符串、null 或元数据对象不作为附件正文。

initialize 在一次调用内先建立 requirements 快照，再创建清单和问题，不再接收独立解析 skill 的输出。快照结构见 analysis-schema.md，必须包含 28 个槽位；随后按 output-schema.md 返回统一响应。没有任何有效输入文本时返回 input_error，不生成澄清问题。

模型返回 status=ask 时，后端原样保存 state 和 questions，再将 questions 推送给前端。状态为 finished 时进入结束处理。status=input_error 时处理输入错误，不展示一批猜测问题。

## 2. 提交第一轮回答并获取第二轮

```json
{
  "operation": "submit_answers",
  "state": "替换为上次响应中的完整 state 对象，不是字符串或摘要",
  "batch_id": "b1",
  "answers": [
    {"question_id":"q1","selected_values":["yes"],"text":null,"skipped":false}
  ]
}
```

state 和 batch_id 必须来自最新保存的响应。answers 是当前批次提交的回答，不传全部聊天历史，不把前端问题标题作为唯一关联依据。选择“其他”时同时提交其 value 和 text；输入框用 text，selected_values 为空；跳过单题用 skipped=true，仅更新该事项进度；是否结束仍按可问事项和轮次判断，不提供结束剩余澄清的请求参数。

模型返回下一轮 status=ask、state.rounds_issued=2、state.active_batch.batch_id=b2 和第二轮 questions；后端保存完整新 state 后向前端推送。这是一次新模型调用，不是模型自行等候用户后续输入。

## 3. 保存与调用顺序

- 响应中的批次号只取 state.active_batch.batch_id；submit_answers 请求中的 batch_id 从这里映射，不读取不存在的响应顶层 batch_id。
- 前端“继续”只保存本批题目的草稿并切换题目，不逐题调用模型；本批最后提交时一次发送 answers。发下一批 ask 才增加 rounds_issued。
- 每个澄清会话只初始化一次。重进页面读取保存的问题，不重新初始化或重新出题。
- 请求只处理一次；后端用业务请求标识去重，重试返回已保存响应，不重复消费答案或推进轮次。请求标识由后端管理，不交给模型猜测。
- 同一会话串行提交，旧 batch_id、未知问题 ID、重复题目答案或已消费批次拒绝处理；不能用旧状态覆盖新状态。允许缺省部分题，其含义见回答规则。
- 轮次按响应中已发问题批次累计，最高三轮。后端为会话状态和预算的权威来源，应校验模型返回轮次及状态，校验失败不推送。
- 第二轮回答再次携带最新 state 和 b2，按相同 submit_answers 操作调用，直到 finished。

## 4. 结束处理

status=finished：取 requirements_input 作为业务系统建设方案生成节点的直接输入，结构见 [业务系统建设方案需求信息](blueprint-input.md)；requirements_input 与 state.requirements 是同份槽位快照；清单和问答历史留给流程持久化与追溯，不要求业务系统建设方案节点重新理解整个澄清过程。

后端检查 requirements_input 与 state.requirements 完全一致，ID 唯一，未决事项未丢失。blocking_construction_plan=false 时进入方案环节；为 true 仅表示无法识别业务意图，提示重新描述。finished 只代表停止当前追问，不代表所有需求明确。

仅在无可问事项，或第三轮回答提交并处理后返回 finished；所有 finished 路径均须返回 requirements_input。首次无需提问时轮次为 0；仍有可问事项且未完成三轮时继续追问。前端不提供主动结束追问的操作；单题跳过不直接结束整轮或整个流程。结束后依据已有信息生成方案，不填造未确定内容。

附件提取与上传失败在模型调用前由上传端处理，结束结果不包含附件读取状态。
## 参数命名迁移

调用端、响应解析和已保存状态须同步迁移为当前参数名称，不混用旧名称。requirements_input 表示结构化需求输入；construction_plan_markdown 表示建设方案正文；current_construction_plan 表示当前方案正文；construction_plan_version 表示方案版本；blocking_construction_plan 表示是否阻塞方案生成。本契约只使用实际涉及的字段，不因此增加额外请求参数。
