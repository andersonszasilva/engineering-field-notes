# Adapter

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, adapter, wrapper, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Converter a interface de uma classe em **outra interface que o cliente espera**, permitindo que classes incompatíveis trabalhem juntas.

### Problema
O domínio define `PaymentGateway.charge(orderId, amount)`, mas o SDK de um fornecedor legado expõe `makePayment(cents, currency, reference)` e devolve um código de status. Não dá (ou não se deve) alterar o SDK, e espalhar chamadas a ele pelo domínio acopla o negócio ao fornecedor.

### Estrutura
- **Target**: interface que o cliente usa (`PaymentGateway`).
- **Adaptee**: classe existente com interface incompatível (`LegacyPayClient`).
- **Adapter**: implementa o Target e traduz as chamadas para o Adaptee (unidades, tipos, erros).
- Variantes: **object adapter** (composição, o mais comum) e **class adapter** (herança múltipla, C++).

É o padrão por trás da camada *anti-corruption* do DDD e dos *adapters* da arquitetura hexagonal (ports & adapters).

### Trade-offs
| Prós | Contras |
|------|---------|
| Integra código legado/terceiros sem alterá-lo. | Uma camada a mais de indireção e de código para manter. |
| Isola o domínio do fornecedor: trocar de fornecedor = novo adapter. | Tradução de erros e semântica pode ser imperfeita (*leaky abstraction*). |
| Ponto natural para conversão de unidades, tipos e erros. | Muitos adapters pequenos podem esconder um design de interfaces ruim. |
| Facilita testes: o Target é fácil de mockar. | Pode haver custo de performance com conversões de dados grandes. |

### Quando usar / evitar
- **Use** ao integrar SDKs externos, sistemas legados, APIs de versões diferentes, ou ao implementar *ports* em arquitetura hexagonal.
- **Evite** quando você controla as duas pontas: ajuste a interface diretamente.

---

## English

### Intent
Convert the interface of a class into **another interface clients expect**, letting incompatible classes work together.

### Problem
The domain defines `PaymentGateway.charge(orderId, amount)`, but a legacy vendor SDK exposes `makePayment(cents, currency, reference)` and returns a status code. You can't (or shouldn't) change the SDK, and scattering calls to it through the domain couples the business to the vendor.

### Structure
- **Target**: the interface the client uses (`PaymentGateway`).
- **Adaptee**: an existing class with an incompatible interface (`LegacyPayClient`).
- **Adapter**: implements Target and translates calls to the Adaptee (units, types, errors).
- Variants: **object adapter** (composition, the most common) and **class adapter** (multiple inheritance, C++).

It's the pattern behind DDD's *anti-corruption layer* and the *adapters* of hexagonal architecture (ports & adapters).

### Trade-offs
| Pros | Cons |
|------|------|
| Integrates legacy/third-party code without changing it. | One more layer of indirection and code to maintain. |
| Isolates the domain from the vendor: switching vendors = new adapter. | Translating errors and semantics may be imperfect (*leaky abstraction*). |
| A natural place for unit, type, and error conversion. | Many small adapters can hide a poor interface design. |
| Eases testing: the Target is easy to mock. | Converting large data structures can cost performance. |

### When to use / avoid
- **Use** when integrating external SDKs, legacy systems, different API versions, or when implementing *ports* in hexagonal architecture.
- **Avoid** when you control both sides: just fix the interface.

---

## Exemplos / Examples

Scenario: adapt a legacy payment SDK (cents + status codes) to the domain's `PaymentGateway` (decimal amount + exceptions/results).

### Java
```java
import java.math.BigDecimal;

// Target
interface PaymentGateway {
    void charge(String orderId, BigDecimal amount);
}

// Adaptee: third-party code we can't change.
class LegacyPayClient {
    int makePayment(long cents, String currency, String reference) {
        System.out.println("legacy: " + cents + " " + currency + " ref=" + reference);
        return cents > 0 ? 0 : 42; // 0 = OK
    }
}

class PaymentFailedException extends RuntimeException {
    PaymentFailedException(String msg) { super(msg); }
}

// Adapter: composition + translation of units and errors.
class LegacyPayAdapter implements PaymentGateway {
    private final LegacyPayClient client;

    LegacyPayAdapter(LegacyPayClient client) { this.client = client; }

    @Override public void charge(String orderId, BigDecimal amount) {
        long cents = amount.movePointRight(2).longValueExact();
        int status = client.makePayment(cents, "BRL", orderId);
        if (status != 0) throw new PaymentFailedException("legacy status " + status);
    }
}

class Main {
    public static void main(String[] args) {
        PaymentGateway gateway = new LegacyPayAdapter(new LegacyPayClient());
        gateway.charge("order-1", new BigDecimal("19.90"));
    }
}
```

### Kotlin
```kotlin
import java.math.BigDecimal

interface PaymentGateway {
    fun charge(orderId: String, amount: BigDecimal): Result<Unit>
}

class LegacyPayClient {
    fun makePayment(cents: Long, currency: String, reference: String): Int {
        println("legacy: $cents $currency ref=$reference")
        return if (cents > 0) 0 else 42
    }
}

class LegacyPayAdapter(private val client: LegacyPayClient) : PaymentGateway {
    override fun charge(orderId: String, amount: BigDecimal): Result<Unit> = runCatching {
        val status = client.makePayment(amount.movePointRight(2).longValueExact(), "BRL", orderId)
        check(status == 0) { "legacy status $status" }
    }
}

// Extension functions are a lightweight alternative when no interface is required.
fun LegacyPayClient.chargeBrl(orderId: String, amount: BigDecimal) =
    makePayment(amount.movePointRight(2).longValueExact(), "BRL", orderId)

fun main() {
    val gateway: PaymentGateway = LegacyPayAdapter(LegacyPayClient())
    println(gateway.charge("order-1", BigDecimal("19.90")))
}
```

