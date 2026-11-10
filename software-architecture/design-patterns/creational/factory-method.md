# Factory Method

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / creational
**Tags:** gof, design-patterns, creational, factory-method, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Definir uma interface para criar um objeto, mas deixar que **subclasses (ou implementações) decidam qual classe concreta instanciar**.

### Problema
Um componente precisa criar objetos, mas não deveria conhecer as classes concretas. Exemplo: um serviço de notificações que envia por e-mail, SMS ou push. Se o código cliente faz `new EmailSender()` em vários lugares, adicionar um novo canal exige alterar todos eles.

### Estrutura
- **Product**: interface comum dos objetos criados (`Notifier`).
- **ConcreteProduct**: implementações (`EmailNotifier`, `SmsNotifier`).
- **Creator**: declara o método fábrica (`createNotifier()`) e, muitas vezes, contém lógica que usa o produto.
- **ConcreteCreator**: sobrescreve o método fábrica para devolver um produto específico.

Na prática moderna, o "método fábrica" aparece muito como uma **função fábrica parametrizada** (`create(channel)`) ou como uma função passada por injeção.

### Trade-offs
| Prós | Contras |
|------|---------|
| Desacopla o cliente das classes concretas. | Mais classes/tipos para algo que às vezes é só um `new`. |
| Open/Closed: novos produtos sem mudar o código cliente. | A versão por herança (subclasse por produto) gera hierarquias paralelas. |
| Ponto único para lógica de criação (cache, validação, configuração). | A fábrica parametrizada com `switch` volta a precisar de edição para cada produto novo. |
| Facilita testes: a fábrica pode devolver fakes. | Indireção extra dificulta seguir o código. |

### Quando usar / evitar
- **Use** quando o tipo exato só é conhecido em runtime (configuração, input, feature flag) ou quando frameworks precisam deixar extensões criarem objetos.
- **Evite** quando há uma só implementação e nenhuma perspectiva real de outra: um construtor basta.

---

## English

### Intent
Define an interface for creating an object, but let **subclasses (or implementations) decide which concrete class to instantiate**.

### Problem
A component needs to create objects but shouldn't know the concrete classes. Example: a notification service that sends via email, SMS, or push. If client code does `new EmailSender()` in many places, adding a new channel means changing all of them.

### Structure
- **Product**: common interface of the created objects (`Notifier`).
- **ConcreteProduct**: implementations (`EmailNotifier`, `SmsNotifier`).
- **Creator**: declares the factory method (`createNotifier()`) and often contains logic that uses the product.
- **ConcreteCreator**: overrides the factory method to return a specific product.

In modern code, the "factory method" often shows up as a **parameterized factory function** (`create(channel)`) or as a function passed via injection.

### Trade-offs
| Pros | Cons |
|------|------|
| Decouples the client from concrete classes. | More classes/types for something that is sometimes just a `new`. |
| Open/Closed: new products without changing client code. | The inheritance version (one subclass per product) creates parallel hierarchies. |
| Single place for creation logic (caching, validation, configuration). | A parameterized factory with a `switch` needs editing for each new product. |
| Eases testing: the factory can return fakes. | Extra indirection makes code harder to follow. |

### When to use / avoid
- **Use** when the exact type is only known at runtime (config, input, feature flag) or when frameworks need to let extensions create objects.
- **Avoid** when there is a single implementation and no real prospect of another: a constructor is enough.

---

## Exemplos / Examples

Scenario: a notification service creates a `Notifier` for a given channel.

