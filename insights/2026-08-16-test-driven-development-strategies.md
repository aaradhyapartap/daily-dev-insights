# 📌 Test-driven development strategies
*August 16, 2026 · Daily Dev Insight*

## 🧠 Overview

Test-Driven Development (TDD) isn't just about writing tests first—it's a design methodology that fundamentally changes how you approach problem-solving. The classic Red-Green-Refactor cycle forces you to think about your API before implementation, leading to more modular, testable code. When done right, TDD acts as a safety net that encourages bold refactoring and catches regressions before they reach production.

The biggest misconception about TDD is that it slows you down. In reality, it shifts when you pay the cost of debugging. Instead of spending hours in a debugger tracking down mysterious bugs in production, you catch issues immediately when the tests fail. This tight feedback loop means you're always working with a known-good state, and you can refactor with confidence knowing your tests will catch any breaking changes.

The key to successful TDD is starting small and focusing on behavior, not implementation. Write the simplest test that fails, write just enough code to make it pass, then improve the design. This discipline prevents over-engineering and keeps your codebase lean and focused on actual requirements rather than hypothetical future needs.

## 💡 Key Concepts

- **Red-Green-Refactor**: Write a failing test (Red), implement the minimal code to pass (Green), then improve the design (Refactor). This cycle keeps you focused and prevents scope creep.

- **Test one behavior at a time**: Each test should verify a single, specific behavior. This makes failures easier to diagnose and tests easier to maintain when requirements change.

- **Inside-out vs Outside-in**: Inside-out starts with low-level components and builds up; outside-in starts with acceptance tests and works inward. Choose based on your problem—use outside-in for user-facing features, inside-out for utilities.

- **Mock strategically, not religiously**: Over-mocking couples tests to implementation details. Mock external dependencies and I/O, but prefer real objects for internal collaborators to keep tests robust during refactoring.

- **Test behavior, not implementation**: Tests should verify what the code does, not how it does it. This keeps tests stable when you refactor internal logic.

## 🐍 Python Example

```python
import pytest
from datetime import datetime, timedelta

# Step 1: Write the failing test first
class TestSubscriptionService:
    def test_trial_subscription_expires_after_14_days(self):
        # Arrange
        service = SubscriptionService()
        user_id = "user_123"
        start_date = datetime(2026, 8, 1)
        
        # Act
        subscription = service.create_trial(user_id, start_date)
        
        # Assert
        assert subscription.is_active(start_date + timedelta(days=13))
        assert not subscription.is_active(start_date + timedelta(days=14))
        assert subscription.status == "trial"
    
    def test_active_subscription_allows_premium_features(self):
        service = SubscriptionService()
        user_id = "user_456"
        
        subscription = service.create_paid(user_id, months=1)
        
        assert subscription.has_access("premium_analytics")
        assert subscription.has_access("priority_support")

# Step 2: Implement minimal code to pass the tests
class Subscription:
    def __init__(self, user_id, start_date, expiry_date, status):
        self.user_id = user_id
        self.start_date = start_date
        self.expiry_date = expiry_date
        self.status = status
    
    def is_active(self, check_date):
        return check_date < self.expiry_date
    
    def has_access(self, feature):
        return self.status in ["trial", "paid"]

class SubscriptionService:
    def create_trial(self, user_id, start_date):
        expiry = start_date + timedelta(days=14)
        return Subscription(user_id, start_date, expiry, "trial")
    
    def create_paid(self, user_id, months):
        start = datetime.now()
        expiry = start + timedelta(days=30 * months)
        return Subscription(user_id, start, expiry, "paid")
```

## 🟨 JavaScript Example

```javascript
// Step 1: Write the failing test (using Jest)
describe('ShoppingCart', () => {
  test('applies 10% discount when total exceeds $100', () => {
    // Arrange
    const cart = new ShoppingCart();
    cart.addItem({ name: 'Laptop', price: 120 });
    
    // Act
    const total = cart.getTotal();
    
    // Assert
    expect(total).toBe(108); // 120 - 10% discount
  });
  
  test('does not apply discount for totals under $100', () => {
    const cart = new ShoppingCart();
    cart.addItem({ name: 'Mouse', price: 50 });
    
    expect(cart.getTotal()).toBe(50);
  });
  
  test('combines multiple items before calculating discount', () => {
    const cart = new ShoppingCart();
    cart.addItem({ name: 'Keyboard', price: 60 });
    cart.addItem({ name: 'Monitor', price: 50 });
    
    // Total: 110, with 10% discount = 99
    expect(cart.getTotal()).toBe(99);
  });
});

// Step 2: Implement minimal code to pass
class ShoppingCart {
  constructor() {
    this.items = [];
  }
  
  addItem(item) {
    this.items.push(item);
  }
  
  getTotal() {
    const subtotal = this.items.reduce((sum, item) => sum + item.price, 0);
    
    // Apply 10% discount if subtotal exceeds $100
    if (subtotal > 100) {
      return subtotal * 0.9;
    }
    
    return subtotal;
  }
}

module.exports = ShoppingCart;
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Building complex business logic where correctness is critical
- Working on long-lived codebases that need to support refactoring
- API or library development where behavior contracts matter
- When collaborating with teams who need clear requirements

**❌ When To Avoid:**
- Early prototyping or throwaway spikes where you're exploring solutions
- Simple CRUD operations with well-established patterns
- UI layout and styling (use visual regression testing instead)
- When you're already late and need to catch up (though this often backfires)

## 📚 Further Reading

- [Test-Driven Development by Example - Kent Beck's foundational book](https://www.oreilly.com/library/view/test-driven-development/0321146530/)
- [Pytest Documentation - Testing best practices](https://docs.pytest.org/en/stable/goodpractices.html)
- [Jest Testing Framework - JavaScript testing guide](https://jestjs.io/docs/getting-started)
- [Martin Fowler on Mocks and Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- [Google Testing Blog - Test behavior, not implementation](https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html)

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*