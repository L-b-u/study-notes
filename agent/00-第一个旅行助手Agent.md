# 第一个旅行助手 Agent

源码位于 `D:\AI_study\chapter1\first_agent`。这是刚开始学习 Agent 时写的一个 Python 小 demo，目标不是使用复杂框架，而是亲手跑通“模型、工具、循环”这条最小链路。

## 它解决什么问题

用户提出类似下面的请求：

```text
请查询北京今天的天气，然后根据天气推荐一个旅游景点。
```

Agent 不能只依靠模型记忆直接回答，而是要按照步骤完成任务：

1. 调用天气工具，获取真实天气。
2. 把城市和天气交给景点搜索工具。
3. 根据工具返回的 Observation 组织答案。
4. 使用 `Finish[最终答案]` 表示任务结束。

## 最小公式

```text
Agent = LLM + Tools + Loop
```

这里的 Loop 使用手写的 ReAct 流程：

```text
Thought → Action → Observation → Thought → ... → Finish[答案]
```

- `Thought`：模型判断下一步需要做什么。
- `Action`：模型选择工具并生成调用参数。
- `Observation`：工具执行后返回的结果。
- `Finish`：模型认为信息足够，输出最终答案。

## 文件分区

| 文件 | 分区 | 职责 |
| --- | --- | --- |
| `main.py` | 入口层 | 读取配置、解析命令行参数、启动 Agent |
| `agent.py` | Agent 层 | 维护 ReAct 循环，解析 Action，执行工具 |
| `prompts.py` | 提示词层 | 规定角色、可用工具和模型输出格式 |
| `llm_client.py` | 模型层 | 调用兼容 OpenAI 接口的语言模型 |
| `tools.py` | 工具层 | 提供天气查询和景点搜索能力 |
| `requirements.txt` | 依赖层 | 记录项目所需的 Python 包 |
| `README.md` | 使用说明 | 记录安装、配置和运行命令 |
| `学习笔记.md` | 复习记录 | 总结这个 demo 的原理、流程和踩坑点 |

这种划分比较简单，但已经把不同职责分开了：入口负责启动，Agent 负责循环，模型负责生成文字，工具负责访问外部世界。

## 一次请求怎么运行

```text
用户请求
   ↓
main.py 读取配置并启动 run_agent
   ↓
拼接 prompt_history 和系统提示词
   ↓
LLM 输出一对 Thought / Action
   ↓
解析 Action 并执行工具
   ↓
把工具结果写成 Observation，追加到历史
   ↓
再次调用 LLM，直到输出 Finish[最终答案]
```

以北京天气为例：

```text
Action: get_weather(city="北京")
    ↓
Observation: 北京当前天气：晴，气温 25 摄氏度
    ↓
Action: get_attraction(city="北京", weather="晴，25 摄氏度")
    ↓
Observation: 景点搜索结果
    ↓
Action: Finish[根据天气推荐的最终答案]
```

默认最多循环 5 次。正常情况下，查询天气、搜索景点、生成答案大约需要 3 次循环。

## 关键实现

### 1. 用提示词规定输出协议

`prompts.py` 要求模型每次只输出一组 `Thought` 和 `Action`：

```text
Thought: 下一步计划
Action: get_weather(city="北京")
```

或者在任务完成时输出：

```text
Thought: 已经获得足够信息
Action: Finish[最终答案]
```

这是一种文本协议，不是 OpenAI 原生的 Function Calling。后面的 `agent.py` 依赖这个固定格式，用正则表达式从模型输出中提取工具名和参数。

### 2. 工具只负责获取外部信息

`tools.py` 中有两个工具：

- `get_weather(city)`：请求 `wttr.in`，返回城市天气描述和气温。
- `get_attraction(city, weather)`：调用 Tavily，根据城市和天气搜索景点。

工具返回的结果统一是字符串，Agent 把它包装成 `Observation` 后交给下一轮模型。工具本身不负责组织最终答案，天气和景点之间的判断由模型在下一轮完成。

### 3. 解析 Action 并执行函数

`agent.py` 的 `parse_and_execute()` 大致分为三步：