### Java
```java
interface Notifier {
    void send(String to, String message);
}

class EmailNotifier implements Notifier {
    public void send(String to, String message) {
        System.out.println("Email to " + to + ": " + message);
    }
}

class SmsNotifier implements Notifier {
    public void send(String to, String message) {
        System.out.println("SMS to " + to + ": " + message);
    }
}

// Classic GoF: the creator holds the workflow; subclasses decide the product.
abstract class NotificationService {
    protected abstract Notifier createNotifier();

    void notifyUser(String to, String message) {
        createNotifier().send(to, message);
    }
}

class EmailNotificationService extends NotificationService {
    protected Notifier createNotifier() { return new EmailNotifier(); }
}

// Modern variant: a parameterized static factory.
final class Notifiers {
    static Notifier of(String channel) {
        return switch (channel) {
            case "email" -> new EmailNotifier();
            case "sms" -> new SmsNotifier();
            default -> throw new IllegalArgumentException("unknown channel: " + channel);
        };
    }
}

class Main {
    public static void main(String[] args) {
        new EmailNotificationService().notifyUser("ana@example.com", "Welcome!");
        Notifiers.of("sms").send("+5511999999999", "Your code is 1234");
    }
}
```

### Kotlin
```kotlin
interface Notifier {
    fun send(to: String, message: String)
}

class EmailNotifier : Notifier {
    override fun send(to: String, message: String) = println("Email to $to: $message")
}

class SmsNotifier : Notifier {
    override fun send(to: String, message: String) = println("SMS to $to: $message")
}

abstract class NotificationService {
    protected abstract fun createNotifier(): Notifier
    fun notifyUser(to: String, message: String) = createNotifier().send(to, message)
}

class SmsNotificationService : NotificationService() {
    override fun createNotifier(): Notifier = SmsNotifier()
}

// Idiomatic: a top-level factory function with an exhaustive `when`.
fun notifierFor(channel: String): Notifier = when (channel) {
    "email" -> EmailNotifier()
    "sms" -> SmsNotifier()
    else -> throw IllegalArgumentException("unknown channel: $channel")
}

fun main() {
    SmsNotificationService().notifyUser("+5511999999999", "Your code is 1234")
    notifierFor("email").send("ana@example.com", "Welcome!")
}
```

### Go
```go
package main

import "fmt"

type Notifier interface {
	Send(to, message string)
}

type EmailNotifier struct{}

func (EmailNotifier) Send(to, message string) { fmt.Printf("Email to %s: %s\n", to, message) }

type SmsNotifier struct{}

func (SmsNotifier) Send(to, message string) { fmt.Printf("SMS to %s: %s\n", to, message) }

// Go has no inheritance: the factory method becomes a function (or a field holding one).
func NewNotifier(channel string) (Notifier, error) {
	switch channel {
	case "email":
		return EmailNotifier{}, nil
	case "sms":
		return SmsNotifier{}, nil
	default:
		return nil, fmt.Errorf("unknown channel: %s", channel)
	}
}

// The "creator" receives the factory as a dependency.
type NotificationService struct {
	newNotifier func() Notifier
}

func (s NotificationService) NotifyUser(to, msg string) { s.newNotifier().Send(to, msg) }

func main() {
	n, err := NewNotifier("sms")
	if err != nil {
		panic(err)
	}
	n.Send("+5511999999999", "Your code is 1234")

	svc := NotificationService{newNotifier: func() Notifier { return EmailNotifier{} }}
	svc.NotifyUser("ana@example.com", "Welcome!")
}
```

### C++
```cpp
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

struct Notifier {
    virtual ~Notifier() = default;
    virtual void send(const std::string& to, const std::string& msg) = 0;
};

struct EmailNotifier : Notifier {
    void send(const std::string& to, const std::string& msg) override {
        std::cout << "Email to " << to << ": " << msg << "\n";
    }
};

struct SmsNotifier : Notifier {
    void send(const std::string& to, const std::string& msg) override {
        std::cout << "SMS to " << to << ": " << msg << "\n";
    }
};

class NotificationService {
public:
    virtual ~NotificationService() = default;
    void notifyUser(const std::string& to, const std::string& msg) { createNotifier()->send(to, msg); }
protected:
    virtual std::unique_ptr<Notifier> createNotifier() = 0;  // the factory method
};

class EmailNotificationService : public NotificationService {
protected:
    std::unique_ptr<Notifier> createNotifier() override { return std::make_unique<EmailNotifier>(); }
};

std::unique_ptr<Notifier> makeNotifier(const std::string& channel) {
    if (channel == "email") return std::make_unique<EmailNotifier>();
    if (channel == "sms") return std::make_unique<SmsNotifier>();
    throw std::invalid_argument("unknown channel: " + channel);
}

int main() {
    EmailNotificationService{}.notifyUser("ana@example.com", "Welcome!");
    makeNotifier("sms")->send("+5511999999999", "Your code is 1234");
}
```

