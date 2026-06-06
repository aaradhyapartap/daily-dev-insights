# 📌 Functional programming concepts
*June 06, 2026 · Daily Dev Insight*

## 🧠 Overview

Functional programming isn't just an academic exercise—it's a powerful paradigm that can dramatically improve code quality, testability, and maintainability. At its core, FP treats computation as the evaluation of mathematical functions, emphasizing immutability, pure functions, and declarative code over imperative instructions. While object-oriented programming asks "what things are," functional programming asks "what things do."

The real magic happens when you start thinking in terms of data transformations rather than state mutations. Instead of creating objects that hold state and methods that change that state, you create pure functions that take input and produce output without side effects. This shift in thinking leads to code that's easier to reason about, test, and debug. Modern languages like Python and JavaScript have embraced many functional concepts, making it easier than ever to write functional code in traditionally imperative languages.

## 💡 Key Concepts

• **Pure Functions**: Functions that always return the same output for the same input and produce no side effects—no API calls, no database writes, no global variable modifications
• **Immutability**: Data structures that cannot be changed after creation; instead of modifying existing data, you create new data structures with the desired changes
• **Higher-Order Functions**: Functions that either take other functions as arguments or return functions as results, enabling powerful composition patterns
• **Function Composition**: Building complex operations by combining simpler functions, creating a pipeline of data transformations
• **Avoiding Shared State**: Eliminating dependencies on external mutable state, making code more predictable and easier to parallelize

## 🐍 Python Example

```python
from functools import reduce
from typing import List, Callable

# Pure function for calculating discounts
def apply_discount(price: float, discount_rate: float) -> float:
    """Pure function - same input always produces same output"""
    return price * (1 - discount_rate)

# Higher-order function that returns a function
def create_tax_calculator(tax_rate: float) -> Callable[[float], float]:
    """Returns a specialized tax calculation function"""
    return lambda price: price * (1 + tax_rate)

# Immutable data processing pipeline
def process_orders(orders: List[dict], vip_discount: float = 0.1) -> dict:
    """
    Functional pipeline for processing e-commerce orders
    Each step creates new data rather than mutating existing data
    """
    
    # Pure function for transforming individual orders
    def calculate_order_total(order: dict) -> dict:
        base_price = order['price'] * order['quantity']
        
        # Apply VIP discount if applicable
        discounted_price = (apply_discount(base_price, vip_discount) 
                          if order['customer_type'] == 'vip' 
                          else base_price)
        
        # Apply tax using our higher-order function
        tax_calc = create_tax_calculator(0.08)  # 8% tax
        final_price = tax_calc(discounted_price)
        
        # Return new dict (immutable approach)
        return {**order, 'total': round(final_price, 2)}
    
    # Functional pipeline using map, filter, and reduce
    processed_orders = list(map(calculate_order_total, orders))
    
    # Calculate summary statistics functionally
    total_revenue = reduce(lambda acc, order: acc + order['total'], 
                          processed_orders, 0)
    
    vip_orders = list(filter(lambda o: o['customer_type'] == 'vip', 
                           processed_orders))
    
    return {
        'orders': processed_orders,
        'total_revenue': round(total_revenue, 2),
        'vip_order_count': len(vip_orders),
        'average_order_value': round(total_revenue / len(processed_orders), 2)
    }

# Example usage
orders_data = [
    {'id': 1, 'price': 100, 'quantity': 2, 'customer_type': 'regular'},
    {'id': 2, 'price': 50, 'quantity': 1, 'customer_type': 'vip'},
    {'id': 3, 'price': 75, 'quantity': 3, 'customer_type': 'vip'}
]

result = process_orders(orders_data)
print(f"Total Revenue: ${result['total_revenue']}")
```

## 🟨 JavaScript Example

```javascript
// Functional utilities for data transformation
const pipe = (...functions) => (value) => 
    functions.reduce((acc, fn) => fn(acc), value);

const curry = (fn) => (...args) => 
    args.length >= fn.length 
        ? fn(...args) 
        : (...nextArgs) => curry(fn)(...args, ...nextArgs);

// Pure functions for user data processing
const validateEmail = (user) => ({
    ...user,
    isValidEmail: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(user.email)
});

const calculateUserScore = (user) => {
    const baseScore = user.posts * 10 + user.comments * 2;
    const bonusMultiplier = user.isPremium ? 1.5 : 1;
    return {
        ...user,
        score: Math.round(baseScore * bonusMultiplier)
    };
};

// Curried function for filtering by minimum score
const filterByMinScore = curry((minScore, users) => 
    users.filter(user => user.score >= minScore)
);

// Higher-order function for creating specialized mappers
const createUserMapper = (transformations) => (users) =>
    users.map(user => 
        transformations.reduce((acc, transform) => transform(acc), user)
    );

// Functional composition for user processing pipeline
const processUsers = (rawUsers) => {
    // Create specialized mapper with our transformations
    const userMapper = createUserMapper([validateEmail, calculateUserScore]);
    
    // Build processing pipeline
    const pipeline = pipe(
        userMapper,                          // Transform each user
        filterByMinScore(50),               // Filter by minimum score
        (users) => users.sort((a, b) => b.score - a.score), // Sort by score
        (users) => ({                       // Create summary object
            users: users.slice(0, 10),     // Top 10 users
            totalUsers: users.length,
            averageScore: users.reduce((sum, u) => sum + u.score, 0) / users.length,
            premiumUsers: users.filter(u => u.isPremium).length
        })
    );
    
    return pipeline(rawUsers);
};

// Example usage with immutable data
const userData = [
    { id: 1, email: 'alice@example.com', posts: 15, comments: 45, isPremium: true },
    { id: 2, email: 'invalid-email', posts: 8, comments: 20, isPremium: false },
    { id: 3, email: 'bob@example.com', posts: 25, comments: 60, isPremium: true },
    { id: 4, email: 'charlie@example.com', posts: 3, comments: 10, isPremium: false }
];

const result = processUsers(userData);
console.log(`Processed ${result.totalUsers} qualifying users`);
console.log(`Average score: ${result.averageScore.toFixed(1)}`);
```

## ⚖️ When To Use / When To Avoid

**✅ Use Functional Programming When:**
• Data transformation pipelines (ETL, API processing, analytics)
• Complex business logic that needs to be easily testable
• Concurrent or parallel processing requirements
• Mathematical computations or algorithm implementations

**❌ Avoid When:**
• Performance-critical code with tight memory constraints
• Heavy UI interaction logic requiring frequent state updates
• Integration with heavily object-oriented frameworks
• Simple CRUD operations where the overhead isn't justified

## 📚 Further Reading

• [Python Functional Programming HOWTO](https://docs.python.org/3/howto/functional.