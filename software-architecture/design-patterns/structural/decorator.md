# Decorator

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, decorator, wrapper, middleware, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
**Adicionar responsabilidades a um objeto dinamicamente**, envolvendo-o em outro objeto com a mesma interface. Uma alternativa flexível à herança para estender comportamento.

### Problema
Um cliente que busca dados em uma API precisa, dependendo do contexto, de logging, cache, retry, métricas. Criar subclasses para cada combinação (`CachingLoggingRetryingFetcher`) não escala, e mexer na classe original viola o SRP.

### Estrutura
- **Component**: interface comum (`Fetcher`).
- **ConcreteComponent**: implementação base (`HttpFetcher`).
- **Decorator**: implementa a mesma interface, guarda uma referência ao componente embrulhado e adiciona comportamento antes/depois de delegar.
- Decorators podem ser **empilhados** em qualquer ordem: `Logging(Caching(Http))`.

É o mesmo conceito de *middleware* HTTP, interceptors, `@decorators` do Python e proxies de AOP.

### Trade-offs
| Prós | Contras |
|------|---------|
| Combina comportamentos em runtime sem explosão de subclasses. | **A ordem importa**: `Cache(Retry(x))` ≠ `Retry(Cache(x))`, e isso não é óbvio. |
| Cada decorator tem uma responsabilidade (SRP). | Muitos objetos pequenos; stack traces e debugging ficam mais longos. |
| Open/Closed: adiciona comportamento sem tocar no original. | Interfaces grandes obrigam o decorator a repassar dezenas de métodos. |
| Reutilizável em qualquer implementação da interface. | Identidade: o objeto decorado não é `==` ao original. |

### Quando usar / evitar
- **Use** para preocupações transversais: logging, cache, retry, métricas, autorização, compressão, criptografia de streams.
- **Evite** quando a interface é grande e instável, ou quando um único ponto de extensão (hook, config) resolve.

---

## English

### Intent
**Attach additional responsibilities to an object dynamically** by wrapping it in another object with the same interface. A flexible alternative to subclassing for extending behavior.

### Problem
A client that fetches data from an API needs, depending on context, logging, caching, retries, metrics. Creating subclasses for each combination (`CachingLoggingRetryingFetcher`) doesn't scale, and changing the original class violates SRP.

### Structure
- **Component**: the common interface (`Fetcher`).
- **ConcreteComponent**: the base implementation (`HttpFetcher`).
- **Decorator**: implements the same interface, holds a reference to the wrapped component, and adds behavior before/after delegating.
- Decorators can be **stacked** in any order: `Logging(Caching(Http))`.

It's the same idea as HTTP *middleware*, interceptors, Python `@decorators`, and AOP proxies.

### Trade-offs
| Pros | Cons |
|------|------|
| Combines behaviors at runtime without a subclass explosion. | **Order matters**: `Cache(Retry(x))` ≠ `Retry(Cache(x))`, and it isn't obvious. |
| Each decorator has one responsibility (SRP). | Many small objects; longer stack traces and harder debugging. |
| Open/Closed: adds behavior without touching the original. | Large interfaces force the decorator to forward dozens of methods. |
| Reusable across any implementation of the interface. | Identity: the decorated object isn't `==` to the original. |

### When to use / avoid
- **Use** for cross-cutting concerns: logging, caching, retry, metrics, authorization, compression, stream encryption.
- **Avoid** when the interface is large and unstable, or when a single extension point (hook, config) is enough.

---

## Exemplos / Examples

Scenario: a `Fetcher` decorated with caching and logging.