### Go
```go
package main

import (
	"fmt"
	"math"
)

// Target: defined by the consumer, as Go recommends.
type PaymentGateway interface {
	Charge(orderID string, amount float64) error
}

// Adaptee
type LegacyPayClient struct{}

func (LegacyPayClient) MakePayment(cents int64, currency, reference string) int {
	fmt.Println("legacy:", cents, currency, "ref="+reference)
	if cents > 0 {
		return 0
	}
	return 42
}

// Adapter
type LegacyPayAdapter struct {
	client LegacyPayClient
}

func (a LegacyPayAdapter) Charge(orderID string, amount float64) error {
	cents := int64(math.Round(amount * 100)) // real code: use integer cents or a decimal type end to end
	if status := a.client.MakePayment(cents, "BRL", orderID); status != 0 {
		return fmt.Errorf("legacy payment failed: status %d", status)
	}
	return nil
}

func main() {
	var gw PaymentGateway = LegacyPayAdapter{}
	if err := gw.Charge("order-1", 19.90); err != nil {
		fmt.Println(err)
	}
}
```

### C++
```cpp
#include <cmath>
#include <cstdint>
#include <iostream>
#include <stdexcept>
#include <string>

// Target
struct PaymentGateway {
    virtual ~PaymentGateway() = default;
    virtual void charge(const std::string& orderId, double amount) = 0;
};

// Adaptee
class LegacyPayClient {
public:
    int makePayment(std::int64_t cents, const std::string& currency, const std::string& ref) {
        std::cout << "legacy: " << cents << " " << currency << " ref=" << ref << "\n";
        return cents > 0 ? 0 : 42;
    }
};

// Object adapter (composition). A class adapter would inherit from both instead.
class LegacyPayAdapter : public PaymentGateway {
public:
    explicit LegacyPayAdapter(LegacyPayClient& client) : client_(client) {}

    void charge(const std::string& orderId, double amount) override {
        auto cents = static_cast<std::int64_t>(std::llround(amount * 100));
        if (int status = client_.makePayment(cents, "BRL", orderId); status != 0)
            throw std::runtime_error("legacy status " + std::to_string(status));
    }
private:
    LegacyPayClient& client_;
};

int main() {
    LegacyPayClient client;
    LegacyPayAdapter adapter(client);
    PaymentGateway& gateway = adapter;
    gateway.charge("order-1", 19.90);
}
```

### Rust
```rust
// Target
trait PaymentGateway {
    fn charge(&self, order_id: &str, amount_cents: u64) -> Result<(), String>;
}

// Adaptee (imagine it comes from an external crate).
struct LegacyPayClient;
impl LegacyPayClient {
    fn make_payment(&self, cents: i64, currency: &str, reference: &str) -> i32 {
        println!("legacy: {cents} {currency} ref={reference}");
        if cents > 0 { 0 } else { 42 }
    }
}

// Newtype adapter wrapping the foreign type. A newtype is mandatory when both the
// trait and the type are foreign (orphan rule).
struct LegacyPayAdapter(LegacyPayClient);

impl PaymentGateway for LegacyPayAdapter {
    fn charge(&self, order_id: &str, amount_cents: u64) -> Result<(), String> {
        let cents = i64::try_from(amount_cents).map_err(|e| e.to_string())?;
        match self.0.make_payment(cents, "BRL", order_id) {
            0 => Ok(()),
            status => Err(format!("legacy status {status}")),
        }
    }
}

fn main() {
    let gateway: &dyn PaymentGateway = &LegacyPayAdapter(LegacyPayClient);
    println!("{:?}", gateway.charge("order-1", 1990));
}
```

### Python
```python
from decimal import Decimal
from typing import Protocol


class PaymentGateway(Protocol):
    def charge(self, order_id: str, amount: Decimal) -> None: ...


class LegacyPayClient:
    def make_payment(self, cents: int, currency: str, reference: str) -> int:
        print(f"legacy: {cents} {currency} ref={reference}")
        return 0 if cents > 0 else 42


class PaymentFailed(Exception):
    pass


class LegacyPayAdapter:
    def __init__(self, client: LegacyPayClient) -> None:
        self._client = client

    def charge(self, order_id: str, amount: Decimal) -> None:
        cents = int(amount * 100)
        status = self._client.make_payment(cents, "BRL", order_id)
        if status != 0:
            raise PaymentFailed(f"legacy status {status}")


if __name__ == "__main__":
    gateway: PaymentGateway = LegacyPayAdapter(LegacyPayClient())
    gateway.charge("order-1", Decimal("19.90"))
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `Arrays.asList`, `InputStreamReader` (bytes → chars) are JDK adapters. Hexagonal architecture formalizes adapters. |
| Kotlin | Extension functions adapt APIs without a wrapper type when no interface is needed. |
| Go | Consumers define small interfaces; adapter types and function adapters (`http.HandlerFunc`) are common. |
| C++ | Object adapter by composition; class adapter via multiple inheritance; `std::stack` is a container adapter. |
| Rust | Newtype pattern to satisfy the orphan rule; `From`/`Into` for value conversions. |
| Python | Duck typing often removes the need; `Protocol` documents the target. |

### Referências / References
- *Design Patterns* (GoF), Adapter
- Alistair Cockburn, [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- *Domain-Driven Design*, Eric Evans: Anti-Corruption Layer
- [Refactoring.Guru: Adapter](https://refactoring.guru/design-patterns/adapter)