1. 先查找模型输出中的 `Action:`。
2. 如果是 `Finish[...]`，提取最终答案并结束循环。
3. 否则解析 `tool_name(key="value")`，从 `available_tools` 中找到对应函数并执行。

工具注册表如下：

```python
available_tools = {
    "get_weather": get_weather,
    "get_attraction": get_attraction,
}
```

模型输出的工具名必须存在于这个字典中，否则会返回错误 Observation，让模型在下一轮重新尝试。

### 4. 为什么要截断多余输出

模型有时会一次性输出多组 Thought-Action，例如还没有拿到天气结果，就提前写出下一步景点搜索。`truncate_llm_output()` 会尽量只保留第一组 Thought-Action。

这样做是为了保证每次执行顺序正确：

```text
模型只计划一步 → 执行一步 → 得到 Observation → 再计划下一步
```

否则模型可能会“假装”已经拿到了工具结果，产生幻觉式的后续步骤。

### 5. 用 prompt_history 保存上下文

`run_agent()` 使用一个列表保存对话过程：

```python
prompt_history = [f"用户请求: {user_prompt}"]

for _ in range(max_steps):
    llm_output = llm.generate("\n".join(prompt_history), system_prompt)
    prompt_history.append(llm_output)
    final_answer, observation = parse_and_execute(llm_output)
    prompt_history.append(f"Observation: {observation}")
```

下一轮会把之前的模型输出和 Observation 拼成一整段文本，再作为 user prompt 发给模型。这种方式实现很简单，适合学习；但上下文会不断变长，角色信息也依靠文本格式约定。

## 如何运行

```powershell
cd D:\AI_study\chapter1\first_agent
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

需要在 `.env` 中配置：

- `LLM_API_KEY`
- `LLM_BASE_URL`
- `LLM_MODEL_ID`
- `TAVILY_API_KEY`

检查天气工具和配置：

```powershell
python main.py --check
```

运行默认请求或指定请求：

```powershell
python main.py
python main.py --query "请查询上海今天的天气，并据此推荐一个景点。"
```

## 踩坑点

- `Action` 必须符合约定格式，参数需要写成 `name="value"`。
- `Finish` 是结束信号，不是 `available_tools` 中的普通工具。
- 模型一次生成多步时，要先截断，避免没有 Observation 就执行后续逻辑。
- 没有配置 `TAVILY_API_KEY` 时，天气工具可以运行，但景点搜索会返回错误。
- `get_weather()` 和 Tavily 都依赖网络，网络异常时工具会返回错误字符串。
- `max_steps` 是循环上限，达到上限仍未 Finish 时，任务会结束但没有最终答案。
- 这是教学版实现，还没有原生 Tool Calling、重试机制、会话记忆和复杂错误恢复。

## 可复用模板

以后写最小 Agent，可以先记住下面的骨架：

```python
history = [f"用户请求: {user_prompt}"]

for _ in range(max_steps):
    output = llm.generate("\n".join(history), system_prompt)
    history.append(output)

    final_answer, observation = parse_and_execute(output)
    if final_answer is not None:
        return final_answer

    history.append(f"Observation: {observation}")
