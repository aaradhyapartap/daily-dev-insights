# 📌 Vector databases and semantic search
*October 02, 2026 · Daily Dev Insight*

## 🧠 Overview

Vector databases aren't just another NoSQL fad—they're solving a fundamental problem that traditional databases can't handle: finding things that are *similar*, not just *exact matches*. When you search for "cute puppies" and want results that also include "adorable dogs," you're dealing with semantic meaning, not string matching. Vector databases store data as high-dimensional numerical vectors (embeddings) that capture semantic relationships, enabling similarity searches that understand context and meaning.

The magic happens through embedding models that convert text, images, or audio into vectors where semantically similar items cluster together in vector space. A traditional database can tell you if two strings are identical; a vector database can tell you if two concepts are related, even if they share no keywords. This is why they've become the backbone of RAG (Retrieval Augmented Generation) systems, recommendation engines, and modern search experiences.

The trade-off? You're exchanging the precision of SQL queries for the fuzzy accuracy of similarity scores. Your database now needs to understand approximate nearest neighbor algorithms, dimension reduction, and distance metrics. But when you need semantic search, content recommendations, or document retrieval for LLMs, vector databases become indispensable infrastructure.

## 💡 Key Concepts

- **Embeddings are the foundation**: Text, images, or data are converted into dense numerical vectors (typically 384-1536 dimensions) using ML models like OpenAI's embeddings or open-source alternatives like sentence-transformers.

- **Similarity metrics matter**: Cosine similarity, Euclidean distance, and dot product each have different use cases. Cosine similarity (normalized dot product) is most common for text because it handles varying vector magnitudes gracefully.

- **Indexing strategies trade speed for accuracy**: HNSW (Hierarchical Navigable Small World) and IVF (Inverted File Index) enable fast approximate searches on millions of vectors, but you're not getting exact results—just "close enough" matches.

- **Metadata filtering is crucial**: Pure vector search isn't enough. You need hybrid queries that combine similarity search with traditional filters (date ranges, categories, user permissions) to build real applications.

- **Not all vector DBs are created equal**: Dedicated solutions like Pinecone and Weaviate offer managed scaling, while pgvector extends PostgreSQL with vector capabilities, and Qdrant provides excellent performance with local-first development.

## 🐍 Python Example

```python
from sentence_transformers import SentenceTransformer
import numpy as np
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Initialize embedding model (runs locally, no API needed)
model = SentenceTransformer('all-MiniLM-L6-v2')  # 384-dim embeddings

# Sample product descriptions
products = [
    {"id": 1, "text": "Wireless bluetooth headphones with noise cancellation", "category": "audio"},
    {"id": 2, "text": "Ergonomic mechanical keyboard for programming", "category": "input"},
    {"id": 3, "text": "Over-ear headphones with studio-quality sound", "category": "audio"},
    {"id": 4, "text": "USB-C charging cable for smartphones", "category": "accessories"},
]

# Generate embeddings for all products
embeddings = model.encode([p["text"] for p in products])

# Initialize Qdrant (using in-memory mode for demo)
client = QdrantClient(":memory:")

# Create collection with vector configuration
client.create_collection(
    collection_name="products",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

# Upload vectors with metadata
points = [
    PointStruct(id=p["id"], vector=emb.tolist(), payload=p)
    for p, emb in zip(products, embeddings)
]
client.upsert(collection_name="products", points=points)

# Semantic search query
query = "headset for music listening"
query_vector = model.encode(query)

# Search with metadata filter
results = client.search(
    collection_name="products",
    query_vector=query_vector.tolist(),
    query_filter={"must": [{"key": "category", "match": {"value": "audio"}}]},
    limit=2
)

for result in results:
    print(f"Score: {result.score:.3f} - {result.payload['text']}")
```

## 🟨 JavaScript Example

```javascript
import { QdrantClient } from '@qdrant/js-client-rest';
import { pipeline } from '@xenova/transformers';

// Initialize embedding pipeline (runs in Node.js via ONNX)
const embedder = await pipeline(
  'feature-extraction',
  'Xenova/all-MiniLM-L6-v2'
);

// Sample documentation chunks
const docs = [
  { id: 1, text: 'React hooks allow functional components to use state', topic: 'react' },
  { id: 2, text: 'Express.js is a minimal Node.js web framework', topic: 'backend' },
  { id: 3, text: 'useState and useEffect are essential React hooks', topic: 'react' },
  { id: 4, text: 'PostgreSQL supports ACID transactions', topic: 'database' },
];

// Generate embeddings
async function embed(text) {
  const output = await embedder(text, { pooling: 'mean', normalize: true });
  return Array.from(output.data);
}

const client = new QdrantClient({ url: 'http://localhost:6333' });

// Create collection
await client.createCollection('docs', {
  vectors: { size: 384, distance: 'Cosine' }
});

// Insert vectors with metadata
for (const doc of docs) {
  await client.upsert('docs', {
    points: [{
      id: doc.id,
      vector: await embed(doc.text),
      payload: doc
    }]
  });
}

// Semantic search
const query = 'How do I manage component state in React?';
const queryVector = await embed(query);

const searchResults = await client.search('docs', {
  vector: queryVector,
  filter: { must: [{ key: 'topic', match: { value: 'react' } }] },
  limit: 2
});

searchResults.forEach(hit => {
  console.log(`${hit.score.toFixed(3)}: ${hit.payload.text}`);
});
```

## ⚖️ When To Use / When To Avoid

**Use vector databases when:**
- Building semantic search over documents, code, or support tickets
- Implementing RAG systems that need relevant context retrieval for LLMs
- Creating recommendation engines based on content similarity
- Detecting duplicate or similar items (fraud detection, content moderation)

**Avoid vector databases when:**
- You need exact match queries (traditional DB indexing is faster and more accurate)
- Your data relationships are clearly structured and relational (stick with PostgreSQL)
- You're doing simple CRUD operations without similarity requirements
- Budget or latency constraints prevent embedding generation overhead

## 📚 Further Reading

- [Pinecone Learning Center: Vector Database Fundamentals](https://www.pinecone.io/learn/vector-database/)
- [PostgreSQL pgvector Extension Documentation](https://github.com/pgvector/pgvector)
- [Hugging Face: Sentence Transformers Documentation](https://www.sbert.net/)
- [Qdrant Documentation: Vector Search Best Practices](https://qdrant.tech/documentation/)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*