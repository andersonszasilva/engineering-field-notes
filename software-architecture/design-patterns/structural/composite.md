# Composite

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, composite, tree, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Compor objetos em **estruturas de árvore** para representar hierarquias parte-todo, permitindo que o cliente trate **objetos individuais e composições de forma uniforme**.

### Problema
Um sistema de arquivos tem arquivos e diretórios; diretórios contêm arquivos e outros diretórios. Calcular o tamanho total com `if (x instanceof Directory) ... else ...` espalhado pelo código é frágil. O mesmo vale para menus, organogramas, ASTs, componentes de UI e regras de permissão aninhadas.

### Estrutura
- **Component**: interface comum (`size()`, `print()`).
- **Leaf**: elemento sem filhos (`File`).
- **Composite**: contém filhos (`Directory`) e implementa as operações delegando a eles, normalmente de forma recursiva.
- Decisão de design: operações de gerenciamento de filhos (`add`, `remove`) ficam no Component (**transparência**) ou só no Composite (**segurança de tipos**).

### Trade-offs
| Prós | Contras |
|------|---------|
| Cliente trata folha e composto do mesmo jeito. | Interface comum pode ficar genérica demais (folha com `add()` que lança exceção). |
| Recursão natural; fácil adicionar novos tipos de nó. | Difícil restringir quais filhos são válidos em cada composto. |
| Modela bem hierarquias do mundo real. | Árvores profundas: risco de stack overflow e custo em operações recursivas. |
| Combina bem com Visitor e Iterator. | Ciclos acidentais (um nó como filho de si mesmo) geram loops infinitos. |

### Quando usar / evitar
- **Use** para estruturas parte-todo: sistemas de arquivos, UI, menus, expressões, organogramas, BOM (bill of materials).
- **Evite** quando a estrutura não é realmente recursiva ou quando folhas e compostos têm comportamentos muito diferentes.

---

## English

### Intent
Compose objects into **tree structures** to represent part-whole hierarchies, letting clients treat **individual objects and compositions uniformly**.

### Problem
A file system has files and directories; directories contain files and other directories. Computing the total size with `if (x instanceof Directory) ... else ...` scattered around is brittle. The same goes for menus, org charts, ASTs, UI components, and nested permission rules.

### Structure
- **Component**: common interface (`size()`, `print()`).
- **Leaf**: an element with no children (`File`).
- **Composite**: holds children (`Directory`) and implements operations by delegating to them, usually recursively.
- Design decision: child-management operations (`add`, `remove`) live in Component (**transparency**) or only in Composite (**type safety**).

### Trade-offs
| Pros | Cons |
|------|------|
| Clients treat leaves and composites the same way. | The common interface can get too generic (a leaf with an `add()` that throws). |
| Natural recursion; easy to add new node types. | Hard to restrict which children are valid in each composite. |
| Models real-world hierarchies well. | Deep trees: risk of stack overflow and cost of recursive operations. |
| Pairs well with Visitor and Iterator. | Accidental cycles (a node as its own child) cause infinite loops. |

### When to use / avoid
- **Use** for part-whole structures: file systems, UIs, menus, expressions, org charts, BOM (bill of materials).
- **Avoid** when the structure isn't really recursive or when leaves and composites behave very differently.

---

## Exemplos / Examples

Scenario: files and directories; `size()` works the same on both.

<details>
<summary><strong>Java</strong></summary>

