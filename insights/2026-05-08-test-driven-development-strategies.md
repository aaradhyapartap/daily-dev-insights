# 📌 Test-driven development strategies
*May 08, 2026 · Daily Dev Insight*

## 🧠 Overview

Test-driven development (TDD) isn't just about writing tests first—it's about fundamentally changing how you think about code design. The Red-Green-Refactor cycle forces you to consider the API and behavior before implementation details, leading to more focused, testable code. What makes TDD powerful isn't the testing itself, but how it drives better architectural decisions through rapid feedback loops.

The biggest misconception about TDD is that it slows development. In reality, TDD frontloads the thinking process. You're forced to clarify requirements, define clear interfaces, and catch integration issues early. The initial investment pays dividends when you're confidently refactoring months later, knowing your test suite will catch regressions. Modern TDD strategies go beyond unit tests—they incorporate integration tests, property-based testing, and contract testing to create comprehensive safety nets.

## 💡 Key Concepts

• **Red-Green-Refactor Cycle**: Write a failing test (Red), implement minimal code to pass (Green), then improve the design (Refactor) while maintaining all tests
• **Test First, Design Second**: Tests act as executable specifications that drive API design and reveal coupling issues before they become architectural debt
• **Triangulation Strategy**: Use multiple test cases to guide generalization—don't write generic solutions until you have at least two examples proving the pattern
• **Inside-Out vs Outside-In**: Start with domain logic (inside-out) or user-facing behavior (outside-in) depending on whether you're exploring or implementing known requirements
• **Test Doubles Hierarchy**: Prefer real objects, then fakes, then stubs, and mocks only when necessary—over-mocking leads to brittle tests that don't catch integration bugs

## 🐍 Python Example

```python
import pytest
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class Task:
    id: str
    title: str
    completed: bool = False
    due_date: Optional[datetime] = None

class TaskManager:
    def __init__(self):
        self._tasks: List[Task] = []
        self._next_id = 1
    
    def add_task(self, title: str, due_date: Optional[datetime] = None) -> str:
        task_id = str(self._next_id)
        self._next_id += 1
        task = Task(id=task_id, title=title, due_date=due_date)
        self._tasks.append(task)
        return task_id
    
    def complete_task(self, task_id: str) -> bool:
        task = self._find_task(task_id)
        if task:
            task.completed = True
            return True
        return False
    
    def get_overdue_tasks(self) -> List[Task]:
        now = datetime.now()
        return [task for task in self._tasks 
                if not task.completed and task.due_date and task.due_date < now]
    
    def _find_task(self, task_id: str) -> Optional[Task]:
        return next((task for task in self._tasks if task.id == task_id), None)

# TDD Tests - These would be written FIRST
class TestTaskManager:
    def test_add_task_returns_unique_id(self):
        manager = TaskManager()
        id1 = manager.add_task("Buy groceries")
        id2 = manager.add_task("Walk dog")
        assert id1 != id2
    
    def test_complete_nonexistent_task_returns_false(self):
        manager = TaskManager()
        assert manager.complete_task("nonexistent") == False
    
    def test_overdue_tasks_excludes_completed_tasks(self):
        manager = TaskManager()
        yesterday = datetime.now() - timedelta(days=1)
        task_id = manager.add_task("Overdue task", yesterday)
        manager.complete_task(task_id)
        assert len(manager.get_overdue_tasks()) == 0
```

## 🟨 JavaScript Example

```javascript
// TDD approach: Tests written first, implementation follows
class PriceCalculator {
  constructor(taxRate = 0.08) {
    this.taxRate = taxRate;
    this.discountRules = [];
  }
  
  addDiscountRule(minQuantity, discountPercent) {
    this.discountRules.push({ minQuantity, discountPercent });
    // Sort by quantity descending to apply best discount first
    this.discountRules.sort((a, b) => b.minQuantity - a.minQuantity);
  }
  
  calculateTotal(items) {
    if (!Array.isArray(items) || items.length === 0) {
      return 0;
    }
    
    const subtotal = items.reduce((sum, item) => {
      return sum + (item.price * item.quantity);
    }, 0);
    
    const totalQuantity = items.reduce((sum, item) => sum + item.quantity, 0);
    const discount = this._calculateDiscount(subtotal, totalQuantity);
    const discountedSubtotal = subtotal - discount;
    const tax = discountedSubtotal * this.taxRate;
    
    return Math.round((discountedSubtotal + tax) * 100) / 100; // Round to 2 decimals
  }
  
  _calculateDiscount(subtotal, totalQuantity) {
    const applicableRule = this.discountRules.find(rule => 
      totalQuantity >= rule.minQuantity
    );
    
    return applicableRule ? subtotal * (applicableRule.discountPercent / 100) : 0;
  }
}

// Tests that drove the implementation above
describe('PriceCalculator', () => {
  test('returns zero for empty cart', () => {
    const calculator = new PriceCalculator();
    expect(calculator.calculateTotal([])).toBe(0);
  });
  
  test('calculates tax on single item', () => {
    const calculator = new PriceCalculator(0.10); // 10% tax
    const items = [{ price: 10.00, quantity: 1 }];
    expect(calculator.calculateTotal(items)).toBe(11.00);
  });
  
  test('applies quantity discount before tax', () => {
    const calculator = new PriceCalculator(0.10);
    calculator.addDiscountRule(5, 20); // 20% off for 5+ items
    
    const items = [
      { price: 10.00, quantity: 3 },
      { price: 5.00, quantity: 2 }
    ];
    // Subtotal: 40, Discount: 8, Taxable: 32, Tax: 3.20, Total: 35.20
    expect(calculator.calculateTotal(items)).toBe(35.20);
  });
});
```

## ⚖️ When To Use / When To Avoid

**✅ Use TDD When:**
- Building business logic with clear, testable requirements
- Working on unfamiliar codebases where tests provide safety nets
- Creating APIs or libraries that other developers will consume
- Requirements are stable enough to write meaningful tests

**❌ Avoid TDD When:**
- Prototyping or exploring unknown problem domains
- Working with heavily UI-dependent features (use BDD instead)
- Dealing with legacy systems that require significant setup to test
- Time-critical bug fixes where manual verification is faster

## 📚 Further Reading

• [Python Testing 101 - Real Python Guide](https://realpython.com/python-testing/)
• [JavaScript Testing Best Practices - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Cross_browser_testing/Testing_strategies)
• [Test-Driven Development by Example - Kent Beck](https://www.oreilly.com/library/view/test-driven-development/0321146530/)
• [Growing Object-Oriented Software, Guided by Tests](https://www.growing-object-oriented-