### Java
```java
import java.util.HashMap;
import java.util.Map;

interface Fetcher {
    String fetch(String url);
}

class HttpFetcher implements Fetcher {
    public String fetch(String url) {
        System.out.println("  (network call to " + url + ")");
        return "data from " + url;
    }
}

class CachingFetcher implements Fetcher {
    private final Fetcher inner;
    private final Map<String, String> cache = new HashMap<>();
    CachingFetcher(Fetcher inner) { this.inner = inner; }

    public String fetch(String url) {
        return cache.computeIfAbsent(url, inner::fetch);
    }
}

class LoggingFetcher implements Fetcher {
    private final Fetcher inner;
    LoggingFetcher(Fetcher inner) { this.inner = inner; }

    public String fetch(String url) {
        long start = System.nanoTime();
        String result = inner.fetch(url);
        System.out.printf("GET %s took %d us%n", url, (System.nanoTime() - start) / 1_000);
        return result;
    }
}

class Main {
    public static void main(String[] args) {
        Fetcher fetcher = new LoggingFetcher(new CachingFetcher(new HttpFetcher()));
        fetcher.fetch("https://api.example.com/users/1"); // network call
        fetcher.fetch("https://api.example.com/users/1"); // served from cache
    }
}
```

### Kotlin
```kotlin
interface Fetcher {
    fun fetch(url: String): String
}

class HttpFetcher : Fetcher {
    override fun fetch(url: String): String {
        println("  (network call to $url)")
        return "data from $url"
    }
}

class CachingFetcher(private val inner: Fetcher) : Fetcher {
    private val cache = mutableMapOf<String, String>()
    override fun fetch(url: String) = cache.getOrPut(url) { inner.fetch(url) }
}

// Class delegation (`by inner`) forwards every other method automatically,
// which removes the main boilerplate cost of decorators on large interfaces.
class LoggingFetcher(private val inner: Fetcher) : Fetcher by inner {
    override fun fetch(url: String): String {
        val start = System.nanoTime()
        return inner.fetch(url).also {
            println("GET $url took ${(System.nanoTime() - start) / 1_000} us")
        }
    }
}

fun main() {
    val fetcher: Fetcher = LoggingFetcher(CachingFetcher(HttpFetcher()))
    fetcher.fetch("https://api.example.com/users/1")
    fetcher.fetch("https://api.example.com/users/1")
}
```

### Go
```go
package main

import (
	"fmt"
	"time"
)

type Fetcher interface {
	Fetch(url string) string
}

// FetcherFunc lets plain functions satisfy the interface (like http.HandlerFunc).
type FetcherFunc func(string) string

func (f FetcherFunc) Fetch(url string) string { return f(url) }

func HttpFetcher() Fetcher {
	return FetcherFunc(func(url string) string {
		fmt.Println("  (network call to", url+")")
		return "data from " + url
	})
}

// Decorators as middleware: func(Fetcher) Fetcher.
func WithCache(next Fetcher) Fetcher {
	cache := map[string]string{} // not goroutine-safe; add a sync.Mutex in real code
	return FetcherFunc(func(url string) string {
		if v, ok := cache[url]; ok {
			return v
		}
		v := next.Fetch(url)
		cache[url] = v
		return v
	})
}

func WithLogging(next Fetcher) Fetcher {
	return FetcherFunc(func(url string) string {
		start := time.Now()
		defer func() { fmt.Printf("GET %s took %s\n", url, time.Since(start)) }()
		return next.Fetch(url)
	})
}

func main() {
	f := WithLogging(WithCache(HttpFetcher()))
	f.Fetch("https://api.example.com/users/1")
	f.Fetch("https://api.example.com/users/1")
}
```

### C++
```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <string>
#include <unordered_map>

struct Fetcher {
    virtual ~Fetcher() = default;
    virtual std::string fetch(const std::string& url) = 0;
};

struct HttpFetcher : Fetcher {
    std::string fetch(const std::string& url) override {
        std::cout << "  (network call to " << url << ")\n";
        return "data from " + url;
    }
};

class CachingFetcher : public Fetcher {
public:
    explicit CachingFetcher(std::unique_ptr<Fetcher> inner) : inner_(std::move(inner)) {}
    std::string fetch(const std::string& url) override {
        if (auto it = cache_.find(url); it != cache_.end()) return it->second;
        return cache_[url] = inner_->fetch(url);
    }
private:
    std::unique_ptr<Fetcher> inner_;
    std::unordered_map<std::string, std::string> cache_;
};

class LoggingFetcher : public Fetcher {
public:
    explicit LoggingFetcher(std::unique_ptr<Fetcher> inner) : inner_(std::move(inner)) {}
    std::string fetch(const std::string& url) override {
        auto start = std::chrono::steady_clock::now();
        auto result = inner_->fetch(url);
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(std::chrono::steady_clock::now() - start);
        std::cout << "GET " << url << " took " << us.count() << " us\n";
        return result;
    }
private:
    std::unique_ptr<Fetcher> inner_;
};

int main() {
    std::unique_ptr<Fetcher> f =
        std::make_unique<LoggingFetcher>(std::make_unique<CachingFetcher>(std::make_unique<HttpFetcher>()));
    f->fetch("https://api.example.com/users/1");
    f->fetch("https://api.example.com/users/1");
}
```

