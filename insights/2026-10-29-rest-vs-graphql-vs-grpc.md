# 📌 REST vs GraphQL vs gRPC
*October 29, 2026 · Daily Dev Insight*

## 🧠 Overview

The API architecture wars are over, and the truth is: they all won. REST, GraphQL, and gRPC aren't competing philosophies anymore—they're specialized tools for different jobs. REST remains the pragmatic default for public APIs and simple CRUD operations. GraphQL shines when frontend teams need flexibility and want to avoid the over-fetching/under-fetching dance. gRPC dominates internal microservices where performance and type safety are non-negotiable.

The real maturity as an engineer comes from recognizing that your choice should be dictated by your constraints, not your preferences. Building a mobile app that needs to minimize round trips and payload size? GraphQL might save you. Building service-to-service communication in a polyglot microservices architecture? gRPC's binary protocol and code generation will make your life easier. Building a simple public API that developers expect to curl from the terminal? REST is still king.

The biggest mistake I see teams make is choosing the "cool" technology instead of the right one. I've watched a startup waste three months building a GraphQL API for a simple todo app, and I've seen enterprises force REST onto internal services that would've benefited from gRPC's streaming capabilities. Know your tradeoffs, measure what matters, and don't be afraid to use multiple protocols in the same system.

## 💡 Key Concepts

- **REST**: Resource-oriented architecture using HTTP verbs. Great for caching, stateless operations, and predictable CRUD. Limited by over-fetching and multiple round trips for complex data graphs.

- **GraphQL**: Query language that lets clients specify exactly what data they need. Solves over-fetching but adds complexity in caching, security (query depth attacks), and server-side implementation. The resolver pattern can lead to N+1 query problems if you're not careful.

- **gRPC**: High-performance RPC framework using Protocol Buffers for serialization. Provides bi-directional streaming, strong typing via code generation, and excellent performance. Trade-offs include limited browser support and steeper learning curve.

- **Protocol overhead**: REST sends verbose JSON over HTTP/1.1 or HTTP/2. GraphQL adds query parsing overhead but saves bandwidth on responses. gRPC uses compact binary Protocol Buffers and HTTP/2, making it 3-10x faster for microservices.

- **Developer experience matters**: REST has universal tooling. GraphQL has amazing introspection and playground tools. gRPC requires proto file sharing and code generation, which is powerful but adds friction for external consumers.

## 🐍 Python Example

```python
# Comparing three API styles for a user profile service

# 1. REST with Flask
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user_rest(user_id):
    # REST fetches everything - over-fetching is common
    user = fetch_user_from_db(user_id)
    return jsonify(user)

# 2. GraphQL with Strawberry
import strawberry
from strawberry.fastapi import GraphQLRouter

@strawberry.type
class User:
    id: int
    name: str
    email: str
    posts: list['Post']  # Lazy-loaded, only fetched if requested

@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> User:
        # Client specifies exact fields needed: { user(id: 1) { name } }
        return fetch_user_from_db(id)

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema)

# 3. gRPC with generated code
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserService(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        # Strongly-typed request/response from .proto definition
        user = fetch_user_from_db(request.user_id)
        return user_pb2.UserResponse(
            id=user.id,
            name=user.name,
            email=user.email
        )

def serve_grpc():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserService(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
```

## 🟨 JavaScript Example

```javascript
// Comparing client implementations for each protocol

// 1. REST with fetch API - simple and universal
async function fetchUserREST(userId) {
  const response = await fetch(`https://api.example.com/users/${userId}`);
  const user = await response.json();
  // Got everything, even if we only needed the name
  return user;
}

// 2. GraphQL with fetch - precise data fetching
async function fetchUserGraphQL(userId) {
  const query = `
    query GetUser($id: Int!) {
      user(id: $id) {
        name
        email
        posts { title }
      }
    }
  `;
  
  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables: { id: userId } })
  });
  
  const { data, errors } = await response.json();
  if (errors) throw new Error(errors[0].message);
  return data.user; // Only the fields we requested
}

// 3. gRPC-web (for browser clients)
import { UserServiceClient } from './generated/user_grpc_web_pb';
import { UserRequest } from './generated/user_pb';

async function fetchUserGRPC(userId) {
  const client = new UserServiceClient('https://api.example.com:50051');
  const request = new UserRequest();
  request.setUserId(userId);
  
  // Type-safe, compiled from .proto files
  return new Promise((resolve, reject) => {
    client.getUser(request, {}, (err, response) => {
      if (err) reject(err);
      else resolve({
        id: response.getId(),
        name: response.getName(),
        email: response.getEmail()
      });
    });
  });
}
```

## ⚖️ When To Use / When To Avoid

| Protocol | ✅ Use When | ❌ Avoid When |
|----------|------------|---------------|
| **REST** | Building public APIs, simple CRUD, need HTTP caching, want universal compatibility | Complex nested data fetching, need real-time updates, internal high-performance services |
| **GraphQL** | Frontend-driven development, mobile apps (reduce payload), aggregating multiple services, evolving APIs frequently | Simple APIs, team lacks GraphQL expertise, can't afford query optimization complexity |
| **gRPC** | Internal microservices, need streaming, performance-critical systems, polyglot environments | Browser-facing APIs, want curl-friendly endpoints, team unfamiliar with Protocol Buffers |

## 📚 Further Reading

- [REST API Best Practices - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) — Comprehensive guide to HTTP methods and RESTful design patterns

- [GraphQL Official Documentation](https://graphql.org/learn/) — Learn query language syntax, schema design, and resolver patterns from the source

- [gRPC Core Concepts](https://grpc.io/docs/what-is-grpc/core-concepts/) — Understanding Protocol Buffers, service definitions, and streaming modes

- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/) — Master the .proto syntax for defining efficient binary serialization schemas

- [API Gateway Patterns: REST vs GraphQL vs gRPC](https://aws.amazon.com/blogs/architecture/) — Real-world architecture considerations for choosing and combining protocols

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*