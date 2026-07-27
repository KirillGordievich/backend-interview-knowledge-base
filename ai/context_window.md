# AI — Context Window и память

## Что такое контекстное окно

**Context window** — максимальное количество **токенов**, которые модель видит одновременно при генерации следующего токена. Это не просто ввод — сюда входит system prompt + история диалога + текущий запрос + ответ модели.

**Токен ≠ слово:** в среднем 1 токен ≈ 0.75 слова (в английском). "Tokenization" = 3 токена. Русский текст ~дороже (~1.5–2 токена на слово).

```python
import tiktoken  # для OpenAI-моделей

enc = tiktoken.encoding_for_model("gpt-4o")
tokens = enc.encode("Hello, world!")
print(len(tokens))  # 4

# Anthropic — через API или approximation
# Claude: ~1 token ≈ 3.5-4 символа (для английского)
```

**Размеры контекстных окон (2024–2025):**

| Модель | Контекст |
|---|---|
| GPT-4o | 128K токенов |
| Claude 3.5 Sonnet / Claude 4 | 200K токенов |
| Gemini 1.5 Pro | 1M токенов |
| Llama 3.1 70B | 128K токенов |

**Проблема:** большой контекст ≠ хороший контекст. Модели хуже используют информацию в середине длинного контекста ("lost in the middle" проблема). Стоимость растёт квадратично по токенам (attention O(n²)).

---

## Стратегии управления контекстом

### 1. Chunking (разбиение на чанки)

Разбить большой документ на части, чтобы обрабатывать их по отдельности или селективно загружать в контекст.

**Fixed-size chunking:**
```python
def chunk_fixed(text: str, size: int = 512, overlap: int = 50) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start = end - overlap  # перекрытие сохраняет контекст на границах
    return chunks
```

**Recursive character splitting (LangChain-подход):**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,         # символов
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],  # пробует сверху вниз
)
chunks = splitter.split_text(document)
```

**Semantic chunking** — разбивает по смыслу (группирует предложения с похожими эмбеддингами), а не по символам. Дороже, но качественнее для RAG.

**Выбор размера чанка:**

| Размер | Плюсы | Минусы |
|---|---|---|
| Маленький (128–256 токенов) | Точный retrieval, мало шума | Теряется контекст |
| Средний (512–1024) | Баланс | — |
| Большой (2048+) | Весь контекст сохранён | Много нерелевантного в retrieval |

---

### 2. Sliding Window

Вместо загрузки всей истории — скользящее окно последних N токенов.

```python
from collections import deque

class SlidingWindowMemory:
    def __init__(self, max_tokens: int = 4000):
        self.messages: deque = deque()
        self.max_tokens = max_tokens
        self.current_tokens = 0

    def add(self, role: str, content: str, tokens: int):
        self.messages.append({"role": role, "content": content, "tokens": tokens})
        self.current_tokens += tokens
        while self.current_tokens > self.max_tokens and self.messages:
            removed = self.messages.popleft()
            self.current_tokens -= removed["tokens"]

    def get_context(self) -> list[dict]:
        return [{"role": m["role"], "content": m["content"]} for m in self.messages]
```

**Проблема:** теряется ранняя история диалога. Пользователь может ссылаться на то, что уже вышло из окна.

---

### 3. Hierarchical Summarization (иерархическое суммирование)

Старые части диалога или документа сворачиваются в summary, который остаётся в контексте.

```
[msg 1][msg 2][msg 3] ... [msg 20]
           ↓ (при переполнении)
[summary of msgs 1-10][msg 11]...[msg 20]
           ↓ (при следующем переполнении)
[summary of summaries][msg 16]...[msg 20]
```

```python
async def compress_history(messages: list[dict], llm) -> dict:
    history_text = "\n".join(
        f"{m['role']}: {m['content']}" for m in messages
    )
    summary = await llm.complete(
        f"Summarize this conversation concisely, preserving key facts:\n{history_text}"
    )
    return {"role": "system", "content": f"[Previous conversation summary]: {summary}"}
```

**Применение:** долгие агентные сессии, чат-боты с памятью.

---

### 4. Retrieval Before Generation (RAG-подход)

Вместо хранения всего в контексте — хранить в векторном хранилище и подгружать только релевантное.

```
Большой корпус документов
    → Embedding → Векторная БД
                        ↓
              Запрос пользователя
                   → Embedding
                   → Top-K retrieval
                   → Только топ чанки → LLM
```

Подробнее: [rag.md](rag.md).

---

### 5. Context Compression

Сжать загружаемый контекст без потери ключевой информации.

**LLMLingua / Selective compression:**
```python
# Идея: удалить токены с наименьшей перплексией (наименее важные)
# LLMLingua, LongLLMLingua — open-source библиотеки от Microsoft
from llmlingua import PromptCompressor

compressor = PromptCompressor(model_name="microsoft/llmlingua-2-bert-base-multilingual-cased-meetingbank")
compressed = compressor.compress_prompt(
    context,
    instruction="Answer the question based on the context",
    question=user_query,
    target_token=300,   # сжать до 300 токенов
)
# Экономия 2–6x токенов при минимальной потере качества
```

**Reranking + truncation:** получить 20 чанков через retrieval → reranker выбирает топ-3 → в контекст идут только они.

---

### 6. KV Cache

Модели кэшируют вычисления attention для уже обработанных токенов. Длинный неизменяемый system prompt → кэшируется → сильно снижает стоимость и задержку.

```python
# Anthropic: prompt caching
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    system=[{
        "type": "text",
        "text": very_long_system_prompt,
        "cache_control": {"type": "ephemeral"}  # кэшировать этот блок
    }],
    messages=[{"role": "user", "content": user_query}]
)
# Первый запрос: полная стоимость
# Следующие запросы с тем же prompt: ~10% стоимости для закэшированных токенов
```

---

## Типы памяти в LLM-приложениях

| Тип | Где хранится | Время жизни | Пример |
|---|---|---|---|
| **In-context** | Context window | Один запрос | Текущий диалог |
| **External (vector)** | Векторная БД | Долгосрочно | Прошлые разговоры, документы |
| **External (key-value)** | Redis / БД | Долгосрочно | Факты о пользователе |
| **Summary** | Context (сжато) | Сессия | Сжатая история |
| **Episodic** | БД + retrieval | Долгосрочно | "Вчера ты упоминал X" |
