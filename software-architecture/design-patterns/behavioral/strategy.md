# Strategy

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, strategy, policy, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Definir uma **família de algoritmos**, encapsular cada um e torná-los **intercambiáveis**. O Strategy permite que o algoritmo varie independentemente dos clientes que o usam.

### Problema
O cálculo de frete depende da modalidade (padrão, expresso, retirada na loja) e novas regras surgem com frequência (frete grátis em campanhas). Um `switch` no meio do serviço de checkout cresce sem parar, mistura regras e obriga a alterar e retestar o checkout a cada nova modalidade.

### Estrutura
- **Strategy**: interface comum do algoritmo (`ShippingPolicy.cost(order)`).
- **ConcreteStrategies**: implementações (`Standard`, `Express`, `FreeAbove`).
- **Context**: usa uma strategy recebida por construtor/parâmetro, sem saber qual é.

Em linguagens com funções de primeira classe, uma strategy é frequentemente **só uma função** (`Comparator`, `key=` no `sorted`).

Strategy vs State: estrutura parecida. No Strategy **o cliente escolhe** o algoritmo e as strategies não se conhecem; no State **os próprios estados** decidem a transição.

### Trade-offs
| Prós | Contras |
|------|---------|
| Elimina condicionais; cada algoritmo isolado e testável. | O cliente precisa conhecer as estratégias para escolher a certa. |
| Troca de algoritmo em runtime (config, A/B test, feature flag). | Mais tipos/objetos para algoritmos que raramente mudam. |
| Open/Closed: novos algoritmos sem alterar o contexto. | A interface comum pode forçar parâmetros que só algumas strategies usam. |
| Favorece composição sobre herança. | Indireção: ler o código exige descobrir qual strategy está injetada. |

### Quando usar / evitar
- **Use** quando há várias formas de fazer a mesma coisa e a escolha depende de contexto: preços, frete, ordenação, compressão, retry/backoff, roteamento, load balancing.
- **Evite** quando só existem duas variações estáveis; um `if` claro é mais simples.

---

## English

### Intent
Define a **family of algorithms**, encapsulate each one, and make them **interchangeable**. Strategy lets the algorithm vary independently from the clients that use it.

### Problem
Shipping cost depends on the method (standard, express, store pickup), and new rules appear often (free shipping during campaigns). A `switch` in the middle of the checkout service keeps growing, mixes rules, and forces you to change and retest checkout for each new method.

### Structure
- **Strategy**: the common algorithm interface (`ShippingPolicy.cost(order)`).
- **ConcreteStrategies**: implementations (`Standard`, `Express`, `FreeAbove`).
- **Context**: uses a strategy received via constructor/parameter, without knowing which one.

In languages with first-class functions, a strategy is often **just a function** (`Comparator`, `key=` in `sorted`).

Strategy vs State: similar structure. In Strategy **the client chooses** the algorithm and strategies don't know each other; in State **the states themselves** decide the transition.

### Trade-offs
| Pros | Cons |
|------|------|
| Removes conditionals; each algorithm is isolated and testable. | The client must know the strategies to pick the right one. |
| Swap algorithms at runtime (config, A/B test, feature flag). | More types/objects for algorithms that rarely change. |
| Open/Closed: new algorithms without changing the context. | The common interface may force parameters only some strategies use. |
| Favors composition over inheritance. | Indirection: reading the code requires finding which strategy is injected. |

### When to use / avoid
- **Use** when there are several ways to do the same thing and the choice depends on context: pricing, shipping, sorting, compression, retry/backoff, routing, load balancing.
- **Avoid** when there are only two stable variations; a clear `if` is simpler.

---

## Exemplos / Examples

Scenario: checkout computes shipping with a pluggable policy (amounts in cents).

<details>
<summary><strong>Java</strong></summary>

