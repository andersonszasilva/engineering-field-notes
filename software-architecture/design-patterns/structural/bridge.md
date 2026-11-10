# Bridge

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, bridge, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
**Desacoplar uma abstração da sua implementação**, para que as duas possam variar independentemente.

### Problema
Há alertas de vários tipos (`Info`, `Critical`) enviados por vários canais (`Slack`, `E-mail`, `PagerDuty`). Modelar com herança gera uma explosão combinatória: `CriticalSlackAlert`, `CriticalEmailAlert`, `InfoSlackAlert`... São N × M classes, e cada novo canal ou tipo multiplica o problema.

### Estrutura
- **Abstraction**: a interface de alto nível usada pelo cliente (`Alert`); guarda uma referência ao Implementor.
- **RefinedAbstraction**: variações da abstração (`CriticalAlert`, `InfoAlert`).
- **Implementor**: interface de baixo nível (`Channel.deliver()`).
- **ConcreteImplementor**: implementações (`SlackChannel`, `EmailChannel`).

Resultado: N + M classes em vez de N × M. A "ponte" é a referência da abstração para o implementor (composição em vez de herança).

### Trade-offs
| Prós | Contras |
|------|---------|
| Elimina a explosão combinatória de subclasses. | Mais complexo de entender à primeira vista (duas hierarquias). |
| Abstração e implementação evoluem e são testadas separadamente. | Projetar a fronteira certa exige conhecer bem as dimensões de variação. |
| Implementação trocável em runtime. | Overkill se só uma das dimensões varia de verdade. |
| Esconde detalhes de plataforma do cliente. | Indireção extra em cada chamada. |

### Quando usar / evitar
- **Use** quando há duas (ou mais) dimensões ortogonais de variação: tipo × plataforma, formato × destino, forma × renderizador.
- **Evite** quando uma das dimensões tem uma única implementação; uma interface simples basta.

Bridge vs Adapter: o Adapter é aplicado **depois** para fazer interfaces existentes conversarem; o Bridge é desenhado **antes** para permitir variação independente.

---

## English

### Intent
**Decouple an abstraction from its implementation** so that the two can vary independently.

### Problem
There are several alert types (`Info`, `Critical`) sent through several channels (`Slack`, `Email`, `PagerDuty`). Modeling this with inheritance leads to a combinatorial explosion: `CriticalSlackAlert`, `CriticalEmailAlert`, `InfoSlackAlert`... N × M classes, and each new channel or type multiplies the problem.

### Structure
- **Abstraction**: the high-level interface used by the client (`Alert`); holds a reference to the Implementor.
- **RefinedAbstraction**: variations of the abstraction (`CriticalAlert`, `InfoAlert`).
- **Implementor**: a low-level interface (`Channel.deliver()`).
- **ConcreteImplementor**: implementations (`SlackChannel`, `EmailChannel`).

Result: N + M classes instead of N × M. The "bridge" is the abstraction's reference to the implementor (composition over inheritance).

### Trade-offs
| Pros | Cons |
|------|------|
| Eliminates the combinatorial explosion of subclasses. | Harder to grasp at first (two hierarchies). |
| Abstraction and implementation evolve and are tested separately. | Drawing the right boundary requires knowing the dimensions of change well. |
| Implementation can be swapped at runtime. | Overkill if only one dimension really varies. |
| Hides platform details from the client. | Extra indirection on each call. |

### When to use / avoid
- **Use** when there are two (or more) orthogonal dimensions of variation: type × platform, format × destination, shape × renderer.
- **Avoid** when one of the dimensions has a single implementation; a plain interface is enough.

Bridge vs Adapter: Adapter is applied **after the fact** to make existing interfaces talk; Bridge is designed **up front** to allow independent variation.

---

## Exemplos / Examples

Scenario: alert types (abstraction) × delivery channels (implementation).

### Java
```java
// Implementor
interface Channel {
    void deliver(String text);
}

class SlackChannel implements Channel {
    public void deliver(String text) { System.out.println("[slack] " + text); }
}

class EmailChannel implements Channel {
    public void deliver(String text) { System.out.println("[email] " + text); }
}

// Abstraction: holds the bridge to the implementor.
abstract class Alert {
    protected final Channel channel;
    Alert(Channel channel) { this.channel = channel; }
    abstract void raise(String message);
}

class InfoAlert extends Alert {
    InfoAlert(Channel channel) { super(channel); }
    void raise(String message) { channel.deliver("INFO: " + message); }
}

class CriticalAlert extends Alert {
    CriticalAlert(Channel channel) { super(channel); }
    void raise(String message) {
        channel.deliver("CRITICAL: " + message.toUpperCase());
        channel.deliver("(escalating to on-call)");
    }
}

class Main {
    public static void main(String[] args) {
        new InfoAlert(new EmailChannel()).raise("deploy finished");
        new CriticalAlert(new SlackChannel()).raise("db replica lag > 30s");
    }
}
```

### Kotlin
```kotlin
fun interface Channel {
    fun deliver(text: String)
}

val slack = Channel { println("[slack] $it") }
val email = Channel { println("[email] $it") }

sealed class Alert(protected val channel: Channel) {
    abstract fun raise(message: String)
}

class InfoAlert(channel: Channel) : Alert(channel) {
    override fun raise(message: String) = channel.deliver("INFO: $message")
}

class CriticalAlert(channel: Channel) : Alert(channel) {
    override fun raise(message: String) {
        channel.deliver("CRITICAL: ${message.uppercase()}")
        channel.deliver("(escalating to on-call)")
    }
}

fun main() {
    InfoAlert(email).raise("deploy finished")
    CriticalAlert(slack).raise("db replica lag > 30s")
}
```

