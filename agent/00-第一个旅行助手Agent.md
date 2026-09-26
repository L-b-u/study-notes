# 第一个旅行助手 Agent

源码目录：`D:\AI_study\chapter1\first_agent`。这是 Hello-Agents 第一章的最小可运行项目，目标不是堆框架，而是亲手把 Agent 骨架跑通。

## 它解决什么问题

用户给出一句话，例如「查北京今天天气，再据此推荐景点」。Agent 不能直接瞎编，必须：

1. 先调用天气工具拿到真实天气。
2. 再把城市和天气交给搜索工具找景点。
3. 最后用 `Finish[最终答案]` 结束。

## 最小公式

```text
Agent = LLM + Tools + Loop
```

这里的 Loop 是手写 ReAct：

```text
Thought -> Action -> Observation -> Thought -> ... -> Finish[答案]
```

| 模块 | 文件 | 职责 |
| --- | --- | --- |
| 提示词 | `prompts.py` | 规定角色、可用工具、输出格式 |
| 工具 | `tools.py` | `get_weather`、`get_attraction` |
| LLM 客户端 | `llm_client.py` | 调用任意 OpenAI 兼容接口 |
| 主循环 | `agent.py` | 解析 Thought-Action、执行工具、拼接 Observation |
| 入口 | `main.py` | 读 `.env`、参数解析、`--check` 体检 |

## 一次请求怎么跑

```text
用户请求
   |
   v
拼接 prompt_history + 系统提示词
   |
   v
LLM 输出一对 Thought / Action
   |
   +-- Action: get_weather(city="北京")
   |      -> Observation: 北京当前天气:...
   |      -> 写回 history，进入下一轮
   |
   +-- Action: get_attraction(city="北京", weather="...")
   |      -> Observation: 景点推荐...
   |      -> 写回 history，进入下一轮
   |
   +-- Action: Finish[最终答案]
          -> 结束循环
```

默认最多 5 步。北京天气查询通常是 3 步：查天气 -> 搜景点 -> Finish。

## 关键实现

### 1. 用提示词约束格式

系统提示词要求每次只输出一对：

```text
Thought: ...
Action: function_name(arg="value")
```

或结束：

```text
Action: Finish[最终答案]
```

这是文本协议，不是 OpenAI 原生 Function Calling。模型必须「按格式说话」，后面才能用正则解析。

### 2. 工具很薄，只负责外部世界

- `get_weather(city)`：请求 `https://wttr.in/{city}?format=j1`，返回天气描述和气温。
- `get_attraction(city, weather)`：用 Tavily 搜索「某城市在某种天气下值得去的景点」。

工具返回字符串 Observation。Agent 自己不判断天气好坏，判断发生在下一轮 Thought 里。

### 3. 解析层把「模型文本」变成「可执行调用」

`agent.py` 做了三件事：

1. 截断多余的 Thought-Action 对。模型有时会一次把后面几步也写出来，必须只保留第一对，否则还没拿到 Observation 就开始假装已经查完了。
2. 用正则抽出 `Action:`，再解析 `tool_name(k="v")` 或 `Finish[...]`。
3. 在本地字典 `available_tools` 里找到函数并执行。

参数解析只认 `key="value"` / `key='value'`。少写引号、写成位置参数，都会解析失败。

### 4. 历史不是 messages 列表

`run_agent()` 把每一轮的模型输出和 Observation 追加进 `prompt_history`，下一轮整段当成 **一条 user prompt** 再发给模型。

这和 Chat Completions 的多轮 `messages` 不同：

- 优点：实现极简，容易看懂。
- 代价：上下文会线性变长；role 信息全靠文本约定。

LLM 调用本身也只有 `system + user` 两条消息，`temperature=0`，追求格式稳定。

## 怎么跑

```powershell
cd D:\AI_study\chapter1\first_agent
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install openai python-dotenv requests tavily-python
Copy-Item .env.example .env
```

`.env` 需要：

- `LLM_API_KEY`
- `LLM_BASE_URL`
- `LLM_MODEL_ID`
- `TAVILY_API_KEY`

兼容旧变量名：`OPENAI_API_KEY`、`OPENAI_BASE_URL`、`MODEL_NAME`。

```powershell
python main.py --check
python main.py
python main.py --query "请查询上海今天的天气，并据此推荐一个景点。"
```

`--check` 会真实请求一次北京天气，并检查四项配置是否齐全。

## 可复用模板

以后自己写最小 Agent，可以按这个骨架：

```python
prompt_history = [f"用户请求: {user_prompt}"]

for _ in range(max_steps):
    llm_output = llm.generate("\n".join(prompt_history), system_prompt)
    prompt_history.append(llm_output)

    final_answer, observation = parse_and_execute(llm_output)
    if final_answer is not None:
        return final_answer

    prompt_history.append(f"Observation: {observation}")
```

核心不在框架，而在三件事：

1. 提示词把工具和输出格式讲清楚。
2. 解析器能把 Action 变成真实函数调用。
3. Observation 必须写回上下文，下一轮才能继续推理。

## 踩坑点

- 模型一次输出多步时必须截断，否则会「幻觉执行」。
- 文本解析很脆：Action 要单行，参数必须 `name="value"`。
- 没配 `TAVILY_API_KEY` 时天气能查，景点工具会直接报错。
- README 写了 `pip install -r requirements.txt`，仓库里目前没有这个文件，按上面四个包安装即可。
- 这是教学版 Agent：没有原生 tool calling、没有重试策略、没有记忆隔离。先理解这条闭环，再对比 LangChain / 原生 function calling。

## 复习检查点

合上代码后，能回答这 5 句就算过关：

1. Agent 和一次普通 LLM 调用差在哪？差在工具和循环。
2. ReAct 三元组是什么？Thought、Action、Observation。
3. 为什么要截断多余 Thought-Action？避免还没观察就提前编造后续步骤。
4. Finish 为什么不是普通工具？它是终止信号，不再产生 Observation。
5. 下一轮模型怎么知道工具结果？因为 Observation 被追加进了 prompt_history。
