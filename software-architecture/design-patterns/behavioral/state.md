# State

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, state, state-machine, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Permitir que um objeto **altere seu comportamento quando seu estado interno muda**. O objeto parece mudar de classe.

### Problema
Um pedido passa por `Pending → Paid → Shipped → Delivered`, e pode ser `Cancelled` apenas antes do envio. Implementar isso com `if (status == ...)` em cada método (`pay`, `ship`, `cancel`) espalha as regras de transição, e cada novo estado exige mexer em todos os métodos. Transições inválidas (enviar um pedido não pago) viram bugs silenciosos.

### Estrutura
- **Context**: o objeto com estado (`Order`); delega as operações ao estado atual.
- **State**: interface com as operações que dependem do estado (`pay()`, `ship()`, `cancel()`).
- **ConcreteStates**: implementam o comportamento de cada estado e decidem a **transição** para o próximo.

Implementações modernas: *sealed types* + `match` exaustivo, enums com métodos, ou *typestate* (estados como tipos, transições inválidas não compilam).

### Trade-offs
| Prós | Contras |
|------|---------|
| Regras de cada estado ficam juntas e explícitas. | Muitas classes para máquinas pequenas (2 ou 3 estados com um `enum` bastam). |
| Elimina condicionais grandes espalhadas pelo código. | Transições ficam distribuídas entre as classes de estado; a visão geral se perde. |
| Novos estados sem mexer nos existentes (Open/Closed). | Estados precisando de dados do contexto geram acoplamento bidirecional. |
| Transições inválidas tratadas de forma consistente. | Persistir e reidratar o estado (banco, eventos) exige mapeamento extra. |

### Quando usar / evitar
- **Use** quando o comportamento muda bastante conforme o estado e as regras de transição importam: pedidos, pagamentos, conexões TCP, workflows de aprovação, circuit breakers.
- **Evite** para poucos estados com comportamento quase igual; um `enum` + tabela de transições pode ser mais claro. Para máquinas complexas, considere uma biblioteca de *state machine* com diagrama.

---

## English

### Intent
Allow an object to **alter its behavior when its internal state changes**. The object will appear to change its class.

### Problem
An order goes through `Pending → Paid → Shipped → Delivered`, and can be `Cancelled` only before shipping. Implementing this with `if (status == ...)` in every method (`pay`, `ship`, `cancel`) scatters the transition rules, and each new state requires touching every method. Invalid transitions (shipping an unpaid order) become silent bugs.

### Structure
- **Context**: the stateful object (`Order`); it delegates operations to the current state.
- **State**: an interface with state-dependent operations (`pay()`, `ship()`, `cancel()`).
- **ConcreteStates**: implement each state's behavior and decide the **transition** to the next one.

