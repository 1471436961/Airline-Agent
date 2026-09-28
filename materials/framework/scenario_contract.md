> **教学环境说明：** 下文保留上游研究框架资料。实际命令以项目根目录 README 和 materials/CLASSROOM.md 为准。本项目没有 run_local_test 或 Client 接口申请工具；使用本地单元测试及远程 t1/p1/t2。当前 OpenAPI 中的退货／取消能力已经启用。

客户场景契约

本文说明可用于客户模拟的 JSON 文件格式。场景并非单元测试，而是对客户带着某种处境、目标和已知事实前来寻求帮助的描述。模拟器会依据该场景与你的智能体对话。

通过调用构建环境中的 run_local_test 工具运行场景：

run_local_test(task_path="workspace/my_customer_scenario.json")


这只会将你提供的场景文件或目录运行在你提交的助手工具包上。对于客户 API 任务，本地测试使用实现了 client_api/openapi.yaml的沙盒服务。只有已记录的 REST 接口及其响应属于开发者契约；本地测试行为不会扩展或覆盖该契约。不会加载保留场景。

开发者编写的场景用于本地行为探测。它们不会成为最终评估套件，也不会定义最终评估的请求分布；仅通过这些场景不足以证明智能体的整体质量。最终质量将在更广泛、未见过的评估案例上衡量。

最小场景

{
  "id": "my_customer_scenario_001",
  "user_scenario": {
    "persona": "A concise customer who answers follow-up questions directly.",
    "instructions": "I am calling because I need help with <situation>. I know <facts the customer knows>. I want the agent to <customer goal>."
  },
  "evaluation_criteria": {
    "nl_assertions": [
      "The agent resolved the customer's request or clearly explained why it could not be resolved.",
      "The agent followed the domain policy in the provided materials."
    ],
    "reward_basis": ["NL_ASSERTION"]
  }
}


设置 reward_basis 需显式设置。如果省略，后端默认值为
["DB", "COMMUNICATE"]，因此自然语言断言仍会被评估，但不会影响所报告的奖励。

后端生命周期

当 run_local_test 运行场景时，后端会执行以下操作：


将 JSON 文件加载到 Task 数据模型中。

加载 workspace/tools.py, workspace/agent.py，以及工具包中存在的任何其他适用工作区文件。

安装可用操作、通用工具包资源访问能力、受约束的模型网关和运行时配置，然后调用 create_agent()。工厂可通过以下方式读取这些设施： get_agent_context().

使用标准本地场景运行时创建模拟客户。

如果领域包含客户侧运行时，则通过所提供的客户侧运行时路由客户侧工具调用。

转换 user_scenario 使用以下方法转换为文本： str(task.user_scenario) ，并将该文本作为模拟客户的私有指令传入。

如果场景包含 user_tools，则将可用的客户侧工具筛选为该列表中的工具。如果省略 user_tools ，则在领域提供客户侧工具时，使用其默认工具。

在客户、智能体和环境之间开展逐轮对话。

依据以下配置评估产生的对话记录及最终环境状态：
evaluation_criteria.reward_basis.

将带时间戳的产物写入 simulations/ ，其中包含对话记录、工具调用、奖励明细，以及构建测试框架可访问的序列化结果数据。


模拟路径允许你使用与提交后相同形态的客户／用户运行时来运行自己的场景，而无需自行实现客户侧的电话或设备工具。

对于 REST 工具包，每个场景必须且只能选择一种客户 API 模式。如果省略
client_api 字段，模式为 seeded：场景会从以下位置所列的合成记录的全新副本开始：
client_api/development_seed.json。这些公开标识符特意保持稳定，以便进行本地测试，它们不属于最终评估。记录使用与该领域常规数据相同的标识符约定和资源结构。某些领域还列出具名的 fixtures ，用于将本地客户／设备运行时置于已记录的状态。

一个 mock 模式的场景具有以下形式：

