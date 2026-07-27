# AI — Галлюцинации LLM

## Что такое галлюцинация

**Галлюцинация** — ситуация, когда LLM генерирует правдоподобно звучащий, но фактически неверный, выдуманный или не подтверждённый контекстом ответ.

**Виды галлюцинаций:**

| Тип | Описание | Пример |
|---|---|---|
| **Фактическая** | Неверные факты о реальном мире | "Эйнштейн родился в 1880 году" |
| **Контекстная** | Противоречие предоставленному контексту | Ссылка на документ, которого нет |
| **Атрибуционная** | Неверная ссылка на источник | Несуществующая цитата или статья |
| **Логическая** | Ошибка в рассуждении | Неверный вывод из верных посылок |
| **Экстраполяционная** | Выход за пределы известного | Домысливание деталей |

---

## Почему возникают галлюцинации

**1. Природа языковых моделей**
LLM — это предиктор следующего токена. Модель не "знает" факты — она усвоила статистику совместной встречаемости токенов. Уверенность в генерации ≠ фактическая точность.

**2. Пробелы в обучающих данных**
Редкие факты, узкие предметные области, свежие события (cutoff date) — модель "заполняет пробелы" правдоподобными, но ложными данными.

**3. Отсутствие верификации**
Модель не имеет механизма "я не знаю, дай проверю". Она всегда генерирует ответ, даже не имея достаточно информации.

**4. RLHF-смещение**
Обучение с подкреплением на основе обратной связи людей (RLHF) оптимизирует под "звучащий правдоподобно" ответ, что может усиливать уверенный тон при неточных данных.

**5. Attention limitations**
Важная информация в середине длинного контекста обрабатывается хуже ("lost in the middle"), что приводит к игнорированию верных данных.

---

## Стратегии снижения галлюцинаций

### 1. RAG (Retrieval-Augmented Generation)

Заземление ответа на конкретных документах — самый эффективный метод для фактических задач.

```python
# Паттерн: сначала получить документы, затем ответить только по ним
system_prompt = """
You are a helpful assistant. Answer ONLY based on the provided context.
If the information is not in the context, say "I don't have information about this."
Do NOT use your general knowledge.
"""

context = retrieve_relevant_chunks(user_query)  # из векторной БД

response = client.messages.create(
    model="claude-sonnet-4-6",
    system=system_prompt,
    messages=[{
        "role": "user",
        "content": f"Context:\n{context}\n\nQuestion: {user_query}"
    }]
)
```

Подробнее: [rag.md](rag.md).

---

### 2. Structured Outputs / Constrained Generation

Ограничить формат ответа — меньше пространства для выдумки.

```python
# Pydantic + instructor (обёртка над API)
import instructor
from pydantic import BaseModel, Field
from anthropic import Anthropic

client = instructor.from_anthropic(Anthropic())

class ProductInfo(BaseModel):
    name: str
    price: float | None = Field(None, description="Price if mentioned in context")
    in_stock: bool
    confidence: float = Field(ge=0.0, le=1.0, description="Confidence 0-1")
    source_quote: str = Field(description="Exact quote from context supporting the answer")

result = client.messages.create(
    model="claude-sonnet-4-6",
    response_model=ProductInfo,
    messages=[{"role": "user", "content": f"Context: {context}\nExtract product info."}]
)
# Если в контексте нет цены → price=None, а не выдуманное число
```

**JSON Mode** (OpenAI/Anthropic):
```python
# Явное указание вернуть JSON снижает свободу галлюцинировать
response = client.messages.create(
    model="claude-sonnet-4-6",
    system="Return only valid JSON. No explanation.",
    messages=[{"role": "user", "content": prompt}]
)
```

---

### 3. Function Calling / Tool Use

Вместо того чтобы "помнить" факты, модель вызывает инструменты для получения актуальных данных.

```python
tools = [
    {
        "name": "get_stock_price",
        "description": "Get real-time stock price. Use this instead of guessing.",
        "input_schema": {
            "type": "object",
            "properties": {"ticker": {"type": "string"}},
            "required": ["ticker"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-6",
    tools=tools,
    messages=[{"role": "user", "content": "What is Apple's current stock price?"}]
)
# Модель вызывает инструмент → получает реальные данные → отвечает на их основе
# Не гадает цену из "памяти"
```

---

### 4. Few-Shot Примеры

Показать модели примеры правильного поведения, включая отказ отвечать при нехватке данных.

