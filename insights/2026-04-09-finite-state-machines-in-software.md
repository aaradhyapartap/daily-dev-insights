# 📌 Finite state machines in software
*April 09, 2026 · Daily Dev Insight*

## 🧠 Overview

Finite state machines (FSMs) are one of those computer science concepts that feel academic until you encounter a problem where they're the perfect solution. At their core, FSMs model systems that can exist in a limited number of states and transition between them based on specific inputs or events. Think of a traffic light, a game character's behavior, or even a user authentication flow—each has clearly defined states and rules for moving between them.

What makes FSMs particularly powerful in software engineering is their ability to eliminate the dreaded "spaghetti code" that often emerges from complex conditional logic. Instead of nested if-else chains that become unmaintainable, FSMs provide a structured approach to state management. They make your code more predictable, testable, and easier to reason about. When you're dealing with workflows, protocols, or any system where "what happens next" depends on "where you are now," FSMs often provide the cleanest solution.

The beauty of FSMs lies in their constraint: by explicitly limiting the possible states and transitions, you're forced to think clearly about your system's behavior. This constraint becomes a feature, not a limitation, especially when building robust applications that need to handle edge cases gracefully.

## 💡 Key Concepts

• **States**: Distinct conditions or modes your system can be in (e.g., "loading", "authenticated", "error")
• **Transitions**: Rules that define how to move from one state to another based on events or conditions
• **Events/Triggers**: Inputs that cause state transitions (user actions, API responses, timeouts)
• **Actions**: Side effects that occur during state transitions (logging, API calls, UI updates)
• **Deterministic behavior**: Given a current state and input, the next state is always predictable

## 🐍 Python Example

```python
from enum import Enum
from typing import Dict, Callable, Optional

class OrderState(Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class OrderFSM:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.current_state = OrderState.PENDING
        
        # Define valid transitions
        self.transitions: Dict[OrderState, Dict[str, OrderState]] = {
            OrderState.PENDING: {"pay": OrderState.PAID, "cancel": OrderState.CANCELLED},
            OrderState.PAID: {"ship": OrderState.SHIPPED, "cancel": OrderState.CANCELLED},
            OrderState.SHIPPED: {"deliver": OrderState.DELIVERED},
            OrderState.DELIVERED: {},  # Terminal state
            OrderState.CANCELLED: {}   # Terminal state
        }
        
        # Define actions to execute on state transitions
        self.actions: Dict[str, Callable] = {
            "pay": self._process_payment,
            "ship": self._create_shipment,
            "deliver": self._send_delivery_notification,
            "cancel": self._refund_payment
        }
    
    def trigger_event(self, event: str) -> bool:
        """Attempt to trigger an event and transition states"""
        if event in self.transitions[self.current_state]:
            old_state = self.current_state
            self.current_state = self.transitions[old_state][event]
            
            # Execute associated action
            if event in self.actions:
                self.actions[event]()
            
            print(f"Order {self.order_id}: {old_state.value} → {self.current_state.value}")
            return True
        else:
            print(f"Invalid transition: cannot '{event}' from state '{self.current_state.value}'")
            return False
    
    def _process_payment(self): print(f"Processing payment for order {self.order_id}")
    def _create_shipment(self): print(f"Creating shipment for order {self.order_id}")
    def _send_delivery_notification(self): print(f"Sending delivery notification for order {self.order_id}")
    def _refund_payment(self): print(f"Processing refund for order {self.order_id}")

# Usage example
order = OrderFSM("ORD-001")
order.trigger_event("pay")      # pending → paid
order.trigger_event("ship")     # paid → shipped  
order.trigger_event("cancel")   # Invalid: can't cancel shipped orders
```

## 🟨 JavaScript Example

```javascript
class ConnectionFSM {
    constructor(endpoint) {
        this.endpoint = endpoint;
        this.currentState = 'disconnected';
        this.retryCount = 0;
        this.maxRetries = 3;
        
        // Define state machine configuration
        this.states = {
            disconnected: {
                connect: 'connecting'
            },
            connecting: {
                success: 'connected',
                failure: 'reconnecting',
                timeout: 'reconnecting'
            },
            connected: {
                disconnect: 'disconnected',
                error: 'reconnecting',
                heartbeat_fail: 'reconnecting'
            },
            reconnecting: {
                retry: 'connecting',
                give_up: 'failed'
            },
            failed: {
                reset: 'disconnected'
            }
        };
        
        this.eventHandlers = new Map();
    }
    
    // Register event listeners for state changes
    on(event, handler) {
        if (!this.eventHandlers.has(event)) {
            this.eventHandlers.set(event, []);
        }
        this.eventHandlers.get(event).push(handler);
    }
    
    emit(event, data) {
        const handlers = this.eventHandlers.get(event) || [];
        handlers.forEach(handler => handler(data));
    }
    
    async transition(event, data = {}) {
        const validTransitions = this.states[this.currentState];
        
        if (!validTransitions || !validTransitions[event]) {
            console.warn(`Invalid transition: ${event} from ${this.currentState}`);
            return false;
        }
        
        const nextState = validTransitions[event];
        const prevState = this.currentState;
        
        // Execute state-specific logic
        await this.executeStateAction(event, data);
        
        this.currentState = nextState;
        this.emit('stateChanged', { from: prevState, to: nextState, event, data });
        
        console.log(`Connection: ${prevState} → ${nextState} (${event})`);
        return true;
    }
    
    async executeStateAction(event, data) {
        // Handle side effects based on the transition
        switch (event) {
            case 'connect':
                this.retryCount = 0;
                break;
            case 'failure':
            case 'timeout':
            case 'error':
                this.retryCount++;
                setTimeout(() => {
                    const nextEvent = this.retryCount >= this.maxRetries ? 'give_up' : 'retry';
                    this.transition(nextEvent);
                }, 1000 * this.retryCount); // Exponential backoff
                break;
        }
    }
}

// Usage example
const connection = new ConnectionFSM('ws://api.example.com');
connection.on('stateChanged', ({ from, to, event }) => {
    console.log(`State changed: ${from} → ${to} via ${event}`);
});

connection.transition('connect');
connection.transition('failure');  // Will auto-retry
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
• Complex workflows with clear states (user onboarding, order processing)
• Protocol implementations (WebSocket connections, API clients)
• Game development (character AI, game states)
• Form validation with multi-step processes
• When you find yourself writing deeply nested conditional logic

**❌ When To Avoid:**
• Simple linear processes with no branching logic
• Mathematical calculations or data transformations
• When state transitions are highly dynamic