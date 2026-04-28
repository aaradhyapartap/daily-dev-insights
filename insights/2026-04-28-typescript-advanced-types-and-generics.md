# 📌 TypeScript advanced types and generics
*April 28, 2026 · Daily Dev Insight*

## 🧠 Overview

TypeScript's advanced types and generics are where the language truly shines, transforming JavaScript from a loosely-typed scripting language into a powerful, type-safe development platform. While basic types help catch obvious errors, advanced types like conditional types, mapped types, and utility types enable you to express complex relationships between data structures that evolve with your code.

Generics, in particular, are the cornerstone of reusable, type-safe code. They allow you to write functions, classes, and interfaces that work with multiple types while maintaining type safety throughout your application. Think of generics as type-level functions—they take type parameters and return new, specialized types. When combined with TypeScript's advanced type operators, you can create incredibly expressive APIs that guide developers toward correct usage while preventing entire classes of runtime errors.

The real power emerges when you start composing these features together. Conditional types let you create type-level logic, mapped types transform object shapes, and template literal types enable sophisticated string manipulation at the type level. This isn't just academic exercise—these features enable libraries like React Query, Prisma, and tRPC to provide exceptional developer experiences with rock-solid type safety.

## 💡 Key Concepts

• **Generic constraints** use `extends` to limit type parameters, ensuring they have required properties or methods while maintaining flexibility
• **Conditional types** with `T extends U ? X : Y` syntax enable type-level branching logic, perfect for creating adaptive APIs
• **Mapped types** transform existing object types by iterating over their keys, essential for utility types like `Partial<T>` and `Required<T>`
• **Template literal types** combine string literals with type parameters, enabling type-safe string manipulation and API route generation
• **Utility types** like `Pick`, `Omit`, `Record`, and `ReturnType` solve common type transformation patterns without custom implementations

## 🐍 Python Example

```python
from typing import TypeVar, Generic, Protocol, Union, List, Dict, Any
from dataclasses import dataclass
from abc import abstractmethod

# Python's equivalent to TypeScript generics using TypeVar
T = TypeVar('T')
K = TypeVar('K')
V = TypeVar('V')

class Serializable(Protocol):
    """Protocol defining what types can be cached (similar to TS interface)"""
    @abstractmethod
    def to_dict(self) -> Dict[str, Any]:
        pass

# Generic cache class with protocol constraint
class TypedCache(Generic[T]):
    def __init__(self) -> None:
        self._storage: Dict[str, T] = {}
    
    def set(self, key: str, value: T) -> None:
        self._storage[key] = value
    
    def get(self, key: str) -> Union[T, None]:
        return self._storage.get(key)
    
    def get_all(self) -> List[T]:
        return list(self._storage.values())

@dataclass
class User:
    id: int
    name: str
    email: str
    
    def to_dict(self) -> Dict[str, Any]:
        return {"id": self.id, "name": self.name, "email": self.email}

@dataclass 
class Product:
    sku: str
    name: str
    price: float
    
    def to_dict(self) -> Dict[str, Any]:
        return {"sku": self.sku, "name": self.name, "price": self.price}

# Usage with type safety
user_cache: TypedCache[User] = TypedCache()
product_cache: TypedCache[Product] = TypedCache()

user_cache.set("user_1", User(1, "Alice", "alice@example.com"))
product_cache.set("prod_1", Product("ABC123", "Laptop", 999.99))

# Type checker ensures we get back the correct types
user: Union[User, None] = user_cache.get("user_1")
product: Union[Product, None] = product_cache.get("prod_1")
```

## 🟨 JavaScript Example

```javascript
// Advanced TypeScript generics and conditional types
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// Conditional type that extracts data type from API response
type ExtractData<T> = T extends ApiResponse<infer U> ? U : never;

// Mapped type that makes all properties optional except specified keys
type PartialExcept<T, K extends keyof T> = Partial<T> & Pick<T, K>;

// Template literal type for building API endpoints
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = `/api/${string}`;
type ApiRoute<M extends HttpMethod, E extends Endpoint> = `${M} ${E}`;

// Generic API client with advanced type constraints
class TypedApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  // Generic method with conditional return types
  async request<T, M extends HttpMethod>(
    method: M,
    endpoint: Endpoint,
    data?: M extends 'GET' ? never : unknown
  ): Promise<ApiResponse<T>> {
    const config: RequestInit = {
      method,
      headers: { 'Content-Type': 'application/json' },
    };

    // Only add body for non-GET requests (enforced by types)
    if (method !== 'GET' && data) {
      config.body = JSON.stringify(data);
    }

    const response = await fetch(`${this.baseUrl}${endpoint}`, config);
    return response.json() as Promise<ApiResponse<T>>;
  }

  // Utility method using our conditional type
  async getData<T extends ApiResponse<any>>(
    response: Promise<T>
  ): Promise<ExtractData<T>> {
    const result = await response;
    return result.data as ExtractData<T>;
  }
}

// Usage with full type safety
interface User {
  id: number;
  name: string;
  email: string;
}

type CreateUserRequest = PartialExcept<User, 'name' | 'email'>;

const client = new TypedApiClient('https://api.example.com');

// TypeScript enforces correct method/data combinations
const userResponse = client.request<User, 'GET'>('GET', '/api/users/1');
const createResponse = client.request<User, 'POST'>('POST', '/api/users', {
  name: 'John Doe',
  email: 'john@example.com'
} as CreateUserRequest);

// Extract data with preserved types
client.getData(userResponse).then(user => {
  console.log(user.name); // TypeScript knows this is a User
});
```

## ⚖️ When To Use / When To Avoid

**✅ Use Advanced Types When:**
• Building reusable libraries or components that need to work with multiple types
• Creating type-safe APIs where incorrect usage should be prevented at compile time
• Working with complex data transformations where runtime errors are costly
• You need to express relationships between types that change together

**❌ Avoid When:**
• Writing simple scripts or prototypes where type safety overhead exceeds benefits
• Team members aren't comfortable with advanced TypeScript concepts
• You're prioritizing development speed over long-term maintainability
• The type complexity makes code harder to understand than the bugs it prevents

## 📚 Further Reading

• [TypeScript Handbook: Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) - Official comprehensive guide to generic types
• [Advanced TypeScript Types Cheat Sheet](https://www.typescriptlang.org/docs/handbook/utility-types.html) - Complete reference for utility types
• [Type-Level TypeScript Book](https://type-level-typescript.com/) - Deep dive into advanced type system patterns
• [TypeScript Conditional Types Guide](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) - Master type-level programming
• [Template Literal Types Documentation](https://