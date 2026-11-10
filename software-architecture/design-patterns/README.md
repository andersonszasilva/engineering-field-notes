# Design Patterns (GoF)

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns
**Tags:** gof, design-patterns, creational, structural, behavioral, trade-offs

[Português](#português) | [English](#english) | [Catálogo / Catalog](#catálogo--catalog)

---

## Português

### Contexto
Os 23 padrões do livro *Design Patterns: Elements of Reusable Object-Oriented Software* (1994), de Erich Gamma, Richard Helm, Ralph Johnson e John Vlissides, a **Gang of Four (GoF)**, são um vocabulário comum para soluções recorrentes de design orientado a objetos. Trinta anos depois, muitos deles viraram recursos das linguagens (iteradores, closures, `object`, pattern matching), mas os **problemas** e os **trade-offs** continuam os mesmos.

Cada nota desta pasta traz: intenção, problema, estrutura, **tabela de trade-offs**, quando usar/evitar e **exemplos executáveis em Java, Kotlin, Go, C++, Rust e Python** (as linguagens de [programming-languages](../../programming-languages/)), com notas sobre a forma idiomática em cada uma.

### Categorias
- **Criacionais**: como objetos são criados; escondem `new` e decidem *o que* e *quando* instanciar.
- **Estruturais**: como classes e objetos são compostos em estruturas maiores.
- **Comportamentais**: como objetos se comunicam e distribuem responsabilidades.

### Como usar padrões sem exagero
1. **Comece pelo problema, não pelo padrão.** Um padrão aplicado sem a dor que ele resolve é só indireção.
2. **Prefira composição a herança.** A maioria dos padrões modernos (Strategy, Decorator, Bridge) é composição.
3. **Use o que a linguagem já oferece.** Em Kotlin, `object` é Singleton; em Python, funções são Strategy e Command; em Rust, `enum` + `match` substitui Visitor e State; em Go, funções e interfaces pequenas substituem hierarquias.
4. **Pague a abstração quando houver a segunda implementação** (ou uma necessidade real de teste/isolamento), não antes.
5. **Nomeie pelo padrão** quando ajudar a comunicação (`PaymentGatewayAdapter`, `RetryDecorator`): o vocabulário compartilhado é metade do valor.

---

## English

### Context
The 23 patterns from *Design Patterns: Elements of Reusable Object-Oriented Software* (1994), by Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides, the **Gang of Four (GoF)**, are a shared vocabulary for recurring object-oriented design solutions. Thirty years later, many of them became language features (iterators, closures, `object`, pattern matching), but the **problems** and **trade-offs** are still the same.

Each note in this folder covers: intent, problem, structure, a **trade-off table**, when to use/avoid, and **runnable examples in Java, Kotlin, Go, C++, Rust, and Python** (the languages in [programming-languages](../../programming-languages/)), with notes on the idiomatic form in each one.

### Categories
- **Creational**: how objects are created; they hide `new` and decide *what* and *when* to instantiate.
- **Structural**: how classes and objects are composed into larger structures.
- **Behavioral**: how objects communicate and distribute responsibilities.

### Using patterns without overdoing it
1. **Start with the problem, not the pattern.** A pattern applied without the pain it solves is just indirection.
2. **Prefer composition over inheritance.** Most modern patterns (Strategy, Decorator, Bridge) are composition.
3. **Use what the language already offers.** In Kotlin, `object` is a Singleton; in Python, functions are Strategy and Command; in Rust, `enum` + `match` replaces Visitor and State; in Go, functions and small interfaces replace hierarchies.
4. **Pay for the abstraction when the second implementation arrives** (or a real testing/isolation need), not before.
5. **Name things after the pattern** when it helps communication (`PaymentGatewayAdapter`, `RetryDecorator`): the shared vocabulary is half the value.

---

## Catálogo / Catalog

### Creational
| Pattern | Intent (EN) | Example scenario | Main trade-off |
|---------|-------------|------------------|----------------|
| [Singleton](creational/singleton.md) | One instance, global access point. | App configuration | Convenience vs hidden global state and hard testing |
| [Factory Method](creational/factory-method.md) | Let subclasses/functions decide which class to instantiate. | Notification channels | Decoupling vs extra types and indirection |
| [Abstract Factory](creational/abstract-factory.md) | Create families of related objects. | AWS vs GCP storage + queue | Family consistency vs rigid interface for new products |
| [Builder](creational/builder.md) | Build complex objects step by step. | Immutable HTTP request | Readability/validation vs boilerplate |
| [Prototype](creational/prototype.md) | Create objects by cloning. | Document templates | Cheap copies vs shallow/deep copy bugs |

### Structural
| Pattern | Intent (EN) | Example scenario | Main trade-off |
|---------|-------------|------------------|----------------|
| [Adapter](structural/adapter.md) | Convert an interface into the one clients expect. | Legacy payment SDK | Isolation from vendors vs extra layer |
| [Bridge](structural/bridge.md) | Decouple abstraction from implementation. | Alert types × channels | N + M instead of N × M vs upfront design |
| [Composite](structural/composite.md) | Treat parts and wholes uniformly in trees. | Files and directories | Uniformity vs overly generic interface |
| [Decorator](structural/decorator.md) | Add responsibilities dynamically by wrapping. | Caching/logging fetcher | Composable behavior vs order-dependent stacks |
| [Facade](structural/facade.md) | Simplified interface to a subsystem. | Checkout orchestration | Simplicity vs god object risk |
| [Flyweight](structural/flyweight.md) | Share state to support many fine-grained objects. | 100k map markers | Memory savings vs complexity and immutability constraints |
| [Proxy](structural/proxy.md) | Surrogate that controls access to an object. | Lazy-loaded protected image | Transparent control vs hidden latency/failures |

### Behavioral
| Pattern | Intent (EN) | Example scenario | Main trade-off |
|---------|-------------|------------------|----------------|
| [Chain of Responsibility](behavioral/chain-of-responsibility.md) | Pass a request along a chain of handlers. | HTTP middleware | Composable steps vs implicit order |
| [Command](behavioral/command.md) | Encapsulate a request as an object. | Editor with undo | Undo/queue/audit vs one type per operation |
| [Interpreter](behavioral/interpreter.md) | Represent and evaluate a grammar. | Feature-flag rules | Rules as data vs building a language |
| [Iterator](behavioral/iterator.md) | Traverse without exposing internals. | Paginated API | Lazy and uniform vs invalidation and single-pass |
| [Mediator](behavioral/mediator.md) | Centralize how objects interact. | Chat room | Decoupled colleagues vs god mediator |
| [Memento](behavioral/memento.md) | Capture and restore state without breaking encapsulation. | Settings snapshots | Reliable rollback vs memory cost |
| [Observer](behavioral/observer.md) | Notify dependents of state changes. | `OrderPaid` event bus | Decoupling vs hard-to-trace flow and leaks |
| [State](behavioral/state.md) | Change behavior when internal state changes. | Order lifecycle | Explicit transitions vs many classes |
| [Strategy](behavioral/strategy.md) | Interchangeable algorithms. | Shipping cost policies | Flexibility vs client must choose |
| [Template Method](behavioral/template-method.md) | Algorithm skeleton with overridable steps. | ETL job | Reused flow vs inheritance coupling |
| [Visitor](behavioral/visitor.md) | Add operations without changing element classes. | Shapes: area and SVG | New operations easy vs new types hard |

### Padrões que se confundem / Commonly confused
| Pair | Difference |
|------|------------|
| Factory Method vs Abstract Factory | One product vs a **family** of compatible products. |
| Adapter vs Bridge | Adapter fixes incompatibility **after the fact**; Bridge is designed **up front** for independent variation. |
| Decorator vs Proxy | Decorator **adds behavior** (composed by the client); Proxy **controls access** (often owns the real object's lifecycle). |
| Strategy vs State | Strategy is **chosen by the client**; State objects **choose the next state** themselves. |
| Command vs Memento (undo) | Command stores the **operation** (and its inverse); Memento stores a **snapshot** of state. |
| Mediator vs Observer | Mediator **centralizes coordination logic**; Observer just **broadcasts** notifications. |
| Composite vs Decorator | Both wrap via a shared interface; Composite has **many children** (tree), Decorator wraps **one** object. |

### Referências / References
- *Design Patterns: Elements of Reusable Object-Oriented Software*, Gamma, Helm, Johnson, Vlissides (1994)
- *Head First Design Patterns*, Eric Freeman & Elisabeth Robson
- *Effective Java*, Joshua Bloch
- [Refactoring.Guru: Design Patterns](https://refactoring.guru/design-patterns)
- Peter Norvig, [Design Patterns in Dynamic Languages](https://norvig.com/design-patterns/) (16 of 23 patterns become simpler or invisible)
