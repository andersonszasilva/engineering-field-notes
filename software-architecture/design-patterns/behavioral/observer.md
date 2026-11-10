# Observer

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, observer, pub-sub, events, reactive, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Definir uma dependência **um-para-muitos** entre objetos, de modo que quando um objeto muda de estado, **todos os dependentes são notificados** automaticamente.

### Problema
Quando um pedido é pago, é preciso enviar e-mail, atualizar o estoque, registrar métricas e notificar o time de logística. Se `Order` chama cada um desses serviços diretamente, fica acoplado a todos eles, e cada nova reação exige alterar `Order`.

### Estrutura
- **Subject** (publisher): mantém a lista de observers e oferece `subscribe`/`unsubscribe`; notifica todos quando algo acontece.
- **Observer** (subscriber): interface com `update(event)`.
- **ConcreteObservers**: reagem ao evento.
- Modelos de entrega: **push** (o evento leva os dados) ou **pull** (o observer consulta o subject).

O mesmo princípio escala para arquitetura: *event listeners*, programação reativa (Rx, Flow), *pub/sub* com brokers (Kafka, SNS, Pub/Sub) e *event-driven architecture*.

### Trade-offs
| Prós | Contras |
|------|---------|
| Subject desacoplado dos observers: só conhece a interface. | **Ordem de notificação** não garantida/implícita. |
| Open/Closed: novas reações sem mudar o subject. | Fluxo difícil de rastrear: "quem reage a este evento?". |
| Suporta broadcast e relações dinâmicas em runtime. | **Memory leaks** com listeners que nunca fazem unsubscribe (*lapsed listener*). |
| Base de UIs reativas e arquiteturas orientadas a eventos. | Um observer lento ou com exceção pode travar/quebrar os outros (síncrono). |
| | Cascatas de atualizações e loops (A notifica B que notifica A). |

### Quando usar / evitar
- **Use** quando mudanças em um objeto precisam disparar reações em outros que variam ou não são conhecidos de antemão: eventos de domínio, UI, cache invalidation, webhooks.
- **Evite** quando a sequência de passos precisa ser garantida e transacional; uma orquestração explícita é mais clara. Em sistemas distribuídos, prefira um broker com entrega confiável.

---

## English

### Intent
Define a **one-to-many** dependency between objects so that when one object changes state, **all its dependents are notified** automatically.

### Problem
When an order is paid, we must send an email, update stock, record metrics, and notify logistics. If `Order` calls each of these services directly, it's coupled to all of them, and every new reaction requires changing `Order`.

### Structure
- **Subject** (publisher): keeps the list of observers and offers `subscribe`/`unsubscribe`; notifies all of them when something happens.
- **Observer** (subscriber): an interface with `update(event)`.
- **ConcreteObservers**: react to the event.
- Delivery models: **push** (the event carries the data) or **pull** (the observer queries the subject).

The same principle scales to architecture: *event listeners*, reactive programming (Rx, Flow), *pub/sub* with brokers (Kafka, SNS, Pub/Sub), and *event-driven architecture*.

### Trade-offs
| Pros | Cons |
|------|------|
| The subject is decoupled from observers: it only knows the interface. | **Notification order** is not guaranteed/implicit. |
| Open/Closed: new reactions without changing the subject. | Hard-to-trace flow: "who reacts to this event?". |
| Supports broadcast and dynamic relationships at runtime. | **Memory leaks** from listeners that never unsubscribe (*lapsed listener*). |
| The basis of reactive UIs and event-driven architectures. | A slow or failing observer can block/break the others (synchronous). |
| | Update cascades and loops (A notifies B, which notifies A). |

### When to use / avoid
- **Use** when changes in one object must trigger reactions in others that vary or aren't known in advance: domain events, UI, cache invalidation, webhooks.
- **Avoid** when the sequence of steps must be guaranteed and transactional; explicit orchestration is clearer. In distributed systems, prefer a broker with reliable delivery.

---

## Exemplos / Examples

Scenario: an in-process event bus publishes `OrderPaid`; email and inventory subscribers react. Subscriptions can be cancelled.

