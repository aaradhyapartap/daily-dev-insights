# 📌 Vector databases and semantic search
*August 13, 2026 · Daily Dev Insight*

## 🧠 Overview

Vector databases aren't just another NoSQL fad—they represent a fundamental shift in how we think about search and similarity. Traditional databases excel at exact matches and structured queries, but they fall apart when you need to find "things that are kinda like this thing." Vector databases store high-dimensional numerical representations (embeddings) of your data, enabling semantic search where you can find relevant results based on *meaning* rather than keyword matching.

The magic happens through embeddings—dense vectors that capture semantic relationships in a continuous space. Words, images, or even entire documents get transformed into points in this space where proximity indicates similarity. Instead of searching for exact text matches, you're searching for conceptual neighbors. This is why you can search for "feline pet" and get results about cats, or find code snippets that accomplish the same goal despite using completely different variable names.

The real engineering challenge isn't understanding the theory—it's choosing the right vector database (Pinecone, Weaviate, Qdrant, or Chroma), managing embedding quality, and balancing accuracy with performance at scale. Most production systems use approximate nearest neighbor (ANN) algorithms because exact searches in high-dimensional space are computationally prohibitive.

## 💡 Key Concepts

- **Embeddings are everything**: Your vector database is only as good as your embedding model. Modern transformers like `text-embedding-3-small` or open-source alternatives like `all-MiniLM-L6-v2` convert text into vectors. Garbage embeddings = garbage results.

- **Dimensionality matters**: More dimensions capture more nuance but increase storage and search time. Common embeddings range from 384 to 1536 dimensions. Don't cargo-cult the largest model—benchmark for your use case.

- **ANN algorithms trade accuracy for speed**: HNSW (Hierarchical Navigable Small World), IVF (Inverted File Index), and other algorithms make search tractable by finding "good enough" neighbors instead of perfect ones. Know your precision/recall tradeoffs.

- **Metadata filtering is crucial**: Pure vector search often isn't enough. You need hybrid queries like "semantically similar to this query AND published after 2025 AND tagged as Python." Not all vector DBs handle this equally well.

- **Cold start problem**: Vector databases need data to be useful. Unlike traditional search where you can launch with basic indexing, you need to generate embeddings for your entire corpus upfront—and keep them updated.

## 🐍 Python Example

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from sentence_transformers import SentenceTransformer

# Initialize embedding model and vector database
model = SentenceTransformer('all-MiniLM-L6-v2')  # 384-dimensional embeddings
client = QdrantClient(":memory:")  # Use ":memory:" for demo, URL for production

# Create a collection for code snippets
collection_name = "code_snippets"
client.create_collection(
    collection_name=collection_name,
    vectors_config=VectorParams(size=384, distance=Distance.COSINE)
)

# Sample documents with metadata
snippets = [
    {"id": 1, "code": "def merge_sort(arr): ...", "language": "python", "topic": "algorithms"},
    {"id": 2, "code": "async function fetchData(url) { ... }", "language": "javascript", "topic": "networking"},
    {"id": 3, "code": "SELECT * FROM users WHERE age > 18", "language": "sql", "topic": "database"},
    {"id": 4, "code": "class BinaryTree: def insert(self, val): ...", "language": "python", "topic": "data-structures"}
]

# Generate embeddings and insert
points = [
    PointStruct(
        id=snippet["id"],
        vector=model.encode(snippet["code"]).tolist(),
        payload=snippet  # Store original metadata
    )
    for snippet in snippets
]
client.upsert(collection_name=collection_name, points=points)

# Semantic search: find sorting-related code
query = "sort an array efficiently"
query_vector = model.encode(query).tolist()

results = client.search(
    collection_name=collection_name,
    query_vector=query_vector,
    limit=2,
    query_filter={"must": [{"key": "language", "match": {"value": "python"}}]}  # Hybrid search!
)

for result in results:
    print(f"Score: {result.score:.3f} | {result.payload['code'][:50]}...")
```

## 🟨 JavaScript Example

```javascript
import { QdrantClient } from '@qdrant/js-client-rest';
import { pipeline } from '@xenova/transformers';

// Initialize embedding pipeline (runs locally via ONNX)
const embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
const client = new QdrantClient({ url: 'http://localhost:6333' });

const collectionName = 'documentation';

// Create collection if it doesn't exist
await client.createCollection(collectionName, {
  vectors: { size: 384, distance: 'Cosine' }
});

// Sample documentation chunks
const docs = [
  { id: 1, text: 'React hooks allow you to use state in functional components', category: 'react' },
  { id: 2, text: 'Express middleware functions have access to req, res, and next', category: 'express' },
  { id: 3, text: 'Async/await is syntactic sugar over promises for cleaner code', category: 'javascript' },
];

// Generate embeddings and upsert
const points = await Promise.all(
  docs.map(async (doc) => {
    const output = await embedder(doc.text, { pooling: 'mean', normalize: true });
    return {
      id: doc.id,
      vector: Array.from(output.data),
      payload: doc
    };
  })
);

await client.upsert(collectionName, { points });

// Semantic search query
const query = 'How do I manage component state?';
const queryEmbedding = await embedder(query, { pooling: 'mean', normalize: true });

const searchResults = await client.search(collectionName, {
  vector: Array.from(queryEmbedding.data),
  limit: 3,
  with_payload: true
});

searchResults.forEach(hit => {
  console.log(`[${hit.score.toFixed(3)}] ${hit.payload.text}`);
});
```

## ⚖️ When To Use / When To Avoid

**Use vector databases when:**
- You need semantic search (documentation, support tickets, product catalogs)
- Building RAG systems for LLMs (retrieval-augmented generation)
- Implementing recommendation engines based on content similarity
- Searching multi-modal content (images, audio, video)
- Users struggle with exact keyword matching

**Avoid vector databases when:**
- Exact matching suffices (user IDs, email addresses, order numbers)
- Your data is purely tabular/relational with no semantic component
- You can't afford the embedding generation overhead
- Sub-millisecond query latency is critical (ANN adds overhead)
- Your dataset is tiny (<1000 items)—traditional search is simpler

## 📚 Further Reading

- [Pinecone Learning Center: Vector Database Fundamentals](https://www.pinecone.io/learn/vector-database/) — Comprehensive introduction with interactive examples
- [Qdrant Documentation: Filtering and Hybrid Search](https://qdrant.tech/documentation/concepts/filtering/) — Best practices for combining semantic and structured queries
- [Hugging Face: Sentence Transformers](https://huggingface.co/sentence-transformers) — Browse pre-trained embedding models and benchmarks
- [arXiv: Efficient and Robust Approximate Nearest Neighbor Search (2023)](https://arxiv.org/abs/1603.09320) — Deep dive into HNSW algorithm internals
-