Modern implementations: *sealed types* + exhaustive `match`, enums with methods, or *typestate* (states as types, so invalid transitions don't compile).

### Trade-offs
| Pros | Cons |
|------|------|
| Each state's rules live together and are explicit. | Many classes for small machines (2 or 3 states are fine with an `enum`). |
| Eliminates large conditionals scattered around the code. | Transitions get spread across state classes; the big picture gets lost. |
| New states without touching existing ones (Open/Closed). | States needing context data create bidirectional coupling. |
| Invalid transitions are handled consistently. | Persisting and rehydrating state (database, events) needs extra mapping. |

### When to use / avoid
- **Use** when behavior changes a lot depending on state and transition rules matter: orders, payments, TCP connections, approval workflows, circuit breakers.
- **Avoid** for a few states with nearly identical behavior; an `enum` + transition table can be clearer. For complex machines, consider a state machine library with a diagram.

---

## Exemplos / Examples

Scenario: an order state machine. `pay`, `ship`, `deliver`, `cancel` are valid only in some states.

<details>
<summary><strong>Java</strong></summary>

```java
// Each state is an immutable object that returns the next state.
sealed interface OrderState permits Pending, Paid, Shipped, Delivered, Cancelled {
    default OrderState pay() { throw invalid("pay"); }
    default OrderState ship() { throw invalid("ship"); }
    default OrderState deliver() { throw invalid("deliver"); }
    default OrderState cancel() { throw invalid("cancel"); }

    private IllegalStateException invalid(String action) {
        return new IllegalStateException("cannot " + action + " when " + getClass().getSimpleName());
    }
}

record Pending() implements OrderState {
    public OrderState pay() { return new Paid(); }
    public OrderState cancel() { return new Cancelled(); }
}
record Paid() implements OrderState {
    public OrderState ship() { return new Shipped(); }
    public OrderState cancel() { System.out.println("  refunding payment"); return new Cancelled(); }
}
record Shipped() implements OrderState {
    public OrderState deliver() { return new Delivered(); }
}
record Delivered() implements OrderState {}
record Cancelled() implements OrderState {}

// Context
class Order {
    private OrderState state = new Pending();
    void pay() { state = state.pay(); }
    void ship() { state = state.ship(); }
    void deliver() { state = state.deliver(); }
    void cancel() { state = state.cancel(); }
    OrderState state() { return state; }
}

class Main {
    public static void main(String[] args) {
        Order order = new Order();
        order.pay();
        order.ship();
        System.out.println(order.state()); // Shipped[]
        try {
            order.cancel();
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage()); // cannot cancel when Shipped
        }
        order.deliver();
        System.out.println(order.state()); // Delivered[]
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
// Sealed hierarchy + exhaustive `when`: the transition table lives in one place.
sealed interface OrderState {
    data object Pending : OrderState
    data object Paid : OrderState
    data object Shipped : OrderState
    data object Delivered : OrderState
    data object Cancelled : OrderState
}

enum class Action { PAY, SHIP, DELIVER, CANCEL }

fun OrderState.on(action: Action): OrderState = when (this) {
    OrderState.Pending -> when (action) {
        Action.PAY -> OrderState.Paid
        Action.CANCEL -> OrderState.Cancelled
        else -> invalid(action)
    }
    OrderState.Paid -> when (action) {
        Action.SHIP -> OrderState.Shipped
        Action.CANCEL -> OrderState.Cancelled.also { println("  refunding payment") }
        else -> invalid(action)
    }
    OrderState.Shipped -> if (action == Action.DELIVER) OrderState.Delivered else invalid(action)
    OrderState.Delivered, OrderState.Cancelled -> invalid(action)
}

private fun OrderState.invalid(action: Action): Nothing = error("cannot $action when $this")

class Order {
    var state: OrderState = OrderState.Pending
        private set
    fun handle(action: Action) { state = state.on(action) }
}

fun main() {
    val order = Order()
    order.handle(Action.PAY)
    order.handle(Action.SHIP)
    println(order.state)
    runCatching { order.handle(Action.CANCEL) }.onFailure { println(it.message) }
    order.handle(Action.DELIVER)
    println(order.state)
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import "fmt"

type State interface {
	Pay() (State, error)
	Ship() (State, error)
	Deliver() (State, error)
	Cancel() (State, error)
	Name() string
}

// base provides "invalid transition" defaults via embedding.
type base struct{ name string }

func (b base) Name() string             { return b.name }
func (b base) invalid(a string) error   { return fmt.Errorf("cannot %s when %s", a, b.name) }
func (b base) Pay() (State, error)      { return nil, b.invalid("pay") }
func (b base) Ship() (State, error)     { return nil, b.invalid("ship") }
func (b base) Deliver() (State, error)  { return nil, b.invalid("deliver") }
func (b base) Cancel() (State, error)   { return nil, b.invalid("cancel") }

type Pending struct{ base }
type Paid struct{ base }
type Shipped struct{ base }
type Delivered struct{ base }
type Cancelled struct{ base }

func (Pending) Pay() (State, error)    { return Paid{base{"Paid"}}, nil }
func (Pending) Cancel() (State, error) { return Cancelled{base{"Cancelled"}}, nil }
func (Paid) Ship() (State, error)      { return Shipped{base{"Shipped"}}, nil }
func (Paid) Cancel() (State, error) {
	fmt.Println("  refunding payment")
	return Cancelled{base{"Cancelled"}}, nil
}
func (Shipped) Deliver() (State, error) { return Delivered{base{"Delivered"}}, nil }

type Order struct{ state State }

func (o *Order) apply(transition func(State) (State, error)) error {
	next, err := transition(o.state)
	if err != nil {
		return err
	}
	o.state = next
	return nil
}

func main() {
	o := &Order{state: Pending{base{"Pending"}}}
	_ = o.apply(State.Pay)
	_ = o.apply(State.Ship)
	fmt.Println(o.state.Name())
	fmt.Println(o.apply(State.Cancel)) // cannot cancel when Shipped
	_ = o.apply(State.Deliver)
	fmt.Println(o.state.Name())
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

struct OrderState {
    virtual ~OrderState() = default;
    virtual std::string name() const = 0;
    virtual std::unique_ptr<OrderState> pay() const { throw invalid("pay"); }
    virtual std::unique_ptr<OrderState> ship() const { throw invalid("ship"); }
    virtual std::unique_ptr<OrderState> deliver() const { throw invalid("deliver"); }
    virtual std::unique_ptr<OrderState> cancel() const { throw invalid("cancel"); }
protected:
    std::logic_error invalid(const std::string& a) const { return std::logic_error("cannot " + a + " when " + name()); }
};

struct Delivered : OrderState { std::string name() const override { return "Delivered"; } };
struct Cancelled : OrderState { std::string name() const override { return "Cancelled"; } };

struct Shipped : OrderState {
    std::string name() const override { return "Shipped"; }
    std::unique_ptr<OrderState> deliver() const override { return std::make_unique<Delivered>(); }
};

struct Paid : OrderState {
    std::string name() const override { return "Paid"; }
    std::unique_ptr<OrderState> ship() const override { return std::make_unique<Shipped>(); }
    std::unique_ptr<OrderState> cancel() const override {
        std::cout << "  refunding payment\n";
        return std::make_unique<Cancelled>();
    }
};

struct Pending : OrderState {
    std::string name() const override { return "Pending"; }
    std::unique_ptr<OrderState> pay() const override { return std::make_unique<Paid>(); }
    std::unique_ptr<OrderState> cancel() const override { return std::make_unique<Cancelled>(); }
};

class Order {
public:
    void pay() { state_ = state_->pay(); }
    void ship() { state_ = state_->ship(); }
    void deliver() { state_ = state_->deliver(); }
    void cancel() { state_ = state_->cancel(); }
    std::string state() const { return state_->name(); }
private:
    std::unique_ptr<OrderState> state_ = std::make_unique<Pending>();
};

int main() {
    Order order;
    order.pay();
    order.ship();
    std::cout << order.state() << "\n";
    try { order.cancel(); } catch (const std::logic_error& e) { std::cout << e.what() << "\n"; }
    order.deliver();
    std::cout << order.state() << "\n";
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
// Typestate: each state is a type; invalid transitions are compile errors, not runtime errors.
struct Pending;
struct Paid;
struct Shipped;
struct Delivered;
struct Cancelled;

struct Order<S> {
    id: String,
    _state: S,
}

impl Order<Pending> {
    fn new(id: &str) -> Self { Order { id: id.into(), _state: Pending } }
    fn pay(self) -> Order<Paid> { Order { id: self.id, _state: Paid } }
    fn cancel(self) -> Order<Cancelled> { Order { id: self.id, _state: Cancelled } }
}

impl Order<Paid> {
    fn ship(self) -> Order<Shipped> { Order { id: self.id, _state: Shipped } }
    fn cancel(self) -> Order<Cancelled> {
        println!("  refunding payment");
        Order { id: self.id, _state: Cancelled }
    }
}

impl Order<Shipped> {
    fn deliver(self) -> Order<Delivered> { Order { id: self.id, _state: Delivered } }
}

fn main() {
    let order = Order::new("order-1").pay().ship();
    // order.cancel(); // compile error: no method `cancel` on Order<Shipped>
    let delivered: Order<Delivered> = order.deliver();
    println!("{} delivered", delivered.id);

    let cancelled: Order<Cancelled> = Order::new("order-2").pay().cancel();
    println!("{} cancelled", cancelled.id);

    let early: Order<Cancelled> = Order::new("order-3").cancel(); // cancel before paying: no refund
    println!("{} cancelled", early.id);
    // When the state is only known at runtime (e.g. loaded from a DB), use an enum + match instead.
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
from __future__ import annotations

from enum import Enum, auto


class State(Enum):
    PENDING = auto()
    PAID = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()


# Transition table: (state, action) -> next state. Easy to read, test, and draw.
TRANSITIONS: dict[tuple[State, str], State] = {
    (State.PENDING, "pay"): State.PAID,
    (State.PENDING, "cancel"): State.CANCELLED,
    (State.PAID, "ship"): State.SHIPPED,
    (State.PAID, "cancel"): State.CANCELLED,
    (State.SHIPPED, "deliver"): State.DELIVERED,
}


class Order:
    def __init__(self) -> None:
        self.state = State.PENDING

    def handle(self, action: str) -> None:
        try:
            next_state = TRANSITIONS[(self.state, action)]
        except KeyError:
            raise ValueError(f"cannot {action} when {self.state.name}") from None
        if self.state is State.PAID and next_state is State.CANCELLED:
            print("  refunding payment")  # transition side effect
        self.state = next_state


if __name__ == "__main__":
    order = Order()
    order.handle("pay")
    order.handle("ship")
    print(order.state)
    try:
        order.handle("cancel")
    except ValueError as e:
        print(e)
    order.handle("deliver")
    print(order.state)
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Sealed interfaces/records or enums with abstract methods; Spring State Machine for complex workflows. |
| Kotlin | Sealed interfaces + `data object` + exhaustive `when`. |
| Go | Interfaces with embedded defaults, or a `map[state]map[event]state` transition table. |
| C++ | Polymorphic state objects, or `std::variant` + `std::visit`; Boost.SML/MSM for state machines. |
| Rust | Typestate for compile-time guarantees; `enum` + `match` when the state is dynamic. |
| Python | `Enum` + transition table; `transitions` library for larger machines. |

### Referências / References
- *Design Patterns* (GoF), State
- [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/)
- [Refactoring.Guru: State](https://refactoring.guru/design-patterns/state)
