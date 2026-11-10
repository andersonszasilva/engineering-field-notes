# Prototype

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / creational
**Tags:** gof, design-patterns, creational, prototype, clone, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Criar novos objetos **copiando uma instância existente** (o protótipo), em vez de construí-los do zero.

### Problema
Às vezes construir um objeto é caro (carrega dados, faz parsing, consulta um serviço) ou ele já está configurado de um jeito que queremos reaproveitar. Também pode ser que o código só tenha uma referência pela interface e não saiba a classe concreta para chamar o construtor.

### Estrutura
- **Prototype**: interface com uma operação `clone()`/`copy()`.
- **ConcretePrototype**: implementa a cópia, decidindo o que é **cópia rasa** (compartilha referências) e o que é **cópia profunda** (duplica o objeto interno).
- **Registry** (opcional): catálogo de protótipos pré-configurados por nome.

### Trade-offs
| Prós | Contras |
|------|---------|
| Evita repetir inicializações caras. | **Cópia rasa vs profunda** é a fonte clássica de bugs: duas cópias mutando a mesma lista. |
| Clona sem conhecer a classe concreta. | Referências circulares e recursos (sockets, arquivos) complicam a cópia. |
| Registry de protótipos substitui hierarquias de fábricas. | `Cloneable`/`clone()` do Java é considerado mal projetado. |
| Bom para "templates" configuráveis em runtime. | Com objetos imutáveis, a cópia raramente é necessária (`with`/`copy`). |

### Quando usar / evitar
- **Use** para templates (documentos, configurações, entidades de jogo), objetos caros de inicializar e quando o tipo concreto é desconhecido.
- **Evite** com objetos imutáveis (basta compartilhar) ou quando um construtor/factory simples resolve.

---

## English

### Intent
Create new objects by **copying an existing instance** (the prototype) rather than building them from scratch.

### Problem
Sometimes building an object is expensive (loads data, parses, calls a service), or it is already configured in a way we want to reuse. The code might also only hold a reference through an interface and not know the concrete class to call its constructor.

### Structure
- **Prototype**: interface with a `clone()`/`copy()` operation.
- **ConcretePrototype**: implements the copy, deciding what is a **shallow copy** (shares references) and what is a **deep copy** (duplicates inner objects).
- **Registry** (optional): a catalog of preconfigured prototypes by name.

### Trade-offs
| Pros | Cons |
|------|------|
| Avoids repeating expensive initialization. | **Shallow vs deep copy** is the classic bug source: two copies mutating the same list. |
| Clones without knowing the concrete class. | Circular references and resources (sockets, files) make copying hard. |
| A prototype registry replaces factory hierarchies. | Java's `Cloneable`/`clone()` is widely considered broken. |
| Good for runtime-configurable "templates". | With immutable objects, copying is rarely needed (`with`/`copy`). |

### When to use / avoid
- **Use** for templates (documents, configurations, game entities), objects that are expensive to initialize, and when the concrete type is unknown.
- **Avoid** with immutable objects (just share them) or when a simple constructor/factory does the job.

---

## Exemplos / Examples

Scenario: a registry of document templates; each new document is a **deep copy** so editing its sections doesn't affect the template.

### Java
```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

interface Prototype<T> { T copy(); }

class Document implements Prototype<Document> {
    String title;
    final List<String> sections;

    Document(String title, List<String> sections) {
        this.title = title;
        this.sections = sections;
    }

    // Copy constructor style instead of Cloneable; deep-copies the mutable list.
    @Override public Document copy() {
        return new Document(title, new ArrayList<>(sections));
    }

    @Override public String toString() { return title + " " + sections; }
}

class TemplateRegistry {
    private final Map<String, Document> templates = new HashMap<>();
    void register(String name, Document doc) { templates.put(name, doc); }
    Document create(String name) { return templates.get(name).copy(); }
}

class Main {
    public static void main(String[] args) {
        var registry = new TemplateRegistry();
        registry.register("adr", new Document("ADR", new ArrayList<>(List.of("Context", "Decision", "Consequences"))));

        Document adr = registry.create("adr");
        adr.title = "ADR-001: Use Kafka";
        adr.sections.add("Alternatives");

        System.out.println(adr);                     // ADR-001: Use Kafka [Context, Decision, Consequences, Alternatives]
        System.out.println(registry.create("adr")); // ADR [Context, Decision, Consequences]
    }
}
```

### Kotlin
```kotlin
// data class `copy()` is shallow: mutable collections must be copied explicitly.
data class Document(val title: String, val sections: MutableList<String>) {
    fun deepCopy(title: String = this.title) = copy(title = title, sections = sections.toMutableList())
}

class TemplateRegistry {
    private val templates = mutableMapOf<String, Document>()
    fun register(name: String, doc: Document) { templates[name] = doc }
    fun create(name: String): Document = templates.getValue(name).deepCopy()
}

fun main() {
    val registry = TemplateRegistry()
    registry.register("adr", Document("ADR", mutableListOf("Context", "Decision", "Consequences")))

    val adr = registry.create("adr").deepCopy(title = "ADR-001: Use Kafka")
    adr.sections += "Alternatives"

    println(adr)
    println(registry.create("adr")) // template unchanged
}
```

