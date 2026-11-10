# Flyweight

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, flyweight, memory, interning, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Usar **compartilhamento** para suportar grandes quantidades de objetos de granularidade fina de forma eficiente em memória.

### Problema
Um mapa mostra 1 milhão de marcadores. Cada um tem posição própria, mas o ícone (bitmap de alguns KB), a cor e o estilo se repetem entre poucos tipos ("restaurante", "hospital", "posto"). Guardar uma cópia do ícone em cada marcador consome gigabytes.

### Estrutura
- **Estado intrínseco**: compartilhado, imutável, independente de contexto (ícone, cor, nome do tipo). Fica no **Flyweight**.
- **Estado extrínseco**: varia por objeto (latitude, longitude). Fica no objeto de contexto ou é passado como parâmetro.
- **FlyweightFactory**: cache que devolve a instância existente para uma chave ou cria uma nova.

Exemplos reais: `String.intern()`, `Integer.valueOf()` para -128..127, glifos em editores de texto, *string interning* em compiladores, texturas compartilhadas em jogos.

### Trade-offs
| Prós | Contras |
|------|---------|
| Redução drástica de memória com muitos objetos repetidos. | Só compensa com **muitos** objetos e alta repetição; caso contrário é complexidade gratuita. |
| Menos alocações e pressão no GC. | O flyweight **precisa ser imutável**; mutação vaza para todos que o compartilham. |
| Melhor localidade de cache em alguns casos. | A fábrica é estado compartilhado: precisa ser thread-safe e pode crescer sem limite (memory leak). |
| | Separar intrínseco/extrínseco complica o modelo e as assinaturas. |

### Quando usar / evitar
- **Use** depois de **medir**: heap dumps mostrando milhões de objetos com dados duplicados.
- **Evite** como otimização especulativa. Considere também alternativas como *struct of arrays*, enums e IDs inteiros.

---

## English

### Intent
Use **sharing** to support large numbers of fine-grained objects efficiently in memory.

### Problem
A map shows 1 million markers. Each has its own position, but the icon (a bitmap of a few KB), color, and style repeat across a handful of types ("restaurant", "hospital", "gas station"). Storing a copy of the icon in each marker takes gigabytes.

### Structure
- **Intrinsic state**: shared, immutable, context-independent (icon, color, type name). Lives in the **Flyweight**.
- **Extrinsic state**: varies per object (latitude, longitude). Lives in the context object or is passed as a parameter.
- **FlyweightFactory**: a cache that returns the existing instance for a key or creates a new one.

Real examples: `String.intern()`, `Integer.valueOf()` for -128..127, glyphs in text editors, *string interning* in compilers, shared textures in games.

### Trade-offs
| Pros | Cons |
|------|------|
| Drastic memory reduction with many repeated objects. | Only pays off with **many** objects and high repetition; otherwise it's free complexity. |
| Fewer allocations and less GC pressure. | The flyweight **must be immutable**; mutations leak to everyone sharing it. |
| Better cache locality in some cases. | The factory is shared state: it must be thread-safe and may grow unbounded (memory leak). |
| | Splitting intrinsic/extrinsic complicates the model and signatures. |

### When to use / avoid
- **Use** after **measuring**: heap dumps showing millions of objects with duplicated data.
- **Avoid** as a speculative optimization. Also consider alternatives like *struct of arrays*, enums, and integer IDs.

---

## Exemplos / Examples

Scenario: 100,000 map markers share 2 `MarkerStyle` flyweights (heavy icon data).

### Java
```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// Flyweight: intrinsic, immutable, shared.
record MarkerStyle(String type, String color, byte[] icon) {}

// Factory: thread-safe cache of flyweights.
final class MarkerStyles {
    private static final Map<String, MarkerStyle> CACHE = new ConcurrentHashMap<>();

    static MarkerStyle of(String type, String color) {
        return CACHE.computeIfAbsent(type + ":" + color,
                k -> new MarkerStyle(type, color, new byte[4_096])); // pretend: loaded icon
    }

    static int size() { return CACHE.size(); }
}

// Context: extrinsic state + reference to the flyweight.
record Marker(double lat, double lon, MarkerStyle style) {}

class Main {
    public static void main(String[] args) {
        List<Marker> markers = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            String type = i % 2 == 0 ? "restaurant" : "hospital";
            markers.add(new Marker(-23.5 + i * 1e-5, -46.6, MarkerStyles.of(type, "red")));
        }
        System.out.println(markers.size() + " markers, " + MarkerStyles.size() + " styles");
    }
}
```

### Kotlin
```kotlin
import java.util.concurrent.ConcurrentHashMap

class MarkerStyle(val type: String, val color: String, val icon: ByteArray)

object MarkerStyles {
    private val cache = ConcurrentHashMap<Pair<String, String>, MarkerStyle>()
    fun of(type: String, color: String): MarkerStyle =
        cache.computeIfAbsent(type to color) { MarkerStyle(type, color, ByteArray(4_096)) }
    val size get() = cache.size
}

data class Marker(val lat: Double, val lon: Double, val style: MarkerStyle)

fun main() {
    val markers = List(100_000) { i ->
        Marker(-23.5 + i * 1e-5, -46.6, MarkerStyles.of(if (i % 2 == 0) "restaurant" else "hospital", "red"))
    }
    println("${markers.size} markers, ${MarkerStyles.size} styles")
}
```

