# 📌 Finite state machines in software
*May 29, 2026 · Daily Dev Insight*

## 🧠 Overview

Finite state machines (FSMs) are one of those computer science concepts that sound intimidating but solve real problems we encounter daily. At their core, FSMs model systems that can exist in a limited number of states and transition between them based on specific inputs or events. Think of them as decision trees with memory—they remember where they've been and use that context to determine what happens next.

The beauty of FSMs lies in their predictability and testability. Unlike sprawling conditional logic that grows organically (and chaotically) over time, state machines force you to explicitly define all possible states and transitions upfront. This makes complex business logic easier to reason about, debug, and modify. Whether you're building a user authentication flow, a game AI system, or managing WebSocket connection states, FSMs provide a structured approach that scales better than nested if-statements.

Modern applications are full of implicit state machines—we just don't always recognize them. That loading spinner? It's managing transitions between idle, loading, success, and error states. Your CI/CD pipeline? It's a state machine tracking builds through queued, running, failed, and deployed states. Making these implicit machines explicit through proper FSM implementation leads to more robust, maintainable code.

## 💡 Key Concepts

• **States are exclusive**: An FSM exists in exactly one state at any given time, eliminating ambiguous system conditions that cause bugs
• **Transitions are explicit**: Every possible state change must be defined upfront, preventing unexpected behavior and making edge cases visible
• **Events drive transitions**: State changes happen in response to specific triggers (user input, API responses, timers), not arbitrary code execution
• **Guards and actions**: Transitions can include conditional logic (guards) and side effects (actions) that execute during state changes
• **Hierarchical states**: Complex FSMs can have nested states and parallel regions, allowing you to model sophisticated systems without explosion of complexity

## 🐍 Python Example

```python
from enum import Enum
from typing import Dict, Callable, Optional

class OrderState(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed" 
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class OrderFSM:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.state = OrderState.PENDING
        self.transitions: Dict[OrderState, Dict[str, OrderState]] = {
            OrderState.PENDING: {
                "confirm": OrderState.CONFIRMED,
                "cancel": OrderState.CANCELLED
            },
            OrderState.CONFIRMED: {
                "ship": OrderState.SHIPPED,
                "cancel": OrderState.CANCELLED
            },
            OrderState.SHIPPED: {
                "deliver": OrderState.DELIVERED
            },
            # Terminal states have no transitions
            OrderState.DELIVERED: {},
            OrderState.CANCELLED: {}
        }
    
    def trigger(self, event: str) -> bool:
        """Attempt to trigger a state transition"""
        if event in self.transitions[self.state]:
            old_state = self.state
            self.state = self.transitions[self.state][event]
            self._on_transition(old_state, event, self.state)
            return True
        return False
    
    def _on_transition(self, from_state: OrderState, event: str, to_state: OrderState):
        """Handle side effects during transitions"""
        print(f"Order {self.order_id}: {from_state.value} -> {to_state.value} (via {event})")
        
        # Execute state-specific actions
        if to_state == OrderState.SHIPPED:
            self._send_tracking_email()
        elif to_state == OrderState.CANCELLED:
            self._process_refund()
    
    def _send_tracking_email(self):
        print(f"📧 Sending tracking email for order {self.order_id}")
    
    def _process_refund(self):
        print(f"💰 Processing refund for order {self.order_id}")

# Usage example
order = OrderFSM("ORD-12345")
order.trigger("confirm")  # pending -> confirmed
order.trigger("ship")     # confirmed -> shipped  
order.trigger("deliver")  # shipped -> delivered
```

## 🟨 JavaScript Example

```javascript
class ConnectionFSM extends EventTarget {
    constructor(websocketUrl) {
        super();
        this.url = websocketUrl;
        this.state = 'disconnected';
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 3;
        
        // Define valid transitions for each state
        this.transitions = {
            'disconnected': ['connecting'],
            'connecting': ['connected', 'error'],
            'connected': ['disconnecting', 'error'],
            'disconnecting': ['disconnected'],
            'error': ['connecting', 'disconnected']
        };
    }
    
    async connect() {
        if (!this._canTransition('connecting')) return false;
        
        this._setState('connecting');
        try {
            this.ws = new WebSocket(this.url);
            
            this.ws.onopen = () => {
                this.reconnectAttempts = 0;
                this._setState('connected');
            };
            
            this.ws.onerror = () => this._setState('error');
            this.ws.onclose = () => this._handleConnectionLoss();
            
            return true;
        } catch (error) {
            this._setState('error');
            return false;
        }
    }
    
    disconnect() {
        if (!this._canTransition('disconnecting')) return;
        
        this._setState('disconnecting');
        if (this.ws) {
            this.ws.close();
            this.ws = null;
        }
        this._setState('disconnected');
    }
    
    _handleConnectionLoss() {
        this._setState('error');
        
        // Auto-reconnect logic based on current state
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            setTimeout(() => this.connect(), 1000 * this.reconnectAttempts);
        } else {
            this._setState('disconnected');
            this.dispatchEvent(new CustomEvent('max-retries-exceeded'));
        }
    }
    
    _canTransition(newState) {
        return this.transitions[this.state]?.includes(newState) || false;
    }
    
    _setState(newState) {
        const oldState = this.state;
        this.state = newState;
        console.log(`Connection: ${oldState} -> ${newState}`);
        this.dispatchEvent(new CustomEvent('state-change', { 
            detail: { from: oldState, to: newState } 
        }));
    }
}
```

## ⚖️ When To Use / When To Avoid

**✅ Use FSMs when:**
- You have complex conditional logic with multiple interconnected states
- System behavior needs to be predictable and easily testable
- You need to prevent invalid state combinations or transitions
- Building user workflows, game mechanics, or protocol implementations
- Debugging state-related bugs is becoming difficult

**❌ Avoid FSMs when:**
- Simple linear processes with minimal branching logic
- Performance is critical and state overhead matters
- The problem domain doesn't naturally map to discrete states
- Your team lacks familiarity and simpler solutions exist
- States and transitions are frequently changing during development

## 📚 Further Reading

• [Python transitions library documentation](https://github.com/pytransitions/transitions) - Powerful FSM library with hierarchical states and async support
• [XState JavaScript library guide](https://xstate.js.org/docs/) - Comprehensive statechart library for complex frontend state management  
• [Finite State Machines in React](https://css-tricks.com/finite-state-machines-in-react/) - Practical examples of FSMs in modern React applications
• [Martin Fowler on State Machines](https://martinfowler.com/articles