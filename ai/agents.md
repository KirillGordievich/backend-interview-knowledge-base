# AI — Агенты (LLM Agents)

## Что такое агент

**LLM Agent** — система, в которой LLM выступает "мозгом": принимает решения, вызывает инструменты, итеративно работает к достижению цели.

```
Agent = LLM + Tools + Memory + Planning loop
```

**Отличие от обычного LLM-вызова:**
- Один вызов → один ответ
- Агент → несколько шагов, каждый шаг зависит от предыдущего, может вызывать внешние инструменты

---

## Tool Use / Function Calling

Базовый блок агентности — возможность вызывать инструменты.

```python
import anthropic
import json

client = anthropic.Anthropic()

# Определение инструментов
tools = [
    {
        "name": "web_search",
        "description": "Search the web for current information",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "read_file",
        "description": "Read contents of a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"}
            },
            "required": ["path"]
        }
    }
]

def execute_tool(name: str, inputs: dict) -> str:
    if name == "web_search":
        return search_web(inputs["query"])   # реальная реализация
    elif name == "read_file":
        return open(inputs["path"]).read()
    raise ValueError(f"Unknown tool: {name}")

# Агентный цикл
def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            tools=tools,
            messages=messages,
            max_tokens=4096,
        )

        # Добавить ответ модели в историю
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            # Агент завершил работу
            text_blocks = [b.text for b in response.content if hasattr(b, "text")]
            return "\n".join(text_blocks)

        if response.stop_reason == "tool_use":
            # Выполнить все запрошенные инструменты
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            messages.append({"role": "user", "content": tool_results})
```

---

## ReAct Pattern (Reason + Act)

Наиболее распространённый паттерн для агентов: модель явно рассуждает перед каждым действием.

```
Thought: Мне нужно найти текущую цену AAPL
Action: web_search("AAPL stock price today")
Observation: Apple Inc. (AAPL) - $189.30 ▲ 1.2%

Thought: Цена найдена. Теперь могу ответить.
Action: [конечный ответ]
```

```python
system_prompt = """You are a helpful agent with access to tools.

For each step:
1. Think: analyze what you need to do
2. Act: use a tool if needed
3. Observe: process the tool result
4. Repeat until you can give a final answer

Always show your reasoning before each action."""
```

Современные модели (Claude 3.5+, GPT-4o) делают это неявно — они сами решают когда рассуждать, а когда вызывать инструменты.

---

## Типы памяти агента

| Тип | Описание | Реализация |
|---|---|---|
| **In-context** | Текущий диалог и результаты инструментов | Список `messages` |
| **Short-term (buffer)** | Последние N взаимодействий | Sliding window |
| **Long-term (semantic)** | Воспоминания по смыслу | Векторная БД + retrieval |
| **Episodic** | Прошлые сессии и события | БД + retrieval по дате/теме |
| **Procedural** | Как делать задачи (инструкции) | System prompt / знания в векторной БД |

```python
# Простая реализация долгосрочной памяти
class AgentMemory:
    def __init__(self, vector_store, embed_fn):
        self.vector_store = vector_store
        self.embed = embed_fn

    def save(self, text: str, metadata: dict):
        self.vector_store.upsert([{
            "vector": self.embed(text),
            "payload": {"text": text, **metadata}
        }])

    def recall(self, query: str, top_k: int = 3) -> list[str]:
        results = self.vector_store.search(self.embed(query), limit=top_k)
        return [r.payload["text"] for r in results]

# Использование в агентном цикле:
# before_response: memory.recall(user_query) → вставить в контекст
# after_response: memory.save(f"User asked: {query}. I answered: {response}")
```

---

## Планирование

### Sequential (последовательное)
Шаги выполняются один за другим. Подходит для чётко определённых задач.

```
Step 1: Найти документ → Step 2: Извлечь данные → Step 3: Форматировать ответ
```

### Parallel (параллельное)
Независимые задачи выполняются одновременно.

```python
import asyncio

async def parallel_research(topics: list[str]) -> dict:
    tasks = [search_and_summarize(topic) for topic in topics]
    results = await asyncio.gather(*tasks)
    return dict(zip(topics, results))
```

### Hierarchical (иерархическое)
Оркестратор разбивает задачу на подзадачи → делегирует подагентам.

```
Orchestrator Agent
    ├── Research Agent  → ищет информацию
    ├── Writing Agent   → пишет текст
    └── Review Agent    → проверяет результат
```

### Plan-and-Execute

```python
async def plan_and_execute(goal: str):
    # Фаза планирования
    plan = await llm.complete(f"""
    Create a step-by-step plan to accomplish: {goal}
    Return JSON: {{"steps": [{{"id": 1, "action": "...", "depends_on": []}}]}}
    """)

    # Фаза исполнения
    results = {}
    for step in plan["steps"]:
        # Ждём зависимости
        context = {dep: results[dep] for dep in step["depends_on"]}
        results[step["id"]] = await execute_step(step["action"], context)

    return results
```

---

## Мультиагентные системы

### Supervisor / Orchestrator pattern

