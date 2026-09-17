# <img width="1258" height="858" alt="image" src="https://github.com/user-attachments/assets/526bb8a9-bca7-46c7-9666-b2def0657b7a" />
# 模型具体怎么调用？
## 模型不会自己执行 TypeScript，也不会自己运行 Python。
它返回一种结构化的模型输出，逻辑上类似：
{
  "type": "tool_call",
  "name": "validate_task_input",
  "arguments": {
    "task_input": {
      "schema_version": "0.2",
      "task_id": "urban-model-update",
      "task_input_version": 1,
      "request": {},
      "assets": {},
      "requirements": {},
      "intake": {}
    },
    "mode": "draft"
  }
}
这段结构是模型输出给 Pi 的请求，意思是：
请 Pi 帮我执行名为 validate_task_input 的工具，并传入这些参数。

模型只负责提出 Tool Call，不直接执行工具。
# 工具结果怎么回到模型？
## Python 的结果依次经过：
Python stdout
    ↓
runPythonValidator()
    ↓
execute()
    ↓
Pi ToolResult
    ↓
写入 Session
    ↓
Pi 重新组装 Context
    ↓
再次调用大模型
## 一句话概括：
Pi 在启动时注册工具，并把工具说明提供给模型；模型根据 Skill 判断何时需要它，然后输出结构化 Tool Call；Pi 拦截这个调用并执行 TypeScript/Python，最后把结果放回 Session 供模型继续推理。
