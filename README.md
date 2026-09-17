# <img width="1258" height="858" alt="image" src="https://github.com/user-attachments/assets/526bb8a9-bca7-46c7-9666-b2def0657b7a" />
# 模型如何调用工具，以及工具结果如何返回模型

## 1. 模型具体怎么调用工具？

大模型本身不会直接执行 TypeScript，也不会自己启动 Python 程序。

当模型根据当前任务、Skill 中的指令以及已经注册的工具信息判断需要调用某个工具时，它会返回一个**结构化的 Tool Call**。

逻辑上可以理解为：

```json
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
```

这段结构化输出不是 Python 调用，也不是 TypeScript 函数调用。

它只是模型向 Pi 发出的一个**工具调用请求**，其含义相当于：

> 请调用名为 `validate_task_input` 的工具，并将 `task_input` 和 `mode` 作为参数传入。

因此，整个过程可以概括为：

```text
模型
  │
  │ 输出 Tool Call
  ▼
Pi
  │
  │ 根据 name 查找已注册工具
  ▼
validate_task_input
  │
  │ 执行 TypeScript 的 execute()
  ▼
实际工具逻辑
```

模型只负责：

1. 判断是否需要使用工具；
2. 选择要调用的工具；
3. 生成符合工具参数 Schema 的参数；
4. 输出结构化 Tool Call。

模型**不负责真正执行工具代码**。

---

## 2. Pi 为什么知道 `validate_task_input` 是什么？

在 Pi 启动时，工具会提前完成注册。

例如，`validate_task_input` 工具会向 Pi 提供类似以下信息：

```text
工具名称：
validate_task_input

工具描述：
检查 TaskInput 是否满足要求。

参数：
- task_input
- mode
```

Pi 会将工具的名称、功能描述以及参数 Schema 提供给模型。

因此模型在推理时能够知道：

```text
当前有哪些工具可以使用
        ↓
每个工具是做什么的
        ↓
每个工具需要哪些参数
```

模型并不知道：

```text
validate_task_input
```

背后具体是：

```text
TypeScript
    ↓
spawn
    ↓
Python
```

这些属于工具的内部实现。

对模型来说，它看到的是一个抽象的工具接口：

```text
validate_task_input(task_input, mode)
```

至于这个接口内部到底通过 TypeScript、Python、Shell，还是其他程序实现，模型并不需要知道。

---

## 3. Skill 在这里起什么作用？

Skill 主要告诉模型：

```text
面对什么任务
应该按照什么流程处理
什么时候应该调用什么工具
工具调用前后应该检查什么
拿到工具结果之后应该怎么继续
```

例如，Skill 中可能规定：

```text
先构造 TaskInput 草案
        ↓
调用 validate_task_input
        ↓
如果验证失败
    根据错误信息修改 TaskInput
        ↓
再次调用 validate_task_input
        ↓
验证通过后进入下一阶段
```

因此：

```text
Skill
```

负责定义**工作流程和决策规则**，

而：

```text
Tool
```

负责完成某个具体的可执行操作。

可以简单理解为：

```text
Skill = 告诉模型“怎么做”

Tool = 帮模型真正“执行某一步”
```

---

# 4. Tool Call 到底是谁执行的？

当模型输出：

```json
{
  "type": "tool_call",
  "name": "validate_task_input",
  "arguments": {
    "task_input": {},
    "mode": "draft"
  }
}
```

Pi 会识别到：

```text
这是一个 Tool Call
```

然后根据：

```text
name = validate_task_input
```

找到启动时已经注册的工具。

之后由 Pi 调用这个工具的 TypeScript `execute()`：

```text
模型
  ↓
Tool Call
  ↓
Pi
  ↓
validate_task_input.execute(...)
```

例如工具内部可能是：

```ts
async execute(_toolCallId, params, signal) {
    const result = await runPythonValidator(
        params.task_input,
        params.mode,
        signal,
    );

    return {
        content: [
            {
                type: "text",
                text: JSON.stringify(result, null, 2),
            },
        ],
        details: result,
    };
}
```

因此真正执行代码的是 Pi 所运行的工具实现，而不是模型。

---

# 5. TypeScript 为什么又会调用 Python？

`validate_task_input` 的 TypeScript 工具本身主要承担的是**适配层**角色。

它收到模型传来的参数后，会进一步调用：

```ts
runPythonValidator(...)
```

即：

```text
Pi
 ↓
execute()
 ↓
runPythonValidator()
```

而 `runPythonValidator()` 内部再负责启动 Python 验证程序。

因此完整调用链变成：

```text
大模型
   ↓
Tool Call
   ↓
Pi
   ↓
validate_task_input.execute()
   ↓
runPythonValidator()
   ↓
Python Validator
```

这里可以将不同层次的职责区分为：