### Go
```go
package main

import (
	"fmt"
	"slices"
)

type Document struct {
	Title    string
	Sections []string
}

// Assigning a struct copies it, but slices/maps still share backing storage.
func (d *Document) Clone() *Document {
	return &Document{Title: d.Title, Sections: slices.Clone(d.Sections)}
}

type TemplateRegistry map[string]*Document

func (r TemplateRegistry) Create(name string) *Document { return r[name].Clone() }

func main() {
	registry := TemplateRegistry{
		"adr": {Title: "ADR", Sections: []string{"Context", "Decision", "Consequences"}},
	}

	adr := registry.Create("adr")
	adr.Title = "ADR-001: Use Kafka"
	adr.Sections = append(adr.Sections, "Alternatives")

	fmt.Println(*adr)
	fmt.Println(*registry.Create("adr")) // template unchanged
}
```

### C++
```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>
#include <vector>

// Polymorphic clone: copies through a base pointer without knowing the concrete type.
struct Prototype {
    virtual ~Prototype() = default;
    virtual std::unique_ptr<Prototype> clone() const = 0;
    virtual void print() const = 0;
};

struct Document : Prototype {
    std::string title;
    std::vector<std::string> sections;  // value semantics: the copy constructor deep-copies

    Document(std::string t, std::vector<std::string> s) : title(std::move(t)), sections(std::move(s)) {}

    std::unique_ptr<Prototype> clone() const override { return std::make_unique<Document>(*this); }

    void print() const override {
        std::cout << title << ":";
        for (const auto& s : sections) std::cout << " " << s;
        std::cout << "\n";
    }
};

int main() {
    std::map<std::string, std::unique_ptr<Prototype>> registry;
    registry["adr"] = std::make_unique<Document>("ADR", std::vector<std::string>{"Context", "Decision", "Consequences"});

    auto copy = registry["adr"]->clone();
    auto* adr = static_cast<Document*>(copy.get());
    adr->title = "ADR-001: Use Kafka";
    adr->sections.push_back("Alternatives");

    adr->print();
    registry["adr"]->print();  // template unchanged
}
```

### Rust
```rust
use std::collections::HashMap;

// `#[derive(Clone)]` is a deep copy of owned data (Vec, String).
// Shared data must be explicit (`Rc`/`Arc`), so shallow vs deep is visible in the types.
#[derive(Clone, Debug)]
struct Document {
    title: String,
    sections: Vec<String>,
}

struct TemplateRegistry {
    templates: HashMap<String, Document>,
}

impl TemplateRegistry {
    fn create(&self, name: &str) -> Option<Document> {
        self.templates.get(name).cloned()
    }
}

fn main() {
    let mut templates = HashMap::new();
    templates.insert(
        "adr".to_string(),
        Document { title: "ADR".into(), sections: vec!["Context".into(), "Decision".into(), "Consequences".into()] },
    );
    let registry = TemplateRegistry { templates };

    let mut adr = registry.create("adr").unwrap();
    adr.title = "ADR-001: Use Kafka".into();
    adr.sections.push("Alternatives".into());

    println!("{adr:?}");
    println!("{:?}", registry.create("adr").unwrap()); // template unchanged
}
```

### Python
```python
import copy
from dataclasses import dataclass, field


@dataclass
class Document:
    title: str
    sections: list[str] = field(default_factory=list)

    def clone(self) -> "Document":
        # copy.copy would share `sections`; deepcopy duplicates nested objects.
        return copy.deepcopy(self)


class TemplateRegistry:
    def __init__(self) -> None:
        self._templates: dict[str, Document] = {}

    def register(self, name: str, doc: Document) -> None:
        self._templates[name] = doc

    def create(self, name: str) -> Document:
        return self._templates[name].clone()


if __name__ == "__main__":
    registry = TemplateRegistry()
    registry.register("adr", Document("ADR", ["Context", "Decision", "Consequences"]))

    adr = registry.create("adr")
    adr.title = "ADR-001: Use Kafka"
    adr.sections.append("Alternatives")

    print(adr)
    print(registry.create("adr"))  # template unchanged
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Prefer copy constructors or static `copyOf` over `Cloneable` (*Effective Java*, Item 13). Records + `with`-style methods for immutable copies. |
| Kotlin | `data class copy()` is shallow; copy mutable collections yourself, or use immutable collections. |
| Go | Struct assignment copies values, not the backing arrays of slices/maps; use `slices.Clone`/`maps.Clone`. |
| C++ | Copy constructors are deep for value members; virtual `clone()` returning `unique_ptr` for polymorphic copies. |
| Rust | `Clone` trait, explicit `.clone()`; `Rc`/`Arc` make sharing explicit. |
| Python | `copy.copy` (shallow) vs `copy.deepcopy`; customize with `__copy__`/`__deepcopy__`. |

### Referências / References
- *Design Patterns* (GoF), Prototype
- *Effective Java*, Joshua Bloch, Item 13: override `clone` judiciously
- [Refactoring.Guru: Prototype](https://refactoring.guru/design-patterns/prototype)
