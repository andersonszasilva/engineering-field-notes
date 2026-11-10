# Memento

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, memento, snapshot, undo, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Sem violar o encapsulamento, **capturar e externalizar o estado interno** de um objeto para que ele possa ser **restaurado** depois.

### Problema
Um formulário de configuração deve permitir "desfazer alterações" e "voltar ao último salvo". Guardar o estado exige acessar os campos privados do objeto; expô-los com getters/setters públicos quebra o encapsulamento e permite que qualquer um altere o estado.

### Estrutura
- **Originator**: o objeto cujo estado é salvo; cria mementos (`save()`) e se restaura a partir deles (`restore(m)`).
- **Memento**: snapshot **imutável** e **opaco** do estado; só o Originator entende seu conteúdo.
- **Caretaker**: guarda os mementos (pilha de histórico) sem inspecioná-los nem alterá-los.

Memento vs Command para undo: o Command guarda a **operação** e sabe revertê-la (pouca memória, mas undo de cada operação precisa ser correto); o Memento guarda o **estado** inteiro (simples e robusto, mas consome mais memória).

### Trade-offs
| Prós | Contras |
|------|---------|
| Undo/rollback simples e confiável: restaura o estado exato. | Consumo de memória alto se o estado é grande ou se snapshots são frequentes. |
| Preserva o encapsulamento do Originator. | O caretaker precisa de política de retenção (limite, expiração). |
| Base para checkpoints, transações e *time travel debugging*. | Snapshots profundos de grafos de objetos são caros e difíceis de acertar. |
| | Mementos persistidos viram contrato: mudanças de esquema exigem migração. |

### Quando usar / evitar
- **Use** para undo/redo de estados pequenos, checkpoints de processamento, rollback de transações em memória e snapshots de *event sourcing*.
- **Evite** para estados grandes e mudanças frequentes; prefira Command (deltas) ou estruturas de dados persistentes/imutáveis.

---

## English

### Intent
Without violating encapsulation, **capture and externalize an object's internal state** so that it can be **restored** later.

### Problem
A settings form should allow "undo changes" and "revert to last saved". Saving the state requires access to the object's private fields; exposing them through public getters/setters breaks encapsulation and lets anyone change the state.

### Structure
- **Originator**: the object whose state is saved; creates mementos (`save()`) and restores itself from them (`restore(m)`).
- **Memento**: an **immutable**, **opaque** snapshot of the state; only the Originator understands its contents.
- **Caretaker**: stores mementos (a history stack) without inspecting or modifying them.