### Rust
```rust
use std::cell::RefCell;
use std::collections::HashMap;
use std::time::Instant;

trait Fetcher {
    fn fetch(&self, url: &str) -> String;
}

struct HttpFetcher;
impl Fetcher for HttpFetcher {
    fn fetch(&self, url: &str) -> String {
        println!("  (network call to {url})");
        format!("data from {url}")
    }
}

// Generic decorators: zero-cost wrapping, resolved at compile time (like tower's `Layer`s).
struct Caching<F: Fetcher> {
    inner: F,
    cache: RefCell<HashMap<String, String>>, // interior mutability behind &self
}

impl<F: Fetcher> Fetcher for Caching<F> {
    fn fetch(&self, url: &str) -> String {
        if let Some(v) = self.cache.borrow().get(url) {
            return v.clone();
        }
        let v = self.inner.fetch(url);
        self.cache.borrow_mut().insert(url.to_string(), v.clone());
        v
    }
}

struct Logging<F: Fetcher> {
    inner: F,
}

impl<F: Fetcher> Fetcher for Logging<F> {
    fn fetch(&self, url: &str) -> String {
        let start = Instant::now();
        let result = self.inner.fetch(url);
        println!("GET {url} took {:?}", start.elapsed());
        result
    }
}

fn main() {
    let fetcher = Logging { inner: Caching { inner: HttpFetcher, cache: RefCell::new(HashMap::new()) } };
    fetcher.fetch("https://api.example.com/users/1");
    fetcher.fetch("https://api.example.com/users/1");
}
```

### Python
```python
import functools
import time
from collections.abc import Callable


# Python's @decorator syntax is the function-level version of this pattern.
def logged(func: Callable[[str], str]) -> Callable[[str], str]:
    @functools.wraps(func)
    def wrapper(url: str) -> str:
        start = time.perf_counter()
        try:
            return func(url)
        finally:
            print(f"GET {url} took {(time.perf_counter() - start) * 1e6:.0f} us")

    return wrapper


@logged
@functools.cache  # applied first: logging wraps caching
def fetch(url: str) -> str:
    print(f"  (network call to {url})")
    return f"data from {url}"


# Object-level decorator, when you need to wrap an instance rather than a function.
class CachingFetcher:
    def __init__(self, inner) -> None:
        self._inner = inner
        self._cache: dict[str, str] = {}

    def fetch(self, url: str) -> str:
        if url not in self._cache:
            self._cache[url] = self._inner.fetch(url)
        return self._cache[url]


if __name__ == "__main__":
    fetch("https://api.example.com/users/1")
    fetch("https://api.example.com/users/1")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `java.io` streams (`BufferedInputStream(new FileInputStream(...))`); `Collections.unmodifiableList`. Spring AOP generates decorators via proxies. |
| Kotlin | Class delegation `: Interface by inner` removes the forwarding boilerplate. |
| Go | `func(Handler) Handler` middleware chains in `net/http`; function types implementing interfaces. |
| C++ | Wrapper classes owning the inner object via `unique_ptr`; templates for static decoration. |
| Rust | Generic wrapper structs (tower `Service`/`Layer`, `BufReader<R>`); zero runtime cost. |
| Python | `@decorator` syntax for functions; `__getattr__` forwarding for object decorators. |

### Referências / References
- *Design Patterns* (GoF), Decorator
- [tower: Service and Layer](https://docs.rs/tower/latest/tower/)
- [Refactoring.Guru: Decorator](https://refactoring.guru/design-patterns/decorator)