### Java
```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

record OrderPaid(String orderId, long cents) {}

class EventPublisher<E> {
    // CopyOnWriteArrayList: safe if listeners subscribe/unsubscribe during notification.
    private final List<Consumer<E>> listeners = new CopyOnWriteArrayList<>();

    Runnable subscribe(Consumer<E> listener) {
        listeners.add(listener);
        return () -> listeners.remove(listener); // returns an "unsubscribe" handle
    }

    void publish(E event) {
        for (Consumer<E> l : listeners) {
            try {
                l.accept(event);
            } catch (RuntimeException e) {
                System.err.println("listener failed: " + e.getMessage()); // isolate failures
            }
        }
    }
}

class Main {
    public static void main(String[] args) {
        var orderEvents = new EventPublisher<OrderPaid>();
        orderEvents.subscribe(e -> System.out.println("email: receipt for " + e.orderId()));
        Runnable cancelInventory = orderEvents.subscribe(e -> System.out.println("inventory: ship " + e.orderId()));

        orderEvents.publish(new OrderPaid("order-1", 5_990));
        cancelInventory.run();
        orderEvents.publish(new OrderPaid("order-2", 1_000)); // only email reacts
    }
}
```

### Kotlin
```kotlin
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.yield

data class OrderPaid(val orderId: String, val cents: Long)

// Simple callback-based observer.
class EventPublisher<E> {
    private val listeners = mutableListOf<(E) -> Unit>()
    fun subscribe(listener: (E) -> Unit): () -> Unit {
        listeners += listener
        return { listeners -= listener }
    }
    fun publish(event: E) = listeners.toList().forEach { it(event) }
}

fun main() = runBlocking {
    val events = EventPublisher<OrderPaid>()
    val unsubscribe = events.subscribe { println("email: receipt for ${it.orderId}") }
    events.publish(OrderPaid("order-1", 5_990))
    unsubscribe()

    // Idiomatic reactive version with coroutines (requires kotlinx-coroutines-core).
    val flow = MutableSharedFlow<OrderPaid>()
    val job = launch { flow.collect { println("inventory: ship ${it.orderId}") } }
    yield() // let the collector subscribe
    flow.emit(OrderPaid("order-2", 1_000))
    job.cancel()
}
```

### Go
```go
package main

import (
	"fmt"
	"sync"
)

type OrderPaid struct {
	OrderID string
	Cents   int64
}

type Bus[E any] struct {
	mu        sync.RWMutex
	nextID    int
	listeners map[int]func(E)
}

func NewBus[E any]() *Bus[E] { return &Bus[E]{listeners: map[int]func(E){}} }

// Subscribe returns an unsubscribe function.
func (b *Bus[E]) Subscribe(fn func(E)) func() {
	b.mu.Lock()
	defer b.mu.Unlock()
	id := b.nextID
	b.nextID++
	b.listeners[id] = fn
	return func() {
		b.mu.Lock()
		defer b.mu.Unlock()
		delete(b.listeners, id)
	}
}

func (b *Bus[E]) Publish(e E) {
	b.mu.RLock()
	fns := make([]func(E), 0, len(b.listeners))
	for _, fn := range b.listeners {
		fns = append(fns, fn)
	}
	b.mu.RUnlock()
	for _, fn := range fns { // call outside the lock to avoid deadlocks
		fn(e)
	}
}

func main() {
	bus := NewBus[OrderPaid]()
	bus.Subscribe(func(e OrderPaid) { fmt.Println("email: receipt for", e.OrderID) })
	cancel := bus.Subscribe(func(e OrderPaid) { fmt.Println("inventory: ship", e.OrderID) })

	bus.Publish(OrderPaid{"order-1", 5990})
	cancel()
	bus.Publish(OrderPaid{"order-2", 1000})
}
```