```python
few_shot = """
Example 1:
Context: "The product was released in March 2023."
Question: "When was the product released?"
Answer: "The product was released in March 2023."

Example 2:
Context: "The product was released in March 2023."
Question: "How much does the product cost?"
Answer: "I cannot find information about the product's price in the provided context."

Example 3:
Context: "Revenue grew 15% YoY to $2.3B."
Question: "What was revenue last year?"
Answer: "Based on the context, this year's revenue is $2.3B with 15% YoY growth, which implies last year's revenue was approximately $2.0B. However, this is a calculation, not a direct statement."
"""
```

---

### 5. System Prompt (инструкции поведения)

Чёткие ограничения в системном промпте снижают риск выдумки.

```python
system_prompt = """
You are a precise factual assistant. Follow these rules strictly:

1. ONLY state facts that are explicitly mentioned in the provided context
2. If unsure, say "I'm not certain" or "The context doesn't specify this"
3. Never invent names, dates, numbers, or citations
4. If asked about something outside the context, say: "This information is not in the provided materials"
5. For numerical claims, always quote the source text

Bad: "The CEO founded the company in 2010" (not in context)
Good: "The context doesn't mention when the CEO founded the company"
"""
```

---

### 6. Verification Chain (цепочка верификации)

Многошаговая проверка: модель генерирует ответ, затем сама же его верифицирует.

```python
async def verified_answer(question: str, context: str) -> dict:
    # Шаг 1: Сгенерировать ответ
    draft = await llm.complete(
        f"Context: {context}\nQuestion: {question}\nAnswer:"
    )

    # Шаг 2: Проверить каждое утверждение
    verification = await llm.complete(f"""
    Context: {context}
    Answer to verify: {draft}

    For each factual claim in the answer:
    1. Find the supporting quote in the context
    2. Mark as SUPPORTED / UNSUPPORTED / EXTRAPOLATED

    Return JSON: {{"claims": [{{"claim": "...", "status": "SUPPORTED", "quote": "..."}}]}}
    """)

    # Шаг 3: Убрать неподтверждённые утверждения
    final = await llm.complete(f"""
    Original answer: {draft}
    Verification: {verification}

    Rewrite the answer keeping only SUPPORTED claims.
    """)

    return {"answer": final, "verification": verification}
```

**Self-Consistency:** задать один вопрос несколько раз с высокой температурой → выбрать наиболее частый ответ (majority voting).

```python
async def self_consistent_answer(question: str, n: int = 5) -> str:
    answers = []
    for _ in range(n):
        response = await llm.complete(question, temperature=0.7)
        answers.append(response)

    # Для числовых ответов — медиана
    # Для текстовых — ещё один вызов для синтеза
    synthesis = await llm.complete(
        f"These {n} answers were generated for the same question: {answers}\n"
        f"What is the most consistent answer? Identify the consensus."
    )
    return synthesis
```

---

### 7. Temperature и параметры сэмплинга

```python
# Для фактических задач (документы, код, данные) — низкая температура
response = client.messages.create(
    model="claude-sonnet-4-6",
    temperature=0.0,   # детерминированный, минимум творчества
    messages=[...]
)

# Для творческих задач — выше
response = client.messages.create(
    model="claude-sonnet-4-6",
    temperature=0.9,
    messages=[...]
)
```

| Параметр | Описание | Для фактических задач |
|---|---|---|
| `temperature` | Случайность сэмплинга | 0.0–0.3 |
| `top_p` | Nucleus sampling (процент вероятности) | 0.9–1.0 |
| `top_k` | K наиболее вероятных токенов | не используется у Claude |

---

## Оценка галлюцинаций (метрики RAGAS)

| Метрика | Что измеряет | Как |
|---|---|---|
| **Faithfulness** | Поддержан ли ответ контекстом | Каждое утверждение → проверка в контексте |
| **Answer Relevancy** | Релевантен ли ответ вопросу | Обратная генерация вопроса из ответа |
| **Context Precision** | Насколько точен retrieved контекст | Доля релевантных чанков в топ-K |
| **Context Recall** | Найдена ли вся нужная информация | Покрытие ground truth |

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision]
)
print(result)  # {"faithfulness": 0.87, "answer_relevancy": 0.92, ...}
```

---

## Сводная таблица стратегий

| Стратегия | Сложность | Эффективность | Когда применять |
|---|---|---|---|
| RAG | Средняя | Высокая | Фактические вопросы по корпусу |
| Structured outputs | Низкая | Средняя | Извлечение данных, классификация |
| Function calling | Средняя | Высокая | Актуальные данные (цены, время, API) |
| Few-shot | Низкая | Средняя | Показать желаемое поведение |
| System prompt | Очень низкая | Низкая–средняя | Базовая защита всегда |
| Verification chain | Высокая | Высокая | Критически важная точность |
| Self-consistency | Высокая | Средняя | Нет ground truth, нужен консенсус |
| Low temperature | Очень низкая | Низкая | Всегда для фактических задач |