Memento vs Command for undo: Command stores the **operation** and knows how to reverse it (little memory, but each operation's undo must be correct); Memento stores the whole **state** (simple and robust, but uses more memory).

### Trade-offs
| Pros | Cons |
|------|------|
| Simple, reliable undo/rollback: restores the exact state. | High memory use if the state is large or snapshots are frequent. |
| Preserves the Originator's encapsulation. | The caretaker needs a retention policy (limit, expiry). |
| Basis for checkpoints, transactions, and *time travel debugging*. | Deep snapshots of object graphs are expensive and hard to get right. |
| | Persisted mementos become a contract: schema changes require migration. |

### When to use / avoid
- **Use** for undo/redo of small states, processing checkpoints, in-memory transaction rollback, and *event sourcing* snapshots.
- **Avoid** for large states with frequent changes; prefer Command (deltas) or persistent/immutable data structures.

---

## Exemplos / Examples

Scenario: a `Settings` object (originator) with a bounded history (caretaker) of opaque snapshots.

### Java
```java
import java.util.ArrayDeque;
import java.util.Deque;

class Settings {
    private String theme = "light";
    private int fontSize = 14;

    // Memento: immutable and opaque (private fields, package-private type).
    static final class Snapshot {
        private final String theme;
        private final int fontSize;
        private Snapshot(String theme, int fontSize) { this.theme = theme; this.fontSize = fontSize; }
    }

    Snapshot save() { return new Snapshot(theme, fontSize); }

    void restore(Snapshot s) { this.theme = s.theme; this.fontSize = s.fontSize; }

    void setTheme(String theme) { this.theme = theme; }
    void setFontSize(int size) { this.fontSize = size; }

    @Override public String toString() { return "theme=" + theme + ", fontSize=" + fontSize; }
}

// Caretaker with a bounded history.
class History {
    private final Deque<Settings.Snapshot> stack = new ArrayDeque<>();
    private final int max;
    History(int max) { this.max = max; }

    void push(Settings.Snapshot s) {
        if (stack.size() == max) stack.removeLast();
        stack.push(s);
    }
    Settings.Snapshot pop() { return stack.pop(); }
}

class Main {
    public static void main(String[] args) {
        var settings = new Settings();
        var history = new History(10);

        history.push(settings.save());
        settings.setTheme("dark");
        history.push(settings.save());
        settings.setFontSize(18);
        System.out.println(settings); // dark, 18

        settings.restore(history.pop());
        System.out.println(settings); // dark, 14
        settings.restore(history.pop());
        System.out.println(settings); // light, 14
    }
}
```

### Kotlin
```kotlin
class Settings {
    var theme = "light"
    var fontSize = 14

    // Kotlin outer classes can't see nested `private` members, so `internal` hides the
    // snapshot's contents from other modules (the closest equivalent to Java's version).
    class Snapshot internal constructor(internal val theme: String, internal val fontSize: Int)

    fun save() = Snapshot(theme, fontSize)
    fun restore(s: Snapshot) { theme = s.theme; fontSize = s.fontSize }
    override fun toString() = "theme=$theme, fontSize=$fontSize"
}

class History(private val max: Int) {
    private val stack = ArrayDeque<Settings.Snapshot>()
    fun push(s: Settings.Snapshot) {
        if (stack.size == max) stack.removeFirst()
        stack.addLast(s)
    }
    fun pop(): Settings.Snapshot? = stack.removeLastOrNull()
}

fun main() {
    val settings = Settings()
    val history = History(10)

    history.push(settings.save())
    settings.theme = "dark"
    history.push(settings.save())
    settings.fontSize = 18
    println(settings)

    history.pop()?.let(settings::restore)
    println(settings)
    history.pop()?.let(settings::restore)
    println(settings)
}
```

### Go
```go
package main

import "fmt"

type Settings struct {
	theme    string
	fontSize int
}

// Memento: unexported fields make it opaque to other packages.
type Snapshot struct {
	theme    string
	fontSize int
}

func (s *Settings) Save() Snapshot     { return Snapshot{s.theme, s.fontSize} }
func (s *Settings) Restore(m Snapshot) { s.theme, s.fontSize = m.theme, m.fontSize }

// Caretaker
type History struct{ stack []Snapshot }

func (h *History) Push(m Snapshot) { h.stack = append(h.stack, m) }
func (h *History) Pop() (Snapshot, bool) {
	if len(h.stack) == 0 {
		return Snapshot{}, false
	}
	m := h.stack[len(h.stack)-1]
	h.stack = h.stack[:len(h.stack)-1]
	return m, true
}

func main() {
	s := &Settings{theme: "light", fontSize: 14}
	h := &History{}

	h.Push(s.Save())
	s.theme = "dark"
	h.Push(s.Save())
	s.fontSize = 18
	fmt.Printf("%+v\n", *s)

	if m, ok := h.Pop(); ok {
		s.Restore(m)
	}
	fmt.Printf("%+v\n", *s)
}
```

### C++
```cpp
#include <iostream>
#include <string>
#include <vector>

class Settings {
public:
    // Memento: all members private; Settings is a friend, so only it can read them.
    class Snapshot {
        friend class Settings;
        Snapshot(std::string t, int f) : theme_(std::move(t)), fontSize_(f) {}
        std::string theme_;
        int fontSize_;
    };

    Snapshot save() const { return Snapshot(theme_, fontSize_); }
    void restore(const Snapshot& s) { theme_ = s.theme_; fontSize_ = s.fontSize_; }

    void setTheme(std::string t) { theme_ = std::move(t); }
    void setFontSize(int f) { fontSize_ = f; }
    void print() const { std::cout << "theme=" << theme_ << ", fontSize=" << fontSize_ << "\n"; }

private:
    std::string theme_ = "light";
    int fontSize_ = 14;
};

int main() {
    Settings settings;
    std::vector<Settings::Snapshot> history;  // caretaker

    history.push_back(settings.save());
    settings.setTheme("dark");
    history.push_back(settings.save());
    settings.setFontSize(18);
    settings.print();

    settings.restore(history.back());
    history.pop_back();
    settings.print();
}
```

### Rust
```rust
mod settings {
    #[derive(Debug)]
    pub struct Settings {
        pub theme: String,
        pub font_size: u32,
    }

    // Memento: public type, private fields -> opaque outside this module.
    #[derive(Clone)]
    pub struct Snapshot {
        theme: String,
        font_size: u32,
    }

    impl Settings {
        pub fn save(&self) -> Snapshot {
            Snapshot { theme: self.theme.clone(), font_size: self.font_size }
        }

        pub fn restore(&mut self, s: Snapshot) {
            self.theme = s.theme;
            self.font_size = s.font_size;
        }
    }
}

use settings::Settings;

fn main() {
    let mut s = Settings { theme: "light".into(), font_size: 14 };
    let mut history = Vec::new(); // caretaker

    history.push(s.save());
    s.theme = "dark".into();
    history.push(s.save());
    s.font_size = 18;
    println!("{s:?}");

    if let Some(m) = history.pop() {
        s.restore(m);
    }
    println!("{s:?}");
}
```

### Python
```python
from dataclasses import dataclass


class Settings:
    @dataclass(frozen=True)
    class _Snapshot:  # leading underscore: private by convention
        theme: str
        font_size: int

    def __init__(self) -> None:
        self.theme = "light"
        self.font_size = 14

    def save(self) -> "Settings._Snapshot":
        return Settings._Snapshot(self.theme, self.font_size)

    def restore(self, snapshot: "Settings._Snapshot") -> None:
        self.theme = snapshot.theme
        self.font_size = snapshot.font_size

    def __repr__(self) -> str:
        return f"Settings(theme={self.theme!r}, font_size={self.font_size})"


if __name__ == "__main__":
    settings = Settings()
    history: list[Settings._Snapshot] = []  # caretaker

    history.append(settings.save())
    settings.theme = "dark"
    history.append(settings.save())
    settings.font_size = 18
    print(settings)

    settings.restore(history.pop())
    print(settings)
    settings.restore(history.pop())
    print(settings)
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Nested class with private fields; records are a simple (non-opaque) alternative. Serialization for persisted snapshots. |
| Kotlin | `data class copy()` gives cheap snapshots of immutable state; `internal` limits visibility to the module. |
| Go | Unexported fields make mementos opaque across packages. |
| C++ | `friend` gives the originator exclusive access to the memento's internals. |
| Rust | Module privacy (`pub struct` with private fields); ownership makes snapshots explicit (`clone`). |
| Python | Privacy is by convention; `copy.deepcopy` for quick snapshots; `pickle` for persistence (never unpickle untrusted data). |

### Referências / References
- *Design Patterns* (GoF), Memento
- [Refactoring.Guru: Memento](https://refactoring.guru/design-patterns/memento)
