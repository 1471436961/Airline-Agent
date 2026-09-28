> **教学环境说明：** 下文保留上游研究框架资料。实际命令以项目根目录 README 和 materials/CLASSROOM.md 为准。本项目没有 run_local_test 或 Client 接口申请工具；使用本地单元测试及远程 t1/p1/t2。当前 OpenAPI 中的退货／取消能力已经启用。

学习者试点覆盖说明： 请先阅读项目根目录中的 README。 workspace/ 表示 agent/ 位于你的代码仓库中。本试点提供远程 t1/t2 提交功能，而非上游的 run_local_test 工具。除非明确选择子集，t2 会运行所选场景中的所有案例；这并非完整的多领域基准测试或上游额度预算评分。允许使用的模型由导出的部署清单和讲师服务设定。


构建客服智能体：airline_plus

你的任务是根据此工具包中的领域材料，构建一个稳健的客服智能体。具体材料集合因领域而异；请通过文件树和文件级文档了解结构，然后将这些材料转化为可运行的实现。

成功的标准

智能体质量以通过的评估案例比例来衡量。评估案例不会提前公开，可能涵盖所提供材料中的全部行为范围：常规请求、复杂的多步骤或多意图请求、少见的边界情况、不完整或变化中的信息，以及必须拒绝或转向其他渠道处理的请求。

只有当智能体正确处理底层操作、使系统处于正确状态、遵守领域规则，并向客户传达正确信息时，案例才算成功。在满足框架契约的前提下，不要求采用特定架构或开发流程。

如何使用此工具包

将工具包目录视为可用材料清单。来源信息可能分散在政策文档、API 契约、知识库文件、培训记录或其他客户提供的材料中。请阅读相关文件头、目录索引和文件名，以了解各材料的内容。

workspace/ 是实现区域。 framework/ 说明你的实现必须满足的契约。 simulations/ 存储仅运行候选实现的本地模拟所生成的产物。

必需的输出

评估器将这些文件导入为稳定入口点。你可以添加辅助模块、检索层、规划器、验证场景或所需的任何其他支撑架构；以下文件只是集成接口。



workspace/tools.py ——一个 ClientAPIToolKitBase 子类，包含使用 @is_tool装饰的智能体操作，用于实现客户提供材料中描述的全部操作。每个方法使用注入的 self.client_api。参见 framework/client_api_contract.md 和 client_api/openapi.yaml.




workspace/agent.py ——智能体实现。仅定义接口的脚手架指定了运行时入口点；请实现你希望接受评估的智能体逻辑。它可以读取任何提供的工具包材料，也可以在 workspace/ 下按任意组织方式添加辅助、提示词、规则、索引或其他证据文件。参见 framework/agent_contract.md.




模拟环境

你可以使用模拟环境，创建模拟客户场景，并让这些客户与你的客服智能体互动。未提供示例场景；请根据客户提供材料中的场景编写自己的 JSON 文件。

参见 framework/scenario_contract.md 了解客户场景格式。

要运行仅针对候选实现的端到端模拟，请调用 run_local_test 工具并传入场景路径，例如 run_local_test(task_path="workspace/my_customer_scenario.json")。该工具只会将你编写的场景文件运行在你自己的助手工具包上。如果领域包含客户侧运行时，客户／用户工具调用会通过所提供的客户侧运行时路由。每次运行都会在 simulations/ 下写入带时间戳的 JSON 产物，方便你以后检查历史对话和奖励。对于黑盒行为检查，请使用自然语言断言并检查返回的对话记录，而不要依赖内部函数名。

性能要求


agent_credit_budget: gpt-5.6-sol, anthropic/claude-opus-5, google/gemini-3.1-pro-preview, moonshotai/kimi-k3, deepseek/deepseek-v4-flash, gpt-5.6-terra, gpt-5.6-luna, anthropic/claude-haiku-4-5, google/gemini-3-flash-preview, qwen/qwen3.8-27b, google/gemma-4-31b-it, gpt-5.4-nano, qwen/qwen3-30b-a3b-instruct-2507, google/gemma-4-26b-a4b-it, gpt-4.1-mini, moonshotai/kimi-k2.6, gpt-4o-mini, anthropic/claude-sonnet-5, gpt-5-mini, google/gemini-3.1-flash-lite 共用每次对话 0.0610 额度的预算。通过模型网关对这些模型进行的每次调用，其全部输入和输出 token 都计入预算；推理 token 计为输出。


使用 run_local_test 在迭代时检查实测性能。超出额度采用软惩罚：最终得分为平均任务奖励减去每次对话超出预算比例的平均值，最低为零。延迟要求仍属于硬性门槛。

重要


允许的智能体模型及其推理约束由 framework/deployment_manifest.json固定。你的实现可以从中选择。

开发者编写的场景是本地探测用例，并不代表最终评估分布。最终评估取决于在更广泛、未见过的客户请求集合上的表现。