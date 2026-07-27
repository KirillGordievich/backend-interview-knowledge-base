# AI — RAG (Retrieval-Augmented Generation)

## Что такое RAG

**RAG** — архитектурный паттерн, при котором LLM получает ответ, опираясь на динамически загружаемые релевантные документы, а не только на параметрическую память (веса).

**Зачем:**
- Устранение галлюцинаций на фактических вопросах
- Работа с данными после cutoff date модели
- Контроль над источниками (корпоративные документы, базы знаний)
- Экономия: не дообучать модель при каждом обновлении данных

**Базовый pipeline:**
```
Документы → Chunking → Embedding → Vector Store
                                         ↓
User Query → Embedding → Similarity Search → Top-K чанков
                                                    ↓
                              LLM(query + chunks) → Ответ
```

---

## Embeddings

**Embedding** — числовой вектор (список float), представляющий семантический смысл текста. Похожие по смыслу тексты → близкие векторы.

```python
# OpenAI
from openai import OpenAI
client = OpenAI()

response = client.embeddings.create(
    model="text-embedding-3-small",  # 1536 dims, дешевле
    # model="text-embedding-3-large", # 3072 dims, точнее
    input="What is the capital of France?"
)
vector = response.data[0].embedding  # list[float]

# Sentence Transformers (open-source, self-hosted)
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")       # мультиязычная
# или "intfloat/multilingual-e5-large-instruct"   # хороша для русского

vectors = model.encode(["текст 1", "текст 2"])   # shape: (2, 1024)
```

**Популярные модели эмбеддингов:**

| Модель | Dims | Языки | Хостинг |
|---|---|---|---|
| `text-embedding-3-small` | 1536 | EN | OpenAI API |
| `text-embedding-3-large` | 3072 | EN | OpenAI API |
| `BAAI/bge-m3` | 1024 | 100+ | Self-hosted |
| `intfloat/e5-large-v2` | 1024 | EN | Self-hosted |
| Cohere `embed-multilingual-v3` | 1024 | 100+ | Cohere API |

**Метрики схожести:**
```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
# → [-1, 1], чем ближе к 1 — тем похожее
```

---

## Векторные базы данных

| БД | Особенности | Когда использовать |
|---|---|---|
| **pgvector** | Расширение PostgreSQL, ACID | Уже есть PG, небольшие объёмы (<5M) |
| **Qdrant** | Rust, быстрый, фильтрация payload | Production, гибкие фильтры |
| **Weaviate** | GraphQL API, модульность | Сложные схемы данных |
| **Chroma** | Простой, in-memory/SQLite | Прототипы, локальная разработка |
| **Pinecone** | Fully managed, serverless | Быстрый старт без инфра |
| **Milvus** | Open-source, очень большие объёмы | Billion-scale |

```python
# Qdrant — пример
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

client = QdrantClient(url="http://localhost:6333")

# Создать коллекцию
client.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

# Индексировать документы
points = [
    PointStruct(
        id=i,
        vector=embed(chunk),
        payload={"text": chunk, "source": filename, "page": page_num}
    )
    for i, (chunk, filename, page_num) in enumerate(chunks)
]
client.upsert(collection_name="docs", points=points)

# Поиск
results = client.search(
    collection_name="docs",
    query_vector=embed(user_query),
    limit=5,
    query_filter={"must": [{"key": "source", "match": {"value": "report.pdf"}}]}
)
for r in results:
    print(r.score, r.payload["text"])
```

```sql
-- pgvector
CREATE EXTENSION vector;
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536),
    metadata JSONB
);

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);  -- количество кластеров

-- Поиск
SELECT content, metadata,
       1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

---

## Стратегии chunking для RAG

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    MarkdownHeaderTextSplitter,
)

# 1. Recursive (универсальный)
splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=64)

# 2. По заголовкам Markdown (для структурированных документов)
md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[("##", "section"), ("###", "subsection")]
)
chunks = md_splitter.split_text(markdown_doc)

# 3. Семантический (группирует похожие предложения)
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

semantic_splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # разрыв при большом семантическом скачке
)
```

**Parent-Child chunking:**
```
Индексируем маленькие чанки (128 токенов) → точный retrieval
Но возвращаем родительский чанк (512 токенов) → достаточно контекста

small_chunk → parent_id → large_chunk (хранится отдельно)
```

---

## Стратегии retrieval

### Dense Retrieval (семантический поиск)
Эмбеддинг запроса → косинусное расстояние до эмбеддингов чанков. Работает для перефразировок и синонимов.

