# 📌 Test-driven development strategies
*June 27, 2026 · Daily Dev Insight*

## 🧠 Overview

Test-Driven Development (TDD) isn't just about writing tests first—it's a fundamental shift in how you think about software design. The classic Red-Green-Refactor cycle forces you to clarify requirements before implementation, leading to more modular, testable code. By writing a failing test first, you're essentially creating a specification for what your code should do, not what it currently does. This subtle distinction prevents the common trap of writing tests that simply verify existing behavior rather than desired behavior.

The real power of TDD emerges when you realize it's a design methodology disguised as a testing practice. When you struggle to write a test for a function, that's not a testing problem—it's a signal that your design has tight coupling or unclear responsibilities. TDD practitioners often find themselves writing smaller, more focused functions with explicit dependencies, not because some style guide told them to, but because the tests demanded it.

However, TDD isn't a silver bullet, and applying it dogmatically to every situation can slow teams down unnecessarily. The key is understanding when the upfront investment in test-first development pays dividends versus when exploratory coding or prototyping makes more sense. Successful teams use TDD as a tool, not a religion, applying it strategically to critical business logic while being more pragmatic with UI code or rapidly changing experimental features.

## 💡 Key Concepts

- **Red-Green-Refactor Cycle**: Write a failing test (Red), make it pass with minimal code (Green), then improve the implementation without changing behavior (Refactor). Each phase has a distinct mindset and purpose.

- **Triangulation**: Start with the simplest test case, then add more test cases to force your implementation to become more general. Don't over-engineer solutions until tests prove you need the complexity.

- **Test Isolation**: Each test should be completely independent, with its own setup and teardown. Tests that depend on execution order or shared state are maintenance nightmares waiting to happen.

- **Inside-Out vs Outside-In**: Inside-out TDD starts with low-level units and builds up; outside-in starts with acceptance tests and drives out the necessary components. Choose based on how well you understand the domain.

- **Mocking Strategy**: Use mocks sparingly and primarily for external dependencies (databases, APIs). Over-mocking leads to brittle tests that break on refactoring and don't catch real integration issues.

## 🐍 Python Example

```python
# Shopping cart with TDD approach - test first!
import pytest
from decimal import Decimal

class TestShoppingCart:
    def test_empty_cart_total_is_zero(self):
        cart = ShoppingCart()
        assert cart.total() == Decimal('0.00')
    
    def test_add_single_item(self):
        cart = ShoppingCart()
        cart.add_item('Apple', Decimal('1.50'), quantity=2)
        assert cart.total() == Decimal('3.00')
    
    def test_bulk_discount_applied_when_quantity_exceeds_threshold(self):
        cart = ShoppingCart()
        # 10% discount when buying 5+ of same item
        cart.add_item('Orange', Decimal('2.00'), quantity=5)
        assert cart.total() == Decimal('9.00')  # 10.00 - 10%

# Now implement to make tests pass
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def add_item(self, name: str, price: Decimal, quantity: int = 1):
        self.items.append({
            'name': name,
            'price': price,
            'quantity': quantity
        })
    
    def total(self) -> Decimal:
        subtotal = Decimal('0.00')
        for item in self.items:
            item_total = item['price'] * item['quantity']
            # Apply bulk discount
            if item['quantity'] >= 5:
                item_total *= Decimal('0.9')
            subtotal += item_total
        return subtotal.quantize(Decimal('0.01'))
```

## 🟨 JavaScript Example

```javascript
// User authentication service - TDD style
const bcrypt = require('bcrypt');

// Tests written FIRST
describe('AuthService', () => {
  let authService;
  
  beforeEach(() => {
    authService = new AuthService();
  });
  
  test('registers new user with hashed password', async () => {
    const user = await authService.register('alice@example.com', 'secret123');
    
    expect(user.email).toBe('alice@example.com');
    expect(user.password).not.toBe('secret123'); // Should be hashed
    expect(await bcrypt.compare('secret123', user.password)).toBe(true);
  });
  
  test('throws error when registering duplicate email', async () => {
    await authService.register('bob@example.com', 'password');
    
    await expect(
      authService.register('bob@example.com', 'different')
    ).rejects.toThrow('Email already registered');
  });
});

// Implementation follows tests
class AuthService {
  constructor() {
    this.users = new Map(); // In-memory store for demo
  }
  
  async register(email, password) {
    if (this.users.has(email)) {
      throw new Error('Email already registered');
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = { email, password: hashedPassword };
    this.users.set(email, user);
    
    return user;
  }
}

module.exports = AuthService;
```

## ⚖️ When To Use / When To Avoid

**Use TDD when:**
- Building core business logic with clear requirements
- Working on financial, healthcare, or safety-critical systems
- Refactoring legacy code (write characterization tests first)
- You're unfamiliar with the problem domain and need to think through edge cases
- Building libraries or APIs that others depend on

**Avoid or adapt TDD when:**
- Prototyping or exploring uncertain requirements (spike first, then TDD)
- Dealing with heavy UI work where visual feedback is more valuable than tests
- The testing infrastructure overhead exceeds the value (simple scripts, one-off tools)
- You're in a startup's early days optimizing for learning over stability
- External dependencies make testing prohibitively complex without clear benefit

## 📚 Further Reading

- [Test-Driven Development by Example](https://www.oreilly.com/library/view/test-driven-development/0321146530/) – Kent Beck's seminal work on TDD fundamentals and philosophy

- [Python unittest documentation](https://docs.python.org/3/library/unittest.html) – Official guide to Python's built-in testing framework with best practices

- [Jest Testing Framework](https://jestjs.io/docs/getting-started) – Comprehensive JavaScript testing with excellent mocking capabilities

- [Martin Fowler on Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) – Critical distinctions between test doubles that shape TDD approach

- [Growing Object-Oriented Software, Guided by Tests](https://www.growing-object-oriented-software.com/) – Advanced outside-in TDD with mockist approach

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*