### Go
```go
package main

import (
	"fmt"
	"strings"
)

// Implementor
type Channel interface {
	Deliver(text string)
}

type SlackChannel struct{}

func (SlackChannel) Deliver(text string) { fmt.Println("[slack]", text) }

type EmailChannel struct{}

func (EmailChannel) Deliver(text string) { fmt.Println("[email]", text) }

// Abstraction
type Alert interface {
	Raise(message string)
}

type InfoAlert struct{ Channel Channel }

func (a InfoAlert) Raise(msg string) { a.Channel.Deliver("INFO: " + msg) }

type CriticalAlert struct{ Channel Channel }

func (a CriticalAlert) Raise(msg string) {
	a.Channel.Deliver("CRITICAL: " + strings.ToUpper(msg))
	a.Channel.Deliver("(escalating to on-call)")
}

func main() {
	alerts := []Alert{
		InfoAlert{Channel: EmailChannel{}},
		CriticalAlert{Channel: SlackChannel{}},
	}
	for _, a := range alerts {
		a.Raise("db replica lag > 30s")
	}
}
```

### C++
```cpp
#include <algorithm>
#include <cctype>
#include <iostream>
#include <memory>
#include <string>

// Implementor
struct Channel {
    virtual ~Channel() = default;
    virtual void deliver(const std::string& text) = 0;
};
struct SlackChannel : Channel {
    void deliver(const std::string& t) override { std::cout << "[slack] " << t << "\n"; }
};
struct EmailChannel : Channel {
    void deliver(const std::string& t) override { std::cout << "[email] " << t << "\n"; }
};

// Abstraction
class Alert {
public:
    explicit Alert(std::shared_ptr<Channel> c) : channel_(std::move(c)) {}
    virtual ~Alert() = default;
    virtual void raise(const std::string& msg) = 0;
protected:
    std::shared_ptr<Channel> channel_;
};

class InfoAlert : public Alert {
public:
    using Alert::Alert;
    void raise(const std::string& msg) override { channel_->deliver("INFO: " + msg); }
};

class CriticalAlert : public Alert {
public:
    using Alert::Alert;
    void raise(const std::string& msg) override {
        std::string up = msg;
        std::transform(up.begin(), up.end(), up.begin(), [](unsigned char c) { return std::toupper(c); });
        channel_->deliver("CRITICAL: " + up);
        channel_->deliver("(escalating to on-call)");
    }
};

int main() {
    InfoAlert(std::make_shared<EmailChannel>()).raise("deploy finished");
    CriticalAlert(std::make_shared<SlackChannel>()).raise("db replica lag > 30s");
}
```

### Rust
```rust
// Implementor
trait Channel {
    fn deliver(&self, text: &str);
}

struct Slack;
impl Channel for Slack {
    fn deliver(&self, text: &str) { println!("[slack] {text}"); }
}

struct Email;
impl Channel for Email {
    fn deliver(&self, text: &str) { println!("[email] {text}"); }
}

// Abstraction: generic over the implementor (static dispatch).
// Use Box<dyn Channel> instead if the channel is chosen at runtime.
trait Alert {
    fn raise(&self, message: &str);
}

struct InfoAlert<C: Channel> { channel: C }
impl<C: Channel> Alert for InfoAlert<C> {
    fn raise(&self, message: &str) { self.channel.deliver(&format!("INFO: {message}")); }
}

struct CriticalAlert<C: Channel> { channel: C }
impl<C: Channel> Alert for CriticalAlert<C> {
    fn raise(&self, message: &str) {
        self.channel.deliver(&format!("CRITICAL: {}", message.to_uppercase()));
        self.channel.deliver("(escalating to on-call)");
    }
}

fn main() {
    InfoAlert { channel: Email }.raise("deploy finished");
    CriticalAlert { channel: Slack }.raise("db replica lag > 30s");
}
```

### Python
```python
from abc import ABC, abstractmethod
from typing import Protocol


class Channel(Protocol):
    def deliver(self, text: str) -> None: ...


class SlackChannel:
    def deliver(self, text: str) -> None:
        print(f"[slack] {text}")


class EmailChannel:
    def deliver(self, text: str) -> None:
        print(f"[email] {text}")


class Alert(ABC):
    def __init__(self, channel: Channel) -> None:
        self.channel = channel

    @abstractmethod
    def raise_alert(self, message: str) -> None: ...


class InfoAlert(Alert):
    def raise_alert(self, message: str) -> None:
        self.channel.deliver(f"INFO: {message}")


class CriticalAlert(Alert):
    def raise_alert(self, message: str) -> None:
        self.channel.deliver(f"CRITICAL: {message.upper()}")
        self.channel.deliver("(escalating to on-call)")


if __name__ == "__main__":
    InfoAlert(EmailChannel()).raise_alert("deploy finished")
    CriticalAlert(SlackChannel()).raise_alert("db replica lag > 30s")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | JDBC is a bridge: `java.sql` API (abstraction) × vendor drivers (implementation). SLF4J × logging backends too. |
| Kotlin | `fun interface` makes implementors one-liners; `sealed` abstractions keep the hierarchy closed. |
| Go | Struct embedding/fields holding an interface; the abstraction is often just a struct with an interface field. |
| C++ | Pimpl idiom is a bridge used to hide implementation and cut compile-time dependencies. |
| Rust | Generics (`Alert<C: Channel>`) for zero-cost bridges; `Box<dyn Channel>` for runtime choice. |
| Python | Composition with duck-typed implementors; the bridge is just an attribute. |

### Referências / References
- *Design Patterns* (GoF), Bridge
- [Refactoring.Guru: Bridge](https://refactoring.guru/design-patterns/bridge)
