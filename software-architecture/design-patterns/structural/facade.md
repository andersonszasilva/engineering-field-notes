# Facade

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, facade, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Fornecer uma **interface unificada e simplificada** para um conjunto de interfaces de um subsistema.

### Problema
Finalizar um pedido envolve reservar estoque, cobrar o pagamento, agendar o envio e notificar o cliente, cada passo com seu serviço, sua API e sua ordem correta. Se cada controller/handler orquestra isso sozinho, a lógica se duplica e todos ficam acoplados aos detalhes de cada subsistema.

### Estrutura
- **Facade**: expõe operações de alto nível (`placeOrder()`) e coordena os subsistemas.
- **Subsystems**: classes existentes que fazem o trabalho real; não conhecem a fachada.
- Clientes podem continuar acessando os subsistemas diretamente quando precisarem de controle fino.

Em arquitetura, aparece como *application service* / *use case*, *API gateway* e *Backend for Frontend* (BFF).

### Trade-offs
| Prós | Contras |
|------|---------|
| Simplifica o uso de subsistemas complexos. | Pode virar um **god object** que conhece e faz tudo. |
| Reduz o acoplamento do cliente aos detalhes internos. | Esconde capacidades: casos avançados precisam furar a fachada. |
| Ponto único para orquestração, transações e tratamento de erros. | Mudanças em qualquer subsistema podem exigir mudar a fachada. |
| Facilita testes e substituição de subsistemas. | Uma camada extra que, se só repassa chamadas, não agrega valor. |

### Quando usar / evitar
- **Use** para oferecer um ponto de entrada simples a uma biblioteca/subsistema complexo, para definir camadas (cada camada expõe uma fachada) ou para isolar código legado.
- **Evite** que a fachada acumule regras de negócio de domínios diferentes; prefira várias fachadas focadas.

---

## English

### Intent
Provide a **unified, simplified interface** to a set of interfaces in a subsystem.

### Problem
Placing an order involves reserving stock, charging payment, scheduling shipping, and notifying the customer, each step with its own service, API, and correct ordering. If every controller/handler orchestrates this on its own, the logic gets duplicated and everyone is coupled to each subsystem's details.

### Structure
- **Facade**: exposes high-level operations (`placeOrder()`) and coordinates the subsystems.
- **Subsystems**: existing classes doing the real work; they don't know about the facade.
- Clients can still access subsystems directly when they need fine-grained control.

In architecture, it shows up as an *application service* / *use case*, an *API gateway*, and a *Backend for Frontend* (BFF).

### Trade-offs
| Pros | Cons |
|------|------|
| Simplifies using complex subsystems. | Can become a **god object** that knows and does everything. |
| Reduces client coupling to internal details. | Hides capabilities: advanced cases must bypass the facade. |
| Single place for orchestration, transactions, and error handling. | Changes in any subsystem may require changing the facade. |
| Eases testing and swapping subsystems. | An extra layer that adds no value if it only forwards calls. |

### When to use / avoid
- **Use** to offer a simple entry point to a complex library/subsystem, to define layers (each layer exposes a facade), or to isolate legacy code.
- **Avoid** letting the facade accumulate business rules from different domains; prefer several focused facades.

---

## Exemplos / Examples

Scenario: a `CheckoutFacade` orchestrates inventory, payment, shipping, and notification, with compensation if payment fails.

<details>
<summary><strong>Java</strong></summary>