```java
import java.util.ArrayList;
import java.util.List;

// Sealed hierarchy (Java 17+): the set of node types is closed and known.
sealed interface Node permits File, Directory {
    String name();
    long size();
}

record File(String name, long size) implements Node {}

record Directory(String name, List<Node> children) implements Node {
    Directory(String name) { this(name, new ArrayList<>()); }

    Directory add(Node child) { children.add(child); return this; }

    @Override public long size() {
        return children.stream().mapToLong(Node::size).sum();
    }
}

class Main {
    public static void main(String[] args) {
        Directory root = new Directory("root")
                .add(new File("README.md", 120))
                .add(new Directory("src")
                        .add(new File("Main.java", 2_000))
                        .add(new File("Util.java", 800)));
        System.out.println(root.name() + " = " + root.size() + " bytes"); // 2920
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
sealed interface Node {
    val name: String
    fun size(): Long
}

data class File(override val name: String, val bytes: Long) : Node {
    override fun size() = bytes
}

class Directory(override val name: String, vararg children: Node) : Node {
    private val children = children.toMutableList()
    fun add(node: Node) = apply { children += node }
    override fun size() = children.sumOf { it.size() }
}

fun main() {
    val root = Directory(
        "root",
        File("README.md", 120),
        Directory("src", File("Main.kt", 2_000), File("Util.kt", 800)),
    )
    println("${root.name} = ${root.size()} bytes") // 2920
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import "fmt"

type Node interface {
	Name() string
	Size() int64
}

type File struct {
	name  string
	bytes int64
}

func (f File) Name() string { return f.name }
func (f File) Size() int64  { return f.bytes }

type Directory struct {
	name     string
	children []Node
}

func (d *Directory) Name() string { return d.name }

func (d *Directory) Add(children ...Node) *Directory {
	d.children = append(d.children, children...)
	return d
}

func (d *Directory) Size() int64 {
	var total int64
	for _, c := range d.children {
		total += c.Size()
	}
	return total
}

func main() {
	src := (&Directory{name: "src"}).Add(File{"main.go", 2000}, File{"util.go", 800})
	root := (&Directory{name: "root"}).Add(File{"README.md", 120}, src)
	fmt.Printf("%s = %d bytes\n", root.Name(), root.Size()) // 2920
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <iostream>
#include <memory>
#include <numeric>
#include <string>
#include <vector>

struct Node {
    virtual ~Node() = default;
    virtual std::string name() const = 0;
    virtual long size() const = 0;
};

class File : public Node {
public:
    File(std::string n, long b) : name_(std::move(n)), bytes_(b) {}
    std::string name() const override { return name_; }
    long size() const override { return bytes_; }
private:
    std::string name_;
    long bytes_;
};

class Directory : public Node {
public:
    explicit Directory(std::string n) : name_(std::move(n)) {}
    Directory& add(std::unique_ptr<Node> child) { children_.push_back(std::move(child)); return *this; }
    std::string name() const override { return name_; }
    long size() const override {
        return std::accumulate(children_.begin(), children_.end(), 0L,
                               [](long acc, const auto& c) { return acc + c->size(); });
    }
private:
    std::string name_;
    std::vector<std::unique_ptr<Node>> children_;  // the composite owns its children
};

int main() {
    auto src = std::make_unique<Directory>("src");
    src->add(std::make_unique<File>("main.cpp", 2000)).add(std::make_unique<File>("util.cpp", 800));

    Directory root("root");
    root.add(std::make_unique<File>("README.md", 120)).add(std::move(src));
    std::cout << root.name() << " = " << root.size() << " bytes\n";  // 2920
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
// An enum is the idiomatic Rust composite: closed set of node kinds, exhaustive matching.
enum Node {
    File { name: String, bytes: u64 },
    Directory { name: String, children: Vec<Node> },
}

impl Node {
    fn file(name: &str, bytes: u64) -> Self {
        Node::File { name: name.into(), bytes }
    }

    fn dir(name: &str, children: Vec<Node>) -> Self {
        Node::Directory { name: name.into(), children }
    }

    fn name(&self) -> &str {
        match self {
            Node::File { name, .. } | Node::Directory { name, .. } => name,
        }
    }

    fn size(&self) -> u64 {
        match self {
            Node::File { bytes, .. } => *bytes,
            Node::Directory { children, .. } => children.iter().map(Node::size).sum(),
        }
    }
}

fn main() {
    let root = Node::dir(
        "root",
        vec![
            Node::file("README.md", 120),
            Node::dir("src", vec![Node::file("main.rs", 2000), Node::file("util.rs", 800)]),
        ],
    );
    println!("{} = {} bytes", root.name(), root.size()); // 2920
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
from __future__ import annotations

from dataclasses import dataclass, field


@dataclass
class File:
    name: str
    bytes: int

    def size(self) -> int:
        return self.bytes


@dataclass
class Directory:
    name: str
    children: list[File | Directory] = field(default_factory=list)

    def add(self, *nodes: File | Directory) -> Directory:
        self.children.extend(nodes)
        return self

    def size(self) -> int:
        return sum(child.size() for child in self.children)


if __name__ == "__main__":
    root = Directory("root").add(
        File("README.md", 120),
        Directory("src").add(File("main.py", 2000), File("util.py", 800)),
    )
    print(f"{root.name} = {root.size()} bytes")  # 2920
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Sealed interfaces + records + pattern matching (`switch`) make composites type-safe. Swing/AWT `Container` is the classic example. |
| Kotlin | `sealed interface` + `when` exhaustiveness; Compose UI is a composite tree. |
| Go | Interface + slice of children; pointer receivers for mutable composites. |
| C++ | `unique_ptr` children express ownership; `shared_ptr` only when nodes are shared (DAGs). |
| Rust | Enums with `Vec<Node>`; recursive types need indirection (`Vec`/`Box`). |
| Python | Duck typing; `__iter__` on the composite enables `for child in directory`. |

### Referências / References
- *Design Patterns* (GoF), Composite
- [Refactoring.Guru: Composite](https://refactoring.guru/design-patterns/composite)
