# Iterator

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, iterator, generators, lazy, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Fornecer uma forma de **acessar sequencialmente os elementos** de uma coleção **sem expor sua representação interna**.

### Problema
Uma API paginada devolve registros em páginas de 100. O código cliente quer apenas "percorrer todos os pedidos", sem lidar com cursores, tokens de página e chamadas HTTP. O mesmo vale para árvores, grafos ou arquivos enormes: o cliente quer um `for`, não os detalhes de navegação.

### Estrutura
- **Iterator**: interface com `hasNext()`/`next()` (ou um `next()` que devolve `Option`/`None` no fim).
- **ConcreteIterator**: guarda a posição atual da travessia.
- **Aggregate / Iterable**: cria iteradores (`iterator()`), permitindo várias travessias independentes ao mesmo tempo.

Hoje o padrão é **parte da linguagem** em quase todas: `for-each`, generators (`yield`), `Iterator` trait, `range`-over-func, streams/sequences lazy.

### Trade-offs
| Prós | Contras |
|------|---------|
| Esconde a estrutura interna (array, árvore, API paginada). | Iteradores externos com estado podem ser invalidados se a coleção mudar (`ConcurrentModificationException`). |
| Avaliação **lazy**: processa sequências enormes ou infinitas com memória constante. | Iteradores lazy adiam erros e efeitos colaterais para o momento do consumo. |
| Várias travessias simultâneas e diferentes ordens (DFS, BFS). | Iteradores single-pass (streams, generators) não podem ser reiniciados. |
| Combina com operações funcionais (`map`, `filter`, `take`). | Pode ser mais lento que um loop direto sobre um array em código crítico. |

### Quando usar / evitar
- **Use** para esconder paginação, percorrer estruturas customizadas, processar dados em streaming e expor sequências lazy.
- **Evite** criar iteradores manuais quando a linguagem já oferece generators ou quando a coleção padrão já é iterável.

---

## English

### Intent
Provide a way to **access the elements** of a collection **sequentially without exposing its underlying representation**.

### Problem
A paginated API returns records in pages of 100. Client code just wants to "go through all orders" without dealing with cursors, page tokens, and HTTP calls. The same applies to trees, graphs, or huge files: the client wants a `for` loop, not the navigation details.

### Structure
- **Iterator**: an interface with `hasNext()`/`next()` (or a `next()` returning `Option`/`None` at the end).
- **ConcreteIterator**: keeps the current traversal position.
- **Aggregate / Iterable**: creates iterators (`iterator()`), allowing several independent traversals at once.

Today the pattern is **part of the language** in almost all of them: `for-each`, generators (`yield`), the `Iterator` trait, range-over-func, lazy streams/sequences.

### Trade-offs
| Pros | Cons |
|------|------|
| Hides the internal structure (array, tree, paginated API). | Stateful external iterators can be invalidated if the collection changes (`ConcurrentModificationException`). |
| **Lazy** evaluation: processes huge or infinite sequences in constant memory. | Lazy iterators defer errors and side effects until consumption. |
| Multiple simultaneous traversals and different orders (DFS, BFS). | Single-pass iterators (streams, generators) can't be restarted. |
| Composes with functional operations (`map`, `filter`, `take`). | Can be slower than a direct array loop in hot code. |

### When to use / avoid
- **Use** to hide pagination, traverse custom structures, process streaming data, and expose lazy sequences.
- **Avoid** writing manual iterators when the language already has generators or when a standard collection is already iterable.

---

## Exemplos / Examples

Scenario: iterate over all orders from a paginated API (simulated), fetching pages lazily.

### Java
```java
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;

// Simulated paginated API: 7 orders, page size 3.
class OrdersApi {
    record Page(List<String> items, Integer nextPage) {}

    Page fetch(int page) {
        System.out.println("  (fetching page " + page + ")");
        int start = page * 3, end = Math.min(start + 3, 7);
        List<String> items = java.util.stream.IntStream.range(start, end).mapToObj(i -> "order-" + i).toList();
        return new Page(items, end < 7 ? page + 1 : null);
    }
}

class AllOrders implements Iterable<String> {
    private final OrdersApi api;
    AllOrders(OrdersApi api) { this.api = api; }

    @Override public Iterator<String> iterator() {
        return new Iterator<>() {
            private Iterator<String> current = List.<String>of().iterator();
            private Integer nextPage = 0;

            @Override public boolean hasNext() {
                while (!current.hasNext() && nextPage != null) {
                    OrdersApi.Page page = api.fetch(nextPage);
                    current = page.items().iterator();
                    nextPage = page.nextPage();
                }
                return current.hasNext();
            }

            @Override public String next() {
                if (!hasNext()) throw new NoSuchElementException();
                return current.next();
            }
        };
    }
}

class Main {
    public static void main(String[] args) {
        for (String order : new AllOrders(new OrdersApi())) {
            System.out.println(order);
        }
    }
}
```

### Kotlin
```kotlin
data class Page(val items: List<String>, val nextPage: Int?)

fun fetch(page: Int): Page {
    println("  (fetching page $page)")
    val start = page * 3
    val end = minOf(start + 3, 7)
    return Page((start until end).map { "order-$it" }, if (end < 7) page + 1 else null)
}

// `sequence { yield }` builds a lazy iterator with a coroutine-based generator.
fun allOrders(): Sequence<String> = sequence {
    var page: Int? = 0
    while (page != null) {
        val p = fetch(page)
        yieldAll(p.items)
        page = p.nextPage
    }
}

fun main() {
    allOrders().forEach(::println)
    println(allOrders().first()) // lazy: fetches only page 0
}
```