### Go
```go
package main

import (
	"fmt"
	"sync"
)

// Flyweight
type MarkerStyle struct {
	Type, Color string
	Icon        []byte
}

type styleKey struct{ typ, color string }

// Factory
type StyleFactory struct {
	mu     sync.Mutex
	styles map[styleKey]*MarkerStyle
}

func (f *StyleFactory) Get(typ, color string) *MarkerStyle {
	f.mu.Lock()
	defer f.mu.Unlock()
	k := styleKey{typ, color}
	if s, ok := f.styles[k]; ok {
		return s
	}
	s := &MarkerStyle{Type: typ, Color: color, Icon: make([]byte, 4096)}
	f.styles[k] = s
	return s
}

// Context: extrinsic state + pointer to the shared style.
type Marker struct {
	Lat, Lon float64
	Style    *MarkerStyle
}

func main() {
	f := &StyleFactory{styles: map[styleKey]*MarkerStyle{}}
	markers := make([]Marker, 0, 100_000)
	for i := 0; i < 100_000; i++ {
		typ := "hospital"
		if i%2 == 0 {
			typ = "restaurant"
		}
		markers = append(markers, Marker{Lat: -23.5 + float64(i)*1e-5, Lon: -46.6, Style: f.Get(typ, "red")})
	}
	fmt.Println(len(markers), "markers,", len(f.styles), "styles")
	// Go 1.23+ also ships `unique.Make` for interning comparable values.
}
```

### C++
```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>
#include <utility>
#include <vector>

struct MarkerStyle {  // flyweight: const after construction
    const std::string type, color;
    const std::vector<unsigned char> icon;
};

class StyleFactory {
public:
    std::shared_ptr<const MarkerStyle> get(const std::string& type, const std::string& color) {
        auto key = std::make_pair(type, color);
        auto it = cache_.find(key);
        if (it != cache_.end()) return it->second;
        auto style = std::make_shared<const MarkerStyle>(MarkerStyle{type, color, std::vector<unsigned char>(4096)});
        cache_.emplace(key, style);
        return style;
    }
    std::size_t size() const { return cache_.size(); }
private:
    std::map<std::pair<std::string, std::string>, std::shared_ptr<const MarkerStyle>> cache_;
};

struct Marker {
    double lat, lon;
    std::shared_ptr<const MarkerStyle> style;
};

int main() {
    StyleFactory factory;
    std::vector<Marker> markers;
    markers.reserve(100000);
    for (int i = 0; i < 100000; ++i)
        markers.push_back({-23.5 + i * 1e-5, -46.6, factory.get(i % 2 == 0 ? "restaurant" : "hospital", "red")});
    std::cout << markers.size() << " markers, " << factory.size() << " styles\n";
}
```

### Rust
```rust
use std::collections::HashMap;
use std::rc::Rc;

// Flyweight: immutable data shared through Rc (use Arc across threads).
struct MarkerStyle {
    kind: String,
    color: String,
    icon: Vec<u8>,
}

#[derive(Default)]
struct StyleFactory {
    styles: HashMap<(String, String), Rc<MarkerStyle>>,
}

impl StyleFactory {
    fn get(&mut self, kind: &str, color: &str) -> Rc<MarkerStyle> {
        self.styles
            .entry((kind.to_string(), color.to_string()))
            .or_insert_with(|| Rc::new(MarkerStyle { kind: kind.into(), color: color.into(), icon: vec![0; 4096] }))
            .clone() // cheap: increments a reference count
    }
}

struct Marker {
    lat: f64,
    lon: f64,
    style: Rc<MarkerStyle>,
}

fn main() {
    let mut factory = StyleFactory::default();
    let markers: Vec<Marker> = (0..100_000)
        .map(|i| Marker {
            lat: -23.5 + i as f64 * 1e-5,
            lon: -46.6,
            style: factory.get(if i % 2 == 0 { "restaurant" } else { "hospital" }, "red"),
        })
        .collect();
    let s = &markers[0].style;
    println!("{} markers, {} styles; first: {} {} ({} bytes) at {},{}",
        markers.len(), factory.styles.len(), s.kind, s.color, s.icon.len(), markers[0].lat, markers[0].lon);
}
```

### Python
```python
from dataclasses import dataclass
from functools import cache


@dataclass(frozen=True, slots=True)
class MarkerStyle:
    type: str
    color: str
    icon: bytes


# The cached factory returns the same instance for the same arguments.
@cache
def marker_style(type_: str, color: str) -> MarkerStyle:
    return MarkerStyle(type_, color, bytes(4096))


@dataclass(slots=True)  # __slots__ also saves memory per instance
class Marker:
    lat: float
    lon: float
    style: MarkerStyle


if __name__ == "__main__":
    markers = [
        Marker(-23.5 + i * 1e-5, -46.6, marker_style("restaurant" if i % 2 == 0 else "hospital", "red"))
        for i in range(100_000)
    ]
    print(f"{len(markers)} markers, {marker_style.cache_info().currsize} styles")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `Integer.valueOf`, `String.intern()`, enums; JVM flags like `-XX:+UseStringDeduplication` (G1) do it automatically for strings. |
| Kotlin | Same JVM tools; `object`s and enum entries are natural flyweights. |
| Go | `unique.Make` (Go 1.23+) canonicalizes comparable values; map + mutex for custom factories. |
| C++ | `shared_ptr<const T>` for shared immutable state; or indices into a table for zero overhead. |
| Rust | `Rc<T>`/`Arc<T>` for shared immutable data; interning crates; indices into a `Vec` avoid refcounts. |
| Python | `sys.intern`, small int cache, `@cache` factories, `__slots__` to shrink instances. |

### Referências / References
- *Design Patterns* (GoF), Flyweight
- [Refactoring.Guru: Flyweight](https://refactoring.guru/design-patterns/flyweight)
- [Go `unique` package](https://pkg.go.dev/unique)
