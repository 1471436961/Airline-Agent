智能体契约

你的 agent.py 文件必须导出一个工厂函数，供评估器调用以构建内循环智能体。


工厂函数

from tau2.hyper.agent_context import get_agent_context

def create_agent():
    """
    Returns:
        An agent instance with get_init_state() and generate_next_message()
    """
    context = get_agent_context()
    ...  # Any agent logic goes here.


get_agent_context() 在工厂运行时可用，并返回四项运行时设施：


action_interface：完整操作目录、其元数据，以及按规范名称选择操作的辅助函数。目录不强制任何分组或排序。

resources：工具包根目录、相对文件清单，以及解析或读取任何所提供材料的辅助函数。

model_gateway：访问允许使用的模型及其强制约束。

runtime_config：领域名称等运行时元数据。


这些输入是能力和资源，而非必须遵循的组织形式。工厂自行决定如何使用，以及是否使用它们。

模型网关是生成的智能体代码所支持的推理路径。每次推理调用都必须显式指定模型。如果允许列表包含该模型的多种配置，请传入足够的受约束参数，以唯一确定其中一种；网关会拒绝含糊、不被允许或存在冲突的请求。如果在以下步骤之后仍需推理，请在返回的智能体或其组件中保留网关对象： create_agent() 返回。

取值为 {"one_of": [...]} 的约束，表示可由你自行选择，而不是固定设置。例如：

{"model": "gpt-5.6-sol", "constraints": {"reasoning_effort": {"one_of": ["high", "medium"]}}}


允许调用传入 reasoning_effort="high" 或 "medium" ，并拒绝任何其他值。若调用省略固定约束，系统会自动提供该值；但可选约束没有默认值，省略会导致错误，因此每次调用都要传入一个选项。你可以在不同调用之间变更选择。


智能体接口

你的智能体必须实现两个方法：

get_init_state(message_history=None) -> state

在对话开始时调用一次。返回一个不透明状态对象，后续每一轮都会传递该对象。

generate_next_message(message, state) -> (AssistantMessage, state)

每轮调用一次。接收 UserMessage （来自客户的文本）或 MultiToolMessage （智能体上一轮发起的工具调用结果）。返回智能体回复和更新后的状态。

该 AssistantMessage 可以包含：


文本内容 ——发给客户的消息

工具调用 ——要调用的一个或多个工具


智能体应当回复文本或执行工具调用，二选一，不能同时进行。

可选钩子

运行时还识别三个可选方法；若智能体未提供，则使用框架默认实现：


is_stop(message) -> bool ——助手消息是否结束对话。默认：智能体从不发出停止信号；对话由客户侧结束，或在达到轮数限制时结束。

set_seed(seed: int) ——为内部随机性设置种子。默认：不执行任何操作。

stop(message, state) ——对话结束后的清理。默认：不执行任何操作。