{
  "id": "declining_service_001",
  "client_api": {
    "mode": "mock",
    "module": "workspace/mock_client_api.py",
    "config": {"account_id": "acct_test", "failures_before_success": 1}
  },
  "user_scenario": {
    "instructions": "I need help with account acct_test."
  },
  "evaluation_criteria": {
    "nl_assertions": ["The agent handled the changing service response."],
    "reward_basis": ["NL_ASSERTION"]
  }
}


模块仅在密封候选沙盒内运行，并且必须定义
create_mock_client_api(config)。每个全新的场景实例都会调用一次该工厂。它返回一个可调用对象，或包含
request(payload)的对象，其中 payload 仅包含公开的 method, path,
query, body，以及 headers。返回常规客户 API 响应封装： status_code, body，以及可选的 headers 和
elapsed_seconds.

class MockClientAPI:
    def __init__(self, config):
        self.calls = 0
        self.failures_before_success = config["failures_before_success"]

    def request(self, request):
        self.calls += 1
        if request["path"] == "/v1/example":
            status = 503 if self.calls <= self.failures_before_success else 200
            return {"status_code": status, "body": {"call": self.calls}}
        return {
            "status_code": 404,
            "body": {"error": {"message": "Not mocked"}},
        }

    def verify(self):
        assert self.calls >= 2, "expected the example operation to be retried"


def create_mock_client_api(config):
    return MockClientAPI(config)


状态属于该返回对象，因此相同请求可能随时间产生不同响应。可选的 verify() 钩子会在对话后运行；抛出异常或断言失败即可使本地场景失败。带时间戳的模拟产物会记录每次模拟请求和响应，以及验证结果。回调异常会报告为本地测试失败。

模拟代码与其他候选运行时代码受相同边界约束：离线、工作区只读、时间限制、请求数量限制、JSON 封装和载荷大小限制。它不能调用或检查真实客户环境。测试框架有意不依据 client_api/openapi.yaml验证模拟操作的数据结构；开发者需自行负责确保模拟能够代表可能出现的客户侧响应。

seeded 和 mock 互斥。模拟场景不能选择
development_fixture ，也不能使用 DB 评分，因为没有真实客户数据库可供比较。应显式将 reward_basis 设置为对话记录、响应、操作或环境断言。如果领域有客户侧模拟工具，它们仍由宿主提供，但其私有运行时不会与模拟回调共享；客户所需的任何模拟客户侧状态都应在场景指令中描述。

基于模拟后端的本地测试运行在当前构建运行时镜像上。这不会更改任务为提交后或保留评估对话所选择的版本化镜像。

客户指令

user_scenario 是提供给模拟客户的信息，而不是提供给你的智能体。智能体只能看到客户在对话中说出的内容。

user_scenario.persona：可选。客户的一般沟通风格或背景。应将其与具体任务处境分开。

user_scenario.instructions：客户的处境、已知信息以及想要达成的目标。可以是纯字符串，也可以是包含以下字段的结构化对象：

{
  "domain": "example_support",
  "reason_for_call": "Why the customer contacted support.",
  "known_info": "Facts the customer knows and may provide.",
  "unknown_info": "Facts the customer does not know and should not invent.",
  "task_instructions": "What the customer is trying to accomplish."
}


结构化形式会呈现为带标签的文本分节。 known_info 应包含客户可以透露的事实。 unknown_info 应包含客户不应编造的事实；如果智能体询问，客户应表示自己不知道，或询问如何查找。

奖励依据

evaluation_criteria.reward_basis 控制哪些检查影响数值奖励。可用值包括：




值
检查内容
典型本地用途





NL_ASSERTION
使用 LLM 评判器判断对话记录是否满足其中的每条字符串： evaluation_criteria.nl_assertions.
是客户模拟的最佳默认选项，因为它检查行为，而不要求精确的工具名称。



RESPONSE_ASSERTION
检查其中确定性的助手回复措辞约束： evaluation_criteria.response_assertions.
适用于禁用词或短语出现次数上限等精确风格约束。



COMMUNICATE
检查其中的每条字符串 evaluation_criteria.communicate_info 是否出现在助手文本回复中。
适用于简单、精确的沟通检查；它基于子字符串匹配，灵活性低于 NL_ASSERTION.