### Rust
```rust
trait Notifier {
    fn send(&self, to: &str, message: &str);
}

struct EmailNotifier;
impl Notifier for EmailNotifier {
    fn send(&self, to: &str, message: &str) {
        println!("Email to {to}: {message}");
    }
}

struct SmsNotifier;
impl Notifier for SmsNotifier {
    fn send(&self, to: &str, message: &str) {
        println!("SMS to {to}: {message}");
    }
}

// Rust has no inheritance; a trait with a required "factory" method plays the Creator role.
trait NotificationService {
    fn create_notifier(&self) -> Box<dyn Notifier>;

    fn notify_user(&self, to: &str, message: &str) {
        self.create_notifier().send(to, message);
    }
}

struct SmsService;
impl NotificationService for SmsService {
    fn create_notifier(&self) -> Box<dyn Notifier> {
        Box::new(SmsNotifier)
    }
}

fn notifier_for(channel: &str) -> Result<Box<dyn Notifier>, String> {
    match channel {
        "email" => Ok(Box::new(EmailNotifier)),
        "sms" => Ok(Box::new(SmsNotifier)),
        other => Err(format!("unknown channel: {other}")),
    }
}

fn main() {
    SmsService.notify_user("+5511999999999", "Your code is 1234");
    notifier_for("email").unwrap().send("ana@example.com", "Welcome!");
}
```

### Python
```python
from abc import ABC, abstractmethod
from typing import Protocol


class Notifier(Protocol):
    def send(self, to: str, message: str) -> None: ...


class EmailNotifier:
    def send(self, to: str, message: str) -> None:
        print(f"Email to {to}: {message}")


class SmsNotifier:
    def send(self, to: str, message: str) -> None:
        print(f"SMS to {to}: {message}")


class NotificationService(ABC):
    @abstractmethod
    def create_notifier(self) -> Notifier: ...

    def notify_user(self, to: str, message: str) -> None:
        self.create_notifier().send(to, message)


class EmailNotificationService(NotificationService):
    def create_notifier(self) -> Notifier:
        return EmailNotifier()


# Idiomatic: classes are callables, so a dict registry is a factory.
NOTIFIERS: dict[str, type] = {"email": EmailNotifier, "sms": SmsNotifier}


def notifier_for(channel: str) -> Notifier:
    try:
        return NOTIFIERS[channel]()
    except KeyError:
        raise ValueError(f"unknown channel: {channel}") from None


if __name__ == "__main__":
    EmailNotificationService().notify_user("ana@example.com", "Welcome!")
    notifier_for("sms").send("+5511999999999", "Your code is 1234")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Static factories (`List.of`, `Optional.of`) are the most common form. `ServiceLoader` is a pluggable factory. |
| Kotlin | Top-level factory functions or `companion object { operator fun invoke(...) }`. `sealed` types make `when` exhaustive. |
| Go | `NewXxx` constructor functions returning interfaces; inject `func() T` instead of subclassing. |
| C++ | Return `std::unique_ptr<Base>`; registries map strings to `std::function<std::unique_ptr<Base>()>`. |
| Rust | Return `Box<dyn Trait>` for runtime choice, or `impl Trait`/generics for compile-time choice. |
| Python | Classes are already factories; a `dict[str, type]` registry or `importlib` for plugins. |

### Referências / References
- *Design Patterns* (GoF), Factory Method
- *Effective Java*, Joshua Bloch, Item 1: static factory methods
- [Refactoring.Guru: Factory Method](https://refactoring.guru/design-patterns/factory-method)
