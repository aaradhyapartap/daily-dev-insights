# 📌 Vector databases and semantic search
*May 05, 2026 · Daily Dev Insight*

## 🧠 Overview

Vector databases have evolved from ML research curiosity to production necessity, fundamentally changing how we search and retrieve information. Unlike traditional keyword-based search that matches exact terms, vector databases store high-dimensional numerical representations (embeddings) of data, enabling semantic similarity searches that understand context and meaning.

The magic happens in the embedding space—a mathematical representation where semantically similar content clusters together. When you search for "happy dog," you'll find results about "joyful puppies" even without exact keyword matches. This isn't just better search; it's the foundation for RAG systems, recommendation engines, and AI applications that need to understand relationships between data points rather than just exact matches.

## 💡 Key Concepts

• **Embeddings are the foundation**: Text, images, or any data gets converted to high-dimensional vectors (typically 384-1536 dimensions) where semantic similarity translates to geometric proximity
• **Approximate Nearest Neighbor (ANN) search**: Vector DBs use algorithms like HNSW or IVF to quickly find similar vectors without comparing every single one—crucial for performance at scale
• **Hybrid search patterns**: The best production systems combine vector similarity with traditional filters (date ranges, categories, exact matches) for precise control
• **Embedding model choice matters**: Different models excel at different domains—sentence-transformers for general text, specialized models for code, medical content, or multilingual scenarios
• **Distance metrics define similarity**: Cosine similarity, Euclidean distance, or dot product each capture different aspects of semantic relationships

## 🐍 Python Example

```python
import chromadb
from sentence_transformers import SentenceTransformer
import uuid

# Initialize embedding model and vector database
model = SentenceTransformer('all-MiniLM-L6-v2')
client = chromadb.PersistentClient(path="./vector_db")
collection = client.get_or_create_collection("code_snippets")

class CodeSearchEngine:
    def __init__(self, collection, model):
        self.collection = collection
        self.model = model
    
    def add_documents(self, docs_with_metadata):
        """Add code snippets with metadata to vector DB"""
        texts = [doc['content'] for doc in docs_with_metadata]
        embeddings = self.model.encode(texts).tolist()
        
        ids = [str(uuid.uuid4()) for _ in docs_with_metadata]
        metadatas = [{k: v for k, v in doc.items() if k != 'content'} 
                    for doc in docs_with_metadata]
        
        self.collection.add(
            embeddings=embeddings,
            documents=texts,
            metadatas=metadatas,
            ids=ids
        )
    
    def semantic_search(self, query, language=None, n_results=5):
        """Search for semantically similar code snippets"""
        query_embedding = self.model.encode([query]).tolist()
        
        # Combine vector similarity with metadata filtering
        where_clause = {"language": language} if language else None
        
        results = self.collection.query(
            query_embeddings=query_embedding,
            n_results=n_results,
            where=where_clause,
            include=["documents", "metadatas", "distances"]
        )
        
        return [{
            'content': doc,
            'metadata': meta,
            'similarity': 1 - distance  # Convert distance to similarity
        } for doc, meta, distance in zip(
            results['documents'][0],
            results['metadatas'][0], 
            results['distances'][0]
        )]

# Usage example
search_engine = CodeSearchEngine(collection, model)

# Add sample code snippets
code_docs = [
    {"content": "async function fetchUserData(userId) { return await api.get(`/users/${userId}`); }", 
     "language": "javascript", "topic": "api"},
    {"content": "def calculate_fibonacci(n): return n if n <= 1 else calculate_fibonacci(n-1) + calculate_fibonacci(n-2)", 
     "language": "python", "topic": "algorithms"}
]

search_engine.add_documents(code_docs)
results = search_engine.semantic_search("get user information from API", language="javascript")
```

## 🟨 JavaScript Example

```javascript
import { ChromaClient } from 'chromadb';
import axios from 'axios';

class ProductRecommendationEngine {
    constructor() {
        this.client = new ChromaClient();
        this.collection = null;
        this.apiKey = process.env.OPENAI_API_KEY;
    }

    async initialize() {
        this.collection = await this.client.getOrCreateCollection({
            name: "products",
            metadata: { "hnsw:space": "cosine" }
        });
    }

    async generateEmbedding(text) {
        try {
            const response = await axios.post(
                'https://api.openai.com/v1/embeddings',
                {
                    input: text,
                    model: "text-embedding-3-small"
                },
                {
                    headers: {
                        'Authorization': `Bearer ${this.apiKey}`,
                        'Content-Type': 'application/json'
                    }
                }
            );
            return response.data.data[0].embedding;
        } catch (error) {
            console.error('Embedding generation failed:', error);
            throw error;
        }
    }

    async addProducts(products) {
        const embeddings = [];
        const documents = [];
        const metadatas = [];
        const ids = [];

        for (const product of products) {
            // Create rich text representation for better embeddings
            const searchText = `${product.name} ${product.description} ${product.category} ${product.tags?.join(' ') || ''}`;
            
            const embedding = await this.generateEmbedding(searchText);
            
            embeddings.push(embedding);
            documents.push(searchText);
            metadatas.push({
                productId: product.id,
                name: product.name,
                category: product.category,
                price: product.price,
                rating: product.rating
            });
            ids.push(product.id.toString());
        }

        await this.collection.add({
            embeddings,
            documents,
            metadatas,
            ids
        });
    }

    async findSimilarProducts(query, options = {}) {
        const { minRating = 0, maxPrice = Infinity, category, limit = 10 } = options;
        
        const queryEmbedding = await this.generateEmbedding(query);
        
        // Build where clause for filtering
        const whereClause = {};
        if (minRating > 0) whereClause.rating = { "$gte": minRating };
        if (maxPrice < Infinity) whereClause.price = { "$lte": maxPrice };
        if (category) whereClause.category = category;

        const results = await this.collection.query({
            queryEmbeddings: [queryEmbedding],
            nResults: limit,
            where: Object.keys(whereClause).length > 0 ? whereClause : undefined,
            include: ["metadatas", "distances"]
        });

        return results.metadatas[0].map((metadata, index) => ({
            ...metadata,
            similarity: 1 - results.distances[0][index]
        }));
    }
}

// Usage
const recommender = new ProductRecommendationEngine();
await recommender.initialize();

const products = [
    { id: 1, name: "Wireless Headphones", description: "Noise-cancelling Bluetooth headphones", category: "electronics", price: 199, rating: 4.5 }
];

await recommender.addProducts(products);
const similar = await recommender.findSimilarProducts("good audio equipment", { minRating: 4.0 });
```

## ⚖️ When To Use / When To Avoid

**✅ Use vector