### C++
```cpp
#include <functional>
#include <iostream>
#include <map>
#include <string>

struct OrderPaid { std::string orderId; long cents; };

template <class E>
class EventPublisher {
public:
    using Listener = std::function<void(const E&)>;
    using Token = int;

    Token subscribe(Listener l) { listeners_[next_] = std::move(l); return next_++; }
    void unsubscribe(Token t) { listeners_.erase(t); }

    void publish(const E& e) const {
        for (const auto& [id, l] : listeners_) l(e);
    }
private:
    std::map<Token, Listener> listeners_;
    Token next_ = 0;
};

int main() {
    EventPublisher<OrderPaid> events;
    events.subscribe([](const OrderPaid& e) { std::cout << "email: receipt for " << e.orderId << "\n"; });
    auto inv = events.subscribe([](const OrderPaid& e) { std::cout << "inventory: ship " << e.orderId << "\n"; });

    events.publish({"order-1", 5990});
    events.unsubscribe(inv);
    events.publish({"order-2", 1000});
    // Beware: lambdas capturing `this` or references can dangle if the observer dies first.
}
```

### Rust
```rust
use std::sync::mpsc;
use std::thread;

#[derive(Clone, Debug)]
struct OrderPaid {
    order_id: String,
    cents: u64,
}

// Callback-based subject. Box<dyn Fn> lets listeners capture their own state.
struct EventPublisher<E> {
    listeners: Vec<Box<dyn Fn(&E)>>,
}

impl<E> EventPublisher<E> {
    fn new() -> Self { Self { listeners: Vec::new() } }
    fn subscribe(&mut self, f: impl Fn(&E) + 'static) { self.listeners.push(Box::new(f)); }
    fn publish(&self, event: &E) {
        for l in &self.listeners {
            l(event);
        }
    }
}

fn main() {
    let mut events = EventPublisher::new();
    events.subscribe(|e: &OrderPaid| println!("email: receipt for {}", e.order_id));
    events.subscribe(|e: &OrderPaid| println!("metrics: revenue += {}", e.cents));
    events.publish(&OrderPaid { order_id: "order-1".into(), cents: 5_990 });

    // Across threads, channels decouple publisher and subscriber.
    let (tx, rx) = mpsc::channel::<OrderPaid>();
    let consumer = thread::spawn(move || {
        for e in rx {
            println!("inventory (thread): ship {}", e.order_id);
        }
    });
    tx.send(OrderPaid { order_id: "order-2".into(), cents: 1_000 }).unwrap();
    drop(tx); // closing the channel ends the consumer loop
    consumer.join().unwrap();
}
```

### Python
```python
from collections.abc import Callable
from dataclasses import dataclass
from typing import Generic, TypeVar

E = TypeVar("E")


@dataclass(frozen=True)
class OrderPaid:
    order_id: str
    cents: int


class EventPublisher(Generic[E]):
    def __init__(self) -> None:
        self._listeners: list[Callable[[E], None]] = []

    def subscribe(self, listener: Callable[[E], None]) -> Callable[[], None]:
        self._listeners.append(listener)
        return lambda: self._listeners.remove(listener)

    def publish(self, event: E) -> None:
        for listener in list(self._listeners):  # copy: listeners may unsubscribe while notified
            listener(event)


if __name__ == "__main__":
    events: EventPublisher[OrderPaid] = EventPublisher()
    events.subscribe(lambda e: print(f"email: receipt for {e.order_id}"))
    cancel = events.subscribe(lambda e: print(f"inventory: ship {e.order_id}"))

    events.publish(OrderPaid("order-1", 5_990))
    cancel()
    events.publish(OrderPaid("order-2", 1_000))
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `java.util.Observable` is deprecated; use listeners, Spring `@EventListener`, `java.util.concurrent.Flow`, Reactor/RxJava. |
| Kotlin | `SharedFlow`/`StateFlow` for reactive streams; `Delegates.observable` for property changes. |
| Go | Callbacks with unsubscribe funcs, or channels per subscriber (mind slow consumers and goroutine leaks). |
| C++ | `std::function` callbacks with tokens; Boost.Signals2 handles lifetime/thread safety. |
| Rust | `Box<dyn Fn>` callbacks; channels (`mpsc`, `tokio::sync::broadcast`) avoid shared ownership issues. |
| Python | Callbacks; `weakref.WeakMethod` to avoid lapsed listeners; `blinker` signals; `asyncio` events. |

### Referências / References
- *Design Patterns* (GoF), Observer
- Martin Fowler, [What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html)
- [Refactoring.Guru: Observer](https://refactoring.guru/design-patterns/observer)
