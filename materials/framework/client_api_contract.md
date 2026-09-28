> **教学环境说明：** 下文保留上游研究框架资料。实际命令以项目根目录 README 和 materials/CLASSROOM.md 为准。本项目没有 run_local_test 或 Client 接口申请工具；使用本地单元测试及远程 t1/p1/t2。当前 OpenAPI 中的退货／取消能力已经启用。

客户 API 契约

将工具实现为 ClientAPIToolKitBase 的子类，放在
workspace/tools.py中。运行时构造工具包时会传入一个 ClientAPI，可通过 self.client_api访问。使用 @is_tool 装饰的方法会向客服智能体公开，并可发起一个或多个客户 API 请求。

from urllib.parse import quote

from tau2.environment.toolkit import ToolType, is_tool
from tau2.hyper.client_api import ClientAPIToolKitBase


class Tools(ClientAPIToolKitBase):
    @is_tool(ToolType.READ)
    def get_order(self, order_id: str) -> dict:
        """Get an order by its customer-facing identifier."""
        response = self.client_api.request(
            "GET",
            f"/v1/orders/{quote(order_id, safe='')}",
        )
        response.raise_for_status()
        return response.body


REST 接口定义于 client_api/openapi.yaml。响应包含
status_code, body, headers，以及 elapsed_seconds。请显式检查状态和正文； raise_for_status() 可在非 2xx 响应应转化为工具错误时使用。

对话上下文

self.client_api.context 包含当前对话的可信只读上下文。目前仅公开 conversation_id。对于 OpenAPI 路径包含 {conversation_id}的操作，应使用此值；与其他路径标识符一样，需进行 URL 编码。不要要求客户或模型提供它。

例如，实时转接工具访问客户侧对话资源：

class Tools(ClientAPIToolKitBase):
    @is_tool(ToolType.GENERIC)
    def transfer_to_human_agents(self, summary: str) -> dict:
        conversation_id = quote(
            self.client_api.context.conversation_id,
            safe="",
        )
        response = self.client_api.request(
            "POST",
            f"/v1/conversations/{conversation_id}/transfers",
            body={"summary": summary},
        )
        response.raise_for_status()
        return response.body


客户侧会将经过认证的对话记录和路由上下文关联到转接。智能体只需提供问题摘要。成功响应表示实时转接已被接受；自动重试并不安全，而且此后该对话的客户侧操作会被拒绝。

OpenAPI 文档是请求与响应数据结构的规范依据。尤其需要注意：


路径中的资源标识符必须进行 URL 编码。这对于包含保留字符的标识符尤为重要——开头的 #
会编码为 %23.

enum 值已穷尽所有可用选项。不要创造或归一化出额外值。

字段接受 JSON null 的前提是其模式包含 null 分支。

每个操作都会通过相应元数据说明：是否修改状态、重复调用是否安全、是否允许自动重试、一致性行为，以及是否分页： x-api-* 字段。

错误响应使用共享的 APIError 封装。 400 表示请求不符合已记录的路径、查询参数或正文模式； 404
表示未找到所引用的资源； 409 表示资源当前状态阻止执行该操作； 422 表示结构有效的请求违反了业务约束。状态码和错误码属于公开契约；消息则是稳定的摘要，不包含私有业务规则和实现细节。


读取操作可安全重复，并允许自动重试。写入操作不保证幂等性，因此在超时或传输结果不明确之后，绝不能自动重试。成功的写入具有强一致性：同一对话中后续的读取能够看到已完成的变更。标记为 x-api-pagination: none 的操作会返回完整结果；不要发送未记录的分页参数。

每个操作还会发布 x-api-request-body-max-bytes 和
x-api-response-body-max-bytes。大小按紧凑 JSON 序列化后的 UTF-8 字节计算。契约允许请求最大为 1,048,576 字节（查询参数和正文合计），响应正文最大为 4,194,304 字节。更大的请求会返回 413 request_too_large；若操作结果无法容纳在限制内，则返回 502 response_too_large。将这两类响应与其他已记录的错误一样处理，不要裁剪、拆分或重试写入。

契约有意将两个接口分开：


面向智能体的工具名称、参数、返回类型和描述编写于 workspace/tools.py.

面向客户侧的 HTTP 方法、路径、请求模式、响应模式和错误编写于 client_api/openapi.yaml.


映射不必一一对应。一个智能体工具可以组合多个客户侧请求，多个智能体工具也可以共享一个客户侧操作。开发者本地的确定性实用函数不代表客户侧资源，可以不发起客户 API 请求。

工具包实例可以在同一对话的多次调用之间保留内存中的会话状态。评分时，会按顺序在全新的工具包实例和全新后端上重新执行对话中记录的工具调用，因此每个工具的行为必须是后端状态、参数以及同一对话中先前调用的确定性函数。依赖其他因素——实际时钟时间、随机性或从对话外部带入的状态——的行为，可能在重新执行时出现差异，导致该对话失败。

本地场景后端

run_local_test 为每个开发者编写的场景提供两种互斥后端：全新的确定性开发种子数据，或在密封候选沙盒中运行、由开发者编写的 Python 模拟后端。后端选择仅影响本地测试。提交后的评估对话和保留评估对话始终使用真实客户 API 运行时。

参见 framework/scenario_contract.md 了解 client_api 场景字段、模拟工厂契约、生命周期、跟踪记录、验证钩子和评分限制。