```

核心只有三件事：

1. 提示词说明模型可以使用哪些工具，以及必须遵守什么格式。
2. 解析器把模型输出的 Action 转换成真实函数调用。
3. 工具结果以 Observation 写回上下文，供下一轮继续判断。

## 复习检查点

合上代码后，能够回答下面这些问题，就算掌握了这个 demo。下面的答案可以作为复习后的核对。

### 1. Agent 和普通的一次 LLM 调用有什么区别？

普通的 LLM 调用通常是“输入问题 → 模型生成回答”，模型只根据上下文生成文本。

Agent 在此基础上增加了工具和循环：模型先判断下一步行动，程序执行工具并把结果返回给模型，模型再根据新结果决定下一步，直到输出最终答案。因此这个 demo 可以概括为：

```text
普通 LLM：用户问题 → 一次模型回答
Agent：用户问题 → 模型决策 → 工具执行 → 结果反馈 → 再次决策 → 最终回答
```

在代码中，`llm_client.py` 负责一次模型调用，`agent.py` 中的 `run_agent()` 负责把多次调用和工具执行串成循环。

### 2. ReAct 的三个核心部分是什么？它们在代码中分别对应哪里？

ReAct 的三个核心部分是 `Thought`、`Action` 和 `Observation`：

- `Thought`：模型思考下一步要做什么，对应模型按照 `prompts.py` 中的格式生成的思考内容。
- `Action`：模型决定调用哪个工具，对应 `agent.py` 中解析出来的 `get_weather(...)`、`get_attraction(...)` 或 `Finish[...]`。
- `Observation`：工具执行后的结果，对应 `parse_and_execute()` 返回的字符串，并由 `run_agent()` 追加回 `prompt_history`。

完整过程就是：模型产生 Thought 和 Action，程序执行 Action，工具返回 Observation，Observation 再交给模型继续判断。

### 3. 为什么模型输出必须限制为一组 Thought-Action？

因为程序每轮只执行一个 Action，并且执行下一个 Action 前必须先拿到上一个 Action 的 Observation。

如果模型一次输出多组行动，例如在天气结果还没有返回时就提前写出景点搜索，后面的步骤就没有真实依据，可能出现“假装已经调用过工具”的情况。`truncate_llm_output()` 会尝试截断多余的 Thought-Action，只保留当前这一轮需要执行的第一步。

所以限制为一组的核心原因是保证：

```text
计划一步 → 执行一步 → 获取结果 → 再计划下一步
```

### 4. `Finish` 为什么不需要执行工具？

`Finish` 不是外部工具，而是 Agent 的终止信号。它表示模型认为已经获得足够信息，可以直接给用户最终回答。

在 `parse_and_execute()` 中，程序会先判断 Action 是否以 `Finish` 开头；如果是，就提取 `Finish[...]` 中的内容作为 `final_answer`，直接返回给 `run_agent()`，不再查找 `available_tools`，也不再生成 Observation。

### 5. 工具返回结果后，下一轮模型是如何看到它的？

工具执行后，`parse_and_execute()` 返回工具结果，`run_agent()` 会把它包装成：

```text
Observation: 工具返回的结果
```

然后追加到 `prompt_history`。下一轮调用模型时，程序使用：

```python
"\n".join(prompt_history)
```

把用户请求、之前的模型输出和 Observation 拼成一段完整文本，再作为下一轮的 user prompt 发送给 LLM。模型正是通过这段历史看到工具结果的。

### 6. `available_tools` 在工具调用过程中起什么作用？

`available_tools` 是工具名称到 Python 函数的注册表：

```python
available_tools = {
    "get_weather": get_weather,
    "get_attraction": get_attraction,
}
```

解析器从模型输出中得到工具名后，会先检查这个名字是否存在于 `available_tools`。如果存在，就取出对应函数并传入参数执行；如果不存在，就返回错误信息。

它相当于连接“模型输出的文本名称”和“程序中的真实函数”的桥梁，也限制了模型能够调用的工具范围。

### 7. 这个文本解析方案相比原生 Function Calling 有什么局限？

这个 demo 依靠提示词和正则表达式解析普通文本，优点是直观、容易实现，适合学习 Agent 的基本流程；但它的可靠性较弱。

主要局限包括：

- 模型必须严格输出 `Thought`、`Action` 和参数格式，少写引号、换行位置不对或格式变化都可能解析失败。
- `KWARG_PATTERN` 只能解析简单的字符串参数，不能自然处理嵌套对象、数组、数字、布尔值等复杂参数。
- 正则表达式无法完整验证参数类型和工具调用结构，错误通常要到运行时才暴露。
- 模型可能输出额外内容或多组 Action，因此需要额外的截断和错误处理逻辑。
- 工具描述、参数 Schema 和校验都靠提示词与手写代码维护，工具一多就容易失控。

原生 Function Calling 通常会由接口返回结构化的工具名和参数，参数还可以配合 Schema 校验，因此比文本正则解析更稳定、更适合生产环境。不过它隐藏了部分底层细节，而这个 demo 使用文本协议正好能帮助理解 Agent 的基本闭环。