```java
import java.util.Map;

record Order(long subtotalCents, double weightKg) {}

@FunctionalInterface
interface ShippingPolicy {
    long cost(Order order);
}

class StandardShipping implements ShippingPolicy {
    public long cost(Order o) { return 1_000 + Math.round(o.weightKg() * 200); }
}

class ExpressShipping implements ShippingPolicy {
    public long cost(Order o) { return 2_500 + Math.round(o.weightKg() * 400); }
}

// Strategies can decorate other strategies.
record FreeAbove(long thresholdCents, ShippingPolicy fallback) implements ShippingPolicy {
    public long cost(Order o) { return o.subtotalCents() >= thresholdCents ? 0 : fallback.cost(o); }
}

// Context
class Checkout {
    private final ShippingPolicy shipping;
    Checkout(ShippingPolicy shipping) { this.shipping = shipping; }
    long total(Order o) { return o.subtotalCents() + shipping.cost(o); }
}

class Main {
    public static void main(String[] args) {
        Map<String, ShippingPolicy> policies = Map.of(
                "standard", new StandardShipping(),
                "express", new ExpressShipping(),
                "campaign", new FreeAbove(20_000, new StandardShipping()),
                "pickup", o -> 0L); // a lambda is a strategy too

        Order order = new Order(15_000, 2.5);
        policies.forEach((name, policy) -> System.out.println(name + " -> " + new Checkout(policy).total(order)));
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
import kotlin.math.roundToLong

data class Order(val subtotalCents: Long, val weightKg: Double)

// A strategy as a function type.
typealias ShippingPolicy = (Order) -> Long

val standard: ShippingPolicy = { 1_000 + (it.weightKg * 200).roundToLong() }
val express: ShippingPolicy = { 2_500 + (it.weightKg * 400).roundToLong() }
fun freeAbove(threshold: Long, fallback: ShippingPolicy): ShippingPolicy =
    { if (it.subtotalCents >= threshold) 0 else fallback(it) }

class Checkout(private val shipping: ShippingPolicy) {
    fun total(order: Order) = order.subtotalCents + shipping(order)
}

fun main() {
    val order = Order(15_000, 2.5)
    val policies = mapOf(
        "standard" to standard,
        "express" to express,
        "campaign" to freeAbove(20_000, standard),
        "pickup" to { _: Order -> 0L },
    )
    policies.forEach { (name, policy) -> println("$name -> ${Checkout(policy).total(order)}") }
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import (
	"fmt"
	"math"
)

type Order struct {
	SubtotalCents int64
	WeightKg      float64
}

// Strategy as a function type: the most common Go form.
type ShippingPolicy func(Order) int64

func Standard(o Order) int64 { return 1000 + int64(math.Round(o.WeightKg*200)) }
func Express(o Order) int64  { return 2500 + int64(math.Round(o.WeightKg*400)) }

func FreeAbove(threshold int64, fallback ShippingPolicy) ShippingPolicy {
	return func(o Order) int64 {
		if o.SubtotalCents >= threshold {
			return 0
		}
		return fallback(o)
	}
}

type Checkout struct{ Shipping ShippingPolicy }

func (c Checkout) Total(o Order) int64 { return o.SubtotalCents + c.Shipping(o) }

func main() {
	order := Order{SubtotalCents: 15000, WeightKg: 2.5}
	policies := map[string]ShippingPolicy{
		"standard": Standard,
		"express":  Express,
		"campaign": FreeAbove(20000, Standard),
		"pickup":   func(Order) int64 { return 0 },
	}
	for name, p := range policies {
		fmt.Println(name, "->", Checkout{Shipping: p}.Total(order))
	}
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <cmath>
#include <functional>
#include <iostream>
#include <map>
#include <string>

struct Order { long subtotalCents; double weightKg; };

// Runtime strategy via std::function...
using ShippingPolicy = std::function<long(const Order&)>;

long standard(const Order& o) { return 1000 + std::lround(o.weightKg * 200); }
long express(const Order& o) { return 2500 + std::lround(o.weightKg * 400); }

ShippingPolicy freeAbove(long threshold, ShippingPolicy fallback) {
    return [=](const Order& o) { return o.subtotalCents >= threshold ? 0L : fallback(o); };
}

class Checkout {
public:
    explicit Checkout(ShippingPolicy p) : shipping_(std::move(p)) {}
    long total(const Order& o) const { return o.subtotalCents + shipping_(o); }
private:
    ShippingPolicy shipping_;
};

// ...or compile-time strategy via a template parameter (policy-based design, zero overhead).
template <class Policy>
long totalWith(const Order& o, Policy policy) { return o.subtotalCents + policy(o); }

int main() {
    Order order{15000, 2.5};
    std::map<std::string, ShippingPolicy> policies{
        {"standard", standard},
        {"express", express},
        {"campaign", freeAbove(20000, standard)},
        {"pickup", [](const Order&) { return 0L; }},
    };
    for (const auto& [name, p] : policies) std::cout << name << " -> " << Checkout(p).total(order) << "\n";
    std::cout << "static -> " << totalWith(order, express) << "\n";
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
struct Order {
    subtotal_cents: u64,
    weight_kg: f64,
}

trait ShippingPolicy {
    fn cost(&self, order: &Order) -> u64;
}

struct Standard;
impl ShippingPolicy for Standard {
    fn cost(&self, o: &Order) -> u64 { 1_000 + (o.weight_kg * 200.0).round() as u64 }
}

struct Express;
impl ShippingPolicy for Express {
    fn cost(&self, o: &Order) -> u64 { 2_500 + (o.weight_kg * 400.0).round() as u64 }
}

struct FreeAbove<P: ShippingPolicy> { threshold: u64, fallback: P }
impl<P: ShippingPolicy> ShippingPolicy for FreeAbove<P> {
    fn cost(&self, o: &Order) -> u64 {
        if o.subtotal_cents >= self.threshold { 0 } else { self.fallback.cost(o) }
    }
}

// Closures are strategies too.
impl<F: Fn(&Order) -> u64> ShippingPolicy for F {
    fn cost(&self, o: &Order) -> u64 { self(o) }
}

// Static dispatch (generic) context; use Box<dyn ShippingPolicy> to choose at runtime.
struct Checkout<P: ShippingPolicy> { shipping: P }
impl<P: ShippingPolicy> Checkout<P> {
    fn total(&self, o: &Order) -> u64 { o.subtotal_cents + self.shipping.cost(o) }
}

fn main() {
    let order = Order { subtotal_cents: 15_000, weight_kg: 2.5 };
    let policies: Vec<(&str, Box<dyn ShippingPolicy>)> = vec![
        ("standard", Box::new(Standard)),
        ("express", Box::new(Express)),
        ("campaign", Box::new(FreeAbove { threshold: 20_000, fallback: Standard })),
        ("pickup", Box::new(|_: &Order| 0u64)),
    ];
    for (name, policy) in &policies {
        println!("{name} -> {}", order.subtotal_cents + policy.cost(&order));
    }
    println!("static -> {}", Checkout { shipping: Express }.total(&order));
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
from collections.abc import Callable
from dataclasses import dataclass


@dataclass(frozen=True)
class Order:
    subtotal_cents: int
    weight_kg: float


# A strategy is just a callable.
ShippingPolicy = Callable[[Order], int]


def standard(o: Order) -> int:
    return 1_000 + round(o.weight_kg * 200)


def express(o: Order) -> int:
    return 2_500 + round(o.weight_kg * 400)


def free_above(threshold: int, fallback: ShippingPolicy) -> ShippingPolicy:
    return lambda o: 0 if o.subtotal_cents >= threshold else fallback(o)


class Checkout:
    def __init__(self, shipping: ShippingPolicy) -> None:
        self._shipping = shipping

    def total(self, order: Order) -> int:
        return order.subtotal_cents + self._shipping(order)


if __name__ == "__main__":
    order = Order(15_000, 2.5)
    policies: dict[str, ShippingPolicy] = {
        "standard": standard,
        "express": express,
        "campaign": free_above(20_000, standard),
        "pickup": lambda _: 0,
    }
    for name, policy in policies.items():
        print(f"{name} -> {Checkout(policy).total(order)}")
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `Comparator`, `@FunctionalInterface` + lambdas; Spring injects strategies by bean name (`Map<String, Policy>`). |
| Kotlin | Function types and `typealias`; `fun interface` when you need a named type. |
| Go | Function types (`type Policy func(Order) int64`) or one-method interfaces (`sort.Interface`). |
| C++ | `std::function` for runtime, templates/policies (`std::sort` comparator, allocators) for zero-cost compile-time strategies. |
| Rust | Traits with generics (static) or `Box<dyn Trait>` (dynamic); blanket impls for closures. |
| Python | Pass functions; `key=` arguments in `sorted`/`max` are strategies. |

### Referências / References
- *Design Patterns* (GoF), Strategy
- *Modern C++ Design*, Andrei Alexandrescu: policy-based design
- [Refactoring.Guru: Strategy](https://refactoring.guru/design-patterns/strategy)