ACTION
检查对话记录中的工具调用是否与 evaluation_criteria.actions 在名称和所选参数上匹配。
适用于明确要求特定工具接口的情况。它可能过度限制其他实现方式。



DB
将最终环境状态与根据以下内容推导的预期状态比较： evaluation_criteria.actions.
当你可以通过操作指定预期状态时，适合进行精确的状态变更检查。如果未提供操作或环境断言，就没有有意义的状态检查。



ENV_ASSERTION
调用以下位置列出的函数： evaluation_criteria.env_assertions ，在环境上执行，并将其布尔结果与 assert_value.
比较。仅在环境中存在相关断言辅助函数时适用。




多项奖励依据会相乘。若任一纳入的检查得到 0，整体奖励即为 0。

评估标准字段

nl_assertions：对对话应满足条件的自然语言陈述。只有在以下情况下才计入奖励： reward_basis 包括
"NL_ASSERTION".

response_assertions：针对助手面向客户消息的确定性断言。只有在以下情况下才计入奖励： reward_basis 包括
"RESPONSE_ASSERTION".

communicate_info：应出现在助手消息中的精确字符串或事实。只有在以下情况下才计入奖励： reward_basis 包括
"COMMUNICATE".

actions：预期的助手或客户工具调用。每项操作包含：

{
  "action_id": "unique_action_name",
  "requestor": "assistant",
  "name": "tool_name",
  "arguments": {"arg_name": "arg_value"},
  "compare_args": ["arg_name"]
}


requestor 可以是 "assistant" 或 "user". compare_args 控制要比较的参数。如果省略 compare_args ，则检查提供的所有参数。

env_assertions：环境函数检查。每条断言包含：

{
  "env_type": "assistant",
  "func_name": "assertion_or_tool_function_name",
  "arguments": {},
  "assert_value": true,
  "message": "Optional failure message."
}


env_type 可以是 "assistant" 或 "user"。仅当目标函数存在于相关环境中时使用。

可选场景字段

description：可选，供自己参考的场景目的、相关政策或特殊边界情况说明。若提供，必须是 JSON 对象；可使用如下字段： purpose 和 notes。例如：

{
  "purpose": "Exercise a repeat caller asking about a pending request.",
  "notes": "REQ-2041 is created and left pending during setup."
}


initial_state：可选，对话或消息历史的初始化设置。大多数本地场景不需要它。在 REST 工具包中，可以包含：

{
  "development_fixture": ["service_paused", "alerts_muted"],
  "initialization_actions": [
    {
      "env_type": "assistant",
      "func_name": "tool_or_setup_function",
      "arguments": {}
    }
  ],
  "message_history": []
}


在 REST 模式下，开发者编写的场景不能使用 initialization_data：它描述的是私有客户存储，而非公开 API。也不能使用用户初始化操作。助手侧 initialization_actions 是允许的，但其名称仅会解析到开发者自己的助手工具，位置为 workspace/tools.py。这些操作会在安装全新客户 API 上下文后运行，因此包装工具可以通过常规的、已记录的 REST 调用来准备案例。它们不会解析到私有客户函数。

在种子数据模式下， development_fixture 可以是该领域在以下位置公开的测试夹具 ID： client_api/development_seed.json，也可以是这些 ID 的列表，例如
"service_paused" 或 ["service_paused", "autopay_off", "alerts_muted"]。它是仅用于本地测试的选择器，用于选择已记录、由宿主管理的状态；不是数据库载荷，也不是私有函数调用。列表按顺序应用，因此如果两个夹具修改相同设置，后面的夹具会覆盖前面的；每个夹具最多列出一次。未知夹具 ID 会被拒绝。当领域没有列出夹具，或默认连接状态已经合适时，请省略该字段。

message_history 会预加载现有对话。旧版非 REST 工具包保留其原有基于数据库的初始化行为。

user_tools：可选，提供给该场景用户模拟器的客户侧工具名称列表。省略此字段可让客户侧运行时选择默认工具。只有希望模拟客户完全通过文本交流时，才使用空列表。

required_documents：可选，在知识密集型领域中预计相关的文档标题列表。这是供分析和调试使用的元数据；不会自动为智能体检索文档。