```python
# LangGraph style
from langgraph.graph import StateGraph, END

def supervisor(state):
    """Решает, какой агент нужен следующим"""
    next_agent = llm.decide(state["messages"], available_agents)
    return {"next": next_agent}

def research_agent(state):
    results = search(state["query"])
    return {"messages": state["messages"] + [{"role": "tool", "content": results}]}

def writer_agent(state):
    draft = llm.write(state["messages"])
    return {"messages": state["messages"] + [{"role": "assistant", "content": draft}]}

graph = StateGraph(AgentState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", research_agent)
graph.add_node("writer", writer_agent)

graph.add_conditional_edges("supervisor", lambda s: s["next"],
    {"researcher": "researcher", "writer": "writer", "END": END})
graph.add_edge("researcher", "supervisor")
graph.add_edge("writer", "supervisor")
```

### Специализация агентов

```
User → Router Agent → [Code Agent | Research Agent | Data Agent]
                                ↓
                        Aggregator Agent → Final Response
```

---

## Frameworks

| Framework | Особенности | Когда использовать |
|---|---|---|
| **LangChain** | Много abstractions, chains, огромная экосистема | Быстрый старт, много готовых компонентов |
| **LangGraph** | Граф состояний, циклы, надёжные агенты | Сложные многоагентные workflow |
| **LlamaIndex** | Специализирован на RAG | Data-heavy RAG приложения |
| **CrewAI** | Ролевые агенты, простой API | Командная работа агентов |
| **AutoGen** | Microsoft, multi-agent conversations | Research, complex reasoning |
| **Anthropic Agent SDK** | Нативный для Claude | Claude-специфичные агенты |

```python
# LangGraph — минимальный пример агента
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Weather in {city}: 22°C, sunny"

model = ChatAnthropic(model="claude-sonnet-4-6")
agent = create_react_agent(model, tools=[get_weather])

result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in Paris?"}]
})
```

---

## Надёжность и наблюдаемость

### Retry и fallback

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
)
async def reliable_llm_call(messages: list) -> str:
    response = await llm.complete(messages)
    return response

# Fallback на другую модель
async def llm_with_fallback(messages: list) -> str:
    try:
        return await call_primary_model(messages)   # claude-opus-4-6
    except Exception:
        return await call_fallback_model(messages)  # claude-haiku-4-5
```

### Трассировка (LangSmith / Langfuse)

```python
# Langfuse — open-source observability для LLM
from langfuse import Langfuse
from langfuse.decorators import observe

langfuse = Langfuse()

@observe()
def run_rag_pipeline(query: str) -> str:
    chunks = retrieve(query)       # автоматически трекается
    answer = generate(query, chunks)  # логируется latency, tokens, cost
    return answer

# В dashboard: traces, latency, token cost, feedback
```

### Guardrails

```python
# Проверка входа/выхода агента
class GuardedAgent:
    def __init__(self, agent, input_guard, output_guard):
        self.agent = agent
        self.input_guard = input_guard
        self.output_guard = output_guard

    async def run(self, query: str) -> str:
        # Проверить вход
        if not await self.input_guard.check(query):
            return "I cannot process this request."

        result = await self.agent.run(query)

        # Проверить выход
        if not await self.output_guard.check(result):
            return "The response was filtered for safety."

        return result

# Примеры guardrails: NeMo Guardrails, LlamaGuard, Guardrails AI
```

---

## Оценка агентов

**Сложнее, чем оценка одного LLM-вызова** — нужно оценивать весь trajectory.

| Что оценивать | Метрика |
|---|---|
| Правильность финального ответа | Accuracy vs ground truth |
| Эффективность: сколько шагов | Avg steps to solution |
| Использование инструментов | Tool call precision/recall |
| Зацикливание | Max steps exceeded rate |
| Стоимость | Avg tokens per task |

```python
# Trajectory evaluation
def evaluate_trajectory(trajectory: list[dict], expected_answer: str) -> dict:
    return {
        "correct": trajectory[-1]["answer"] == expected_answer,
        "steps": len([t for t in trajectory if t["type"] == "tool_call"]),
        "redundant_calls": count_redundant_tool_calls(trajectory),
        "total_tokens": sum(t.get("tokens", 0) for t in trajectory),
    }
```

---

## Типичные паттерны на собесе

**Q: Как предотвратить бесконечные циклы агента?**
- Max iterations limit
- Budget token limit
- Детектор повторяющихся действий
- Timeout

**Q: Как агент решает, когда остановиться?**
- Явное условие завершения в system prompt
- `stop_reason == "end_turn"` (Claude перестаёт вызывать инструменты)
- Специальный инструмент `finish(answer)` который агент должен вызвать

**Q: Как справиться с ошибками инструментов?**
```python
try:
    result = execute_tool(name, inputs)
except Exception as e:
    result = f"Tool error: {e}. Try a different approach or tool."
# Агент сам решит как поступить с ошибкой
```

**Q: Параллельные вызовы инструментов?**
Claude поддерживает параллельные tool_use блоки в одном ответе:
```python
# Claude может запросить несколько инструментов одновременно
# response.content = [ToolUseBlock(name="search"), ToolUseBlock(name="read_file")]
# → выполнить параллельно через asyncio.gather()
```