### Sparse Retrieval (BM25 / TF-IDF)
Лексическое совпадение. Хорошо работает для точных терминов, кодов, имён собственных.

```python
from rank_bm25 import BM25Okapi

corpus = [chunk.split() for chunk in chunks]
bm25 = BM25Okapi(corpus)
scores = bm25.get_scores(user_query.split())
```

### Hybrid Search (рекомендуется)
Объединяет dense + sparse через Reciprocal Rank Fusion (RRF).

```python
def reciprocal_rank_fusion(dense_results, sparse_results, k=60) -> list:
    scores = {}
    for rank, doc_id in enumerate(dense_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    for rank, doc_id in enumerate(sparse_results):
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

Qdrant, Weaviate, Elasticsearch — поддерживают гибридный поиск нативно.

### HyDE (Hypothetical Document Embeddings)

```python
# Проблема: вопрос и ответ могут иметь разные эмбеддинги
# HyDE: сначала сгенерировать гипотетический ответ, эмбеддить его
async def hyde_retrieve(query: str) -> list:
    hypothetical_doc = await llm.complete(
        f"Write a detailed paragraph that would answer this question: {query}"
    )
    # Эмбеддинг гипотетического ответа ближе к реальным документам
    return vector_store.search(embed(hypothetical_doc), top_k=5)
```

### Multi-Query Retrieval

```python
# Генерировать несколько перефразировок вопроса → больше охват
async def multi_query_retrieve(query: str) -> list:
    rephrases = await llm.complete(f"""
    Generate 3 different ways to ask this question for document retrieval:
    Question: {query}
    Return as JSON array of strings.
    """)
    all_results = []
    for q in rephrases:
        all_results.extend(vector_store.search(embed(q), top_k=3))
    return deduplicate(all_results)
```

---

## Reranking

**Проблема:** Vector search возвращает приблизительные результаты. Reranker — более тяжёлая cross-encoder модель, которая точнее оценивает релевантность.

```
Vector Search → Top-20 → Reranker → Top-3 → LLM
```

```python
# Cohere Rerank
import cohere
co = cohere.Client()

results = co.rerank(
    query=user_query,
    documents=[chunk.text for chunk in retrieved_chunks],
    model="rerank-multilingual-v3.0",
    top_n=3,
)
top_chunks = [retrieved_chunks[r.index] for r in results.results]

# Open-source: cross-encoder/ms-marco-MiniLM-L-6-v2
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
pairs = [(user_query, chunk) for chunk in chunks]
scores = reranker.predict(pairs)
top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:3]
```

---

## Полный RAG pipeline

```python
import anthropic
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer

embed_model = SentenceTransformer("BAAI/bge-m3")
vector_db = QdrantClient(url="http://localhost:6333")
llm = anthropic.Anthropic()

def embed(text: str) -> list[float]:
    return embed_model.encode(text).tolist()

def retrieve(query: str, top_k: int = 5) -> list[str]:
    results = vector_db.search(
        collection_name="docs",
        query_vector=embed(query),
        limit=top_k,
    )
    return [r.payload["text"] for r in results]

def answer(query: str) -> str:
    chunks = retrieve(query)
    context = "\n\n---\n\n".join(chunks)

    response = llm.messages.create(
        model="claude-sonnet-4-6",
        system="""Answer based ONLY on the provided context.
If the answer is not in the context, say "I don't have this information."
Always cite relevant parts of the context.""",
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {query}"
        }],
        max_tokens=1024,
    )
    return response.content[0].text
```

---

## Оценка RAG (RAGAS)

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,          # ответ поддержан контекстом?
    answer_relevancy,      # ответ релевантен вопросу?
    context_precision,     # в top-K много нерелевантного?
    context_recall,        # весь нужный контекст найден?
)
from datasets import Dataset

data = {
    "question": ["What is RAG?"],
    "answer": ["RAG is Retrieval-Augmented Generation..."],
    "contexts": [["RAG stands for Retrieval-Augmented Generation..."]],
    "ground_truth": ["RAG combines retrieval with generation..."],
}
result = evaluate(Dataset.from_dict(data), metrics=[
    faithfulness, answer_relevancy, context_precision, context_recall
])
```

---

## Продвинутые паттерны

**Corrective RAG (CRAG):** если retrieved документы нерелевантны → web search как fallback.

**Self-RAG:** модель сама решает, нужен ли retrieval для этого вопроса (не все вопросы требуют поиска).

**Adaptive RAG:** классификатор сложности запроса → простые отвечаются без retrieval, сложные — с multi-step retrieval.

**RAG Fusion:** несколько queries → несколько retrieval → RRF fusion → LLM с объединёнными результатами.