```java
class InventoryService {
    boolean reserve(String sku, int qty) { System.out.println("reserved " + qty + "x " + sku); return true; }
    void release(String sku, int qty) { System.out.println("released " + qty + "x " + sku); }
}
class PaymentService {
    boolean charge(String customer, long cents) { System.out.println("charged " + cents + " to " + customer); return cents < 100_000; }
}
class ShippingService {
    String schedule(String customer, String sku) { System.out.println("shipping " + sku); return "TRACK-123"; }
}
class NotificationService {
    void send(String customer, String msg) { System.out.println("notify " + customer + ": " + msg); }
}

// The facade: one call, correct ordering, compensation on failure.
class CheckoutFacade {
    private final InventoryService inventory = new InventoryService();
    private final PaymentService payment = new PaymentService();
    private final ShippingService shipping = new ShippingService();
    private final NotificationService notification = new NotificationService();

    String placeOrder(String customer, String sku, int qty, long cents) {
        if (!inventory.reserve(sku, qty)) throw new IllegalStateException("out of stock");
        if (!payment.charge(customer, cents)) {
            inventory.release(sku, qty);
            throw new IllegalStateException("payment declined");
        }
        String tracking = shipping.schedule(customer, sku);
        notification.send(customer, "Order shipped: " + tracking);
        return tracking;
    }
}

class Main {
    public static void main(String[] args) {
        System.out.println(new CheckoutFacade().placeOrder("ana", "SKU-1", 2, 5_990));
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
class InventoryService {
    fun reserve(sku: String, qty: Int) = true.also { println("reserved ${qty}x $sku") }
    fun release(sku: String, qty: Int) = println("released ${qty}x $sku")
}
class PaymentService {
    fun charge(customer: String, cents: Long) = (cents < 100_000).also { println("charged $cents to $customer") }
}
class ShippingService {
    fun schedule(customer: String, sku: String) = "TRACK-123".also { println("shipping $sku") }
}
class NotificationService {
    fun send(customer: String, msg: String) = println("notify $customer: $msg")
}

// Dependencies injected through the constructor, with defaults for convenience.
class CheckoutFacade(
    private val inventory: InventoryService = InventoryService(),
    private val payment: PaymentService = PaymentService(),
    private val shipping: ShippingService = ShippingService(),
    private val notification: NotificationService = NotificationService(),
) {
    fun placeOrder(customer: String, sku: String, qty: Int, cents: Long): String {
        check(inventory.reserve(sku, qty)) { "out of stock" }
        if (!payment.charge(customer, cents)) {
            inventory.release(sku, qty)
            error("payment declined")
        }
        return shipping.schedule(customer, sku).also { notification.send(customer, "Order shipped: $it") }
    }
}

fun main() {
    println(CheckoutFacade().placeOrder("ana", "SKU-1", 2, 5_990))
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import (
	"errors"
	"fmt"
)

type Inventory struct{}

func (Inventory) Reserve(sku string, qty int) error { fmt.Printf("reserved %dx %s\n", qty, sku); return nil }
func (Inventory) Release(sku string, qty int)       { fmt.Printf("released %dx %s\n", qty, sku) }

type Payments struct{}

func (Payments) Charge(customer string, cents int64) error {
	fmt.Println("charged", cents, "to", customer)
	if cents >= 100_000 {
		return errors.New("payment declined")
	}
	return nil
}

type Shipping struct{}

func (Shipping) Schedule(customer, sku string) string { fmt.Println("shipping", sku); return "TRACK-123" }

type Notifier struct{}

func (Notifier) Send(customer, msg string) { fmt.Printf("notify %s: %s\n", customer, msg) }

// CheckoutFacade: in Go this is usually a "service" struct in its own package.
type CheckoutFacade struct {
	inventory Inventory
	payments  Payments
	shipping  Shipping
	notifier  Notifier
}

func (c CheckoutFacade) PlaceOrder(customer, sku string, qty int, cents int64) (string, error) {
	if err := c.inventory.Reserve(sku, qty); err != nil {
		return "", fmt.Errorf("reserve: %w", err)
	}
	if err := c.payments.Charge(customer, cents); err != nil {
		c.inventory.Release(sku, qty)
		return "", fmt.Errorf("charge: %w", err)
	}
	tracking := c.shipping.Schedule(customer, sku)
	c.notifier.Send(customer, "Order shipped: "+tracking)
	return tracking, nil
}

func main() {
	tracking, err := CheckoutFacade{}.PlaceOrder("ana", "SKU-1", 2, 5_990)
	fmt.Println(tracking, err)
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class InventoryService {
public:
    bool reserve(const std::string& sku, int qty) { std::cout << "reserved " << qty << "x " << sku << "\n"; return true; }
    void release(const std::string& sku, int qty) { std::cout << "released " << qty << "x " << sku << "\n"; }
};
class PaymentService {
public:
    bool charge(const std::string& c, long cents) { std::cout << "charged " << cents << " to " << c << "\n"; return cents < 100000; }
};
class ShippingService {
public:
    std::string schedule(const std::string&, const std::string& sku) { std::cout << "shipping " << sku << "\n"; return "TRACK-123"; }
};
class NotificationService {
public:
    void send(const std::string& c, const std::string& msg) { std::cout << "notify " << c << ": " << msg << "\n"; }
};

class CheckoutFacade {
public:
    std::string placeOrder(const std::string& customer, const std::string& sku, int qty, long cents) {
        if (!inventory_.reserve(sku, qty)) throw std::runtime_error("out of stock");
        if (!payment_.charge(customer, cents)) {
            inventory_.release(sku, qty);
            throw std::runtime_error("payment declined");
        }
        auto tracking = shipping_.schedule(customer, sku);
        notification_.send(customer, "Order shipped: " + tracking);
        return tracking;
    }
private:
    InventoryService inventory_;
    PaymentService payment_;
    ShippingService shipping_;
    NotificationService notification_;
};

int main() {
    CheckoutFacade checkout;
    std::cout << checkout.placeOrder("ana", "SKU-1", 2, 5990) << "\n";
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
struct Inventory;
impl Inventory {
    fn reserve(&self, sku: &str, qty: u32) -> Result<(), String> { println!("reserved {qty}x {sku}"); Ok(()) }
    fn release(&self, sku: &str, qty: u32) { println!("released {qty}x {sku}"); }
}

struct Payments;
impl Payments {
    fn charge(&self, customer: &str, cents: u64) -> Result<(), String> {
        println!("charged {cents} to {customer}");
        if cents < 100_000 { Ok(()) } else { Err("payment declined".into()) }
    }
}

struct Shipping;
impl Shipping {
    fn schedule(&self, _customer: &str, sku: &str) -> String { println!("shipping {sku}"); "TRACK-123".into() }
}

struct Notifier;
impl Notifier {
    fn send(&self, customer: &str, msg: &str) { println!("notify {customer}: {msg}"); }
}

// In Rust, a facade is often a module's `pub` API hiding private submodules.
pub struct CheckoutFacade {
    inventory: Inventory,
    payments: Payments,
    shipping: Shipping,
    notifier: Notifier,
}

impl CheckoutFacade {
    pub fn new() -> Self {
        Self { inventory: Inventory, payments: Payments, shipping: Shipping, notifier: Notifier }
    }

    pub fn place_order(&self, customer: &str, sku: &str, qty: u32, cents: u64) -> Result<String, String> {
        self.inventory.reserve(sku, qty)?;
        if let Err(e) = self.payments.charge(customer, cents) {
            self.inventory.release(sku, qty);
            return Err(e);
        }
        let tracking = self.shipping.schedule(customer, sku);
        self.notifier.send(customer, &format!("Order shipped: {tracking}"));
        Ok(tracking)
    }
}

fn main() {
    println!("{:?}", CheckoutFacade::new().place_order("ana", "SKU-1", 2, 5_990));
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
class InventoryService:
    def reserve(self, sku: str, qty: int) -> bool:
        print(f"reserved {qty}x {sku}")
        return True

    def release(self, sku: str, qty: int) -> None:
        print(f"released {qty}x {sku}")


class PaymentService:
    def charge(self, customer: str, cents: int) -> bool:
        print(f"charged {cents} to {customer}")
        return cents < 100_000


class ShippingService:
    def schedule(self, customer: str, sku: str) -> str:
        print(f"shipping {sku}")
        return "TRACK-123"


class NotificationService:
    def send(self, customer: str, msg: str) -> None:
        print(f"notify {customer}: {msg}")


class CheckoutFacade:
    def __init__(self) -> None:
        self._inventory = InventoryService()
        self._payment = PaymentService()
        self._shipping = ShippingService()
        self._notification = NotificationService()

    def place_order(self, customer: str, sku: str, qty: int, cents: int) -> str:
        if not self._inventory.reserve(sku, qty):
            raise RuntimeError("out of stock")
        if not self._payment.charge(customer, cents):
            self._inventory.release(sku, qty)
            raise RuntimeError("payment declined")
        tracking = self._shipping.schedule(customer, sku)
        self._notification.send(customer, f"Order shipped: {tracking}")
        return tracking


if __name__ == "__main__":
    print(CheckoutFacade().place_order("ana", "SKU-1", 2, 5_990))
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Application services in layered/hexagonal architectures; SLF4J is a facade over logging backends. |
| Kotlin | Constructor defaults allow both convenience and injection for tests. |
| Go | A package with a small exported API over unexported internals *is* a facade. |
| C++ | Facades often hide heavy headers (combine with Pimpl to reduce compile times). |
| Rust | `pub` API of a module/crate over private submodules; `pub use` re-exports shape the facade. |
| Python | A package `__init__.py` exposing a few functions (`requests.get`) is a facade over many modules. |

### Referências / References
- *Design Patterns* (GoF), Facade
- Sam Newman, [Backends For Frontends](https://samnewman.io/patterns/architectural/bff/)
- [Refactoring.Guru: Facade](https://refactoring.guru/design-patterns/facade)