```text
大模型
负责决策

Pi
负责 Tool Call 调度

TypeScript Tool
负责工具接口适配

Python
负责具体业务逻辑
```

---

# 6. Python 的结果怎么返回给模型？

Python 执行完成后，会将验证结果写到标准输出 `stdout`。

例如：

```json
{
  "ok": false,
  "errors": [
    {
      "field": "request",
      "message": "request cannot be empty"
    }
  ]
}
```

之后结果会沿着调用链逐层返回。

完整流程为：

```text
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
```

---

## 7. 第一步：Python 输出结果

Python Validator 完成检查之后，通过：

```text
stdout
```

输出 JSON 结果。

例如：

```json
{
  "ok": true,
  "errors": []
}
```

---

## 8. 第二步：`runPythonValidator()` 获取结果

TypeScript 中的：

```ts
runPythonValidator()
```

负责启动 Python 子进程，并读取 Python 的：

```text
stdout
```

然后将 JSON 字符串解析为 TypeScript 对象。

逻辑上相当于：

```text
Python
  ↓ stdout
JSON 字符串
  ↓
runPythonValidator()
  ↓ JSON.parse(...)
result
```

---

## 9. 第三步：`execute()` 返回 ToolResult

`runPythonValidator()` 得到结果后：

```ts
const result = await runPythonValidator(...);
```

`execute()` 再把结果包装成 Pi 能够识别的 ToolResult：

```ts
return {
    content: [
        {
            type: "text",
            text: JSON.stringify(result, null, 2),
        },
    ],
    details: result,
};
```

此时：

```text
Python 的执行结果
```

已经转换成：

```text
Pi ToolResult
```

---

# 10. 第四步：Pi 把 ToolResult 写入 Session

Pi 收到 ToolResult 后，会把这次工具调用及其结果记录到当前 Session 中。

逻辑上类似：

```text
User Message

Assistant Tool Call:
validate_task_input(...)

Tool Result:
{
    "ok": false,
    "errors": [...]
}
```

这样工具调用就成为当前对话历史的一部分。

---

# 11. 第五步：Pi 再次调用模型

工具执行结束并不意味着整个任务结束。

Pi 会根据当前 Session 重新组装模型输入 Context，其中包括：

```text
System Prompt

Skill 内容

用户请求

模型之前的 Tool Call

Tool Result
```

然后再次调用大模型。

因此模型第二次推理时能够看到：

```text
我刚才调用了 validate_task_input

工具返回：

ok = false

错误：
request 不能为空
```

模型就可以继续决定：

```text
修改 TaskInput
    ↓
重新调用 validate_task_input
```

或者：

```text
验证成功
    ↓
继续执行下一阶段
```

---

# 12. 完整调用链

将整个过程连起来，可以表示为：

```text
                    ┌──────────────────┐
                    │      Skill       │
                    │  定义工作流程与规则 │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │      大模型       │
                    │ 判断需要调用工具   │
                    └────────┬─────────┘
                             │
                             │ Tool Call
                             ▼
                    ┌──────────────────┐
                    │       Pi         │
                    │  工具调度 / Runtime│
                    └────────┬─────────┘
                             │
                             ▼
               ┌──────────────────────────┐
               │ validate_task_input Tool │
               │      execute()           │
               └────────────┬─────────────┘
                            │
                            ▼
               ┌──────────────────────────┐
               │   runPythonValidator()   │
               └────────────┬─────────────┘
                            │
                            ▼
                    ┌──────────────────┐
                    │ Python Validator │
                    │   实际验证逻辑    │
                    └────────┬─────────┘
                             │
                             │ stdout
                             ▼
                    runPythonValidator()
                             │
                             ▼
                         execute()
                             │
                             ▼
                      Pi ToolResult
                             │
                             ▼
                         Session
                             │
                             ▼
                    重新组装 Context
                             │
                             ▼
                         大模型
                             │
                             ▼
                       继续下一步
```

---

# 13. 一句话概括

Pi 在启动阶段注册工具，并将工具名称、描述和参数 Schema 提供给模型；模型根据当前任务和 Skill 中定义的工作流程判断何时需要调用工具，并输出结构化 Tool Call；Pi 拦截该 Tool Call 后执行对应的 TypeScript 工具，TypeScript 再根据实现需要调用 Python 等底层程序；工具执行结果最终以 ToolResult 的形式写回 Session，Pi 再将其加入新的 Context 中重新调用模型，使模型能够基于工具结果继续后续推理与执行。

更简化地说：

```text
Skill 决定“应该怎么做”
        ↓
模型决定“现在要调用哪个工具”
        ↓
Pi 负责“真正调用工具”
        ↓
TypeScript / Python 执行具体逻辑
        ↓
结果返回 Pi
        ↓
Pi 把结果重新交给模型
        ↓
模型继续决策
```