### Go
```go
package main

import (
	"fmt"
	"iter"
)

type Page struct {
	Items    []string
	NextPage int // -1 when there are no more pages
}

func fetch(page int) Page {
	fmt.Printf("  (fetching page %d)\n", page)
	start, end := page*3, min(page*3+3, 7)
	items := make([]string, 0, end-start)
	for i := start; i < end; i++ {
		items = append(items, fmt.Sprintf("order-%d", i))
	}
	next := -1
	if end < 7 {
		next = page + 1
	}
	return Page{Items: items, NextPage: next}
}

// Go 1.23+ range-over-func iterator: iteration stops when yield returns false.
func AllOrders() iter.Seq[string] {
	return func(yield func(string) bool) {
		for page := 0; page != -1; {
			p := fetch(page)
			for _, item := range p.Items {
				if !yield(item) {
					return
				}
			}
			page = p.NextPage
		}
	}
}

func main() {
	for order := range AllOrders() {
		fmt.Println(order)
	}
}
```

### C++
```cpp
#include <algorithm>
#include <iostream>
#include <optional>
#include <string>
#include <vector>

struct Page { std::vector<std::string> items; std::optional<int> nextPage; };

Page fetch(int page) {
    std::cout << "  (fetching page " << page << ")\n";
    Page p;
    int start = page * 3, end = std::min(start + 3, 7);
    for (int i = start; i < end; ++i) p.items.push_back("order-" + std::to_string(i));
    if (end < 7) p.nextPage = page + 1;
    return p;
}

// A minimal input iterator + range so the class works with range-based for.
class AllOrders {
public:
    class iterator {
    public:
        explicit iterator(bool end) : end_(end) { if (!end_) advance(); }
        const std::string& operator*() const { return page_.items[idx_]; }
        iterator& operator++() { ++idx_; advance(); return *this; }
        bool operator!=(const iterator& o) const { return end_ != o.end_; }
    private:
        void advance() {
            while (idx_ >= page_.items.size()) {
                if (!next_) { end_ = true; return; }
                page_ = fetch(*next_);
                next_ = page_.nextPage;
                idx_ = 0;
            }
        }
        Page page_{};
        std::optional<int> next_{0};
        std::size_t idx_ = 0;
        bool end_;
    };
    iterator begin() const { return iterator(false); }
    iterator end() const { return iterator(true); }
};

int main() {
    for (const auto& order : AllOrders{}) std::cout << order << "\n";
    // C++23 offers std::generator<T> with co_yield for a much shorter version.
}
```

### Rust
```rust
struct Page {
    items: Vec<String>,
    next_page: Option<u32>,
}

fn fetch(page: u32) -> Page {
    println!("  (fetching page {page})");
    let (start, end) = (page * 3, (page * 3 + 3).min(7));
    Page { items: (start..end).map(|i| format!("order-{i}")).collect(), next_page: (end < 7).then_some(page + 1) }
}

struct AllOrders {
    buffer: std::vec::IntoIter<String>,
    next_page: Option<u32>,
}

impl AllOrders {
    fn new() -> Self {
        Self { buffer: Vec::new().into_iter(), next_page: Some(0) }
    }
}

// Implementing `Iterator` gives you map/filter/take/collect for free.
impl Iterator for AllOrders {
    type Item = String;

    fn next(&mut self) -> Option<String> {
        loop {
            if let Some(item) = self.buffer.next() {
                return Some(item);
            }
            let page = fetch(self.next_page?); // `?` returns None when there are no more pages
            self.buffer = page.items.into_iter();
            self.next_page = page.next_page;
        }
    }
}

fn main() {
    for order in AllOrders::new() {
        println!("{order}");
    }
    let first_two: Vec<_> = AllOrders::new().take(2).collect(); // fetches only page 0
    println!("{first_two:?}");
}
```

### Python
```python
from collections.abc import Iterator
from dataclasses import dataclass


@dataclass
class Page:
    items: list[str]
    next_page: int | None


def fetch(page: int) -> Page:
    print(f"  (fetching page {page})")
    start, end = page * 3, min(page * 3 + 3, 7)
    return Page([f"order-{i}" for i in range(start, end)], page + 1 if end < 7 else None)


# A generator function is the idiomatic iterator: state lives in the paused frame.
def all_orders() -> Iterator[str]:
    page: int | None = 0
    while page is not None:
        p = fetch(page)
        yield from p.items
        page = p.next_page


if __name__ == "__main__":
    for order in all_orders():
        print(order)
    print(next(all_orders()))  # lazy: fetches only page 0
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `Iterable`/`Iterator` for `for-each`; `Stream` for lazy pipelines; `Spliterator` for parallel iteration. |
| Kotlin | `sequence { yield(...) }` generators; `Sequence` for lazy, `Flow` for async streams. |
| Go | Range-over-func iterators (`iter.Seq`, Go 1.23+); channels are also used but cost goroutines. |
| C++ | Iterator concepts and ranges (C++20 `std::ranges`); `std::generator` (C++23). |
| Rust | The `Iterator` trait with dozens of zero-cost adapters; `IntoIterator` for `for` loops. |
| Python | Generators (`yield`), `__iter__`/`__next__`, `itertools`. |

### Referências / References
- *Design Patterns* (GoF), Iterator
- [Go: Range Over Function Types](https://go.dev/blog/range-functions)
- [Refactoring.Guru: Iterator](https://refactoring.guru/design-patterns/iterator)
