# Builder

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / creational
**Tags:** gof, design-patterns, creational, builder, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Separar a **construção** de um objeto complexo da sua **representação**, permitindo montá-lo passo a passo.

### Problema
Objetos com muitos parâmetros, vários deles opcionais, levam ao *telescoping constructor* (`new Request(url, "GET", null, null, 30, true, ...)`): ilegível e fácil de errar a ordem. Setters resolvem a legibilidade, mas deixam o objeto mutável e possivelmente inválido no meio do caminho.

### Estrutura
- **Builder**: acumula os parâmetros com métodos encadeáveis e valida tudo em `build()`.
- **Product**: objeto final, normalmente **imutável**.
- **Director** (opcional no GoF): encapsula sequências de construção reutilizáveis ("requisição JSON autenticada padrão").

### Trade-offs
| Prós | Contras |
|------|---------|
| Chamadas legíveis e autoexplicativas. | Duplica os campos do produto no builder (boilerplate). |
| Produto imutável e validado de uma vez em `build()`. | Erros de campo obrigatório ausente só aparecem em runtime (salvo *typestate*). |
| Parâmetros opcionais com defaults sem explosão de construtores. | Exagero para objetos com 2 ou 3 campos. |
| Pode reutilizar o builder para gerar variações. | Builder mutável compartilhado entre threads é fonte de bugs. |

### Quando usar / evitar
- **Use** para objetos com muitos campos opcionais, configurações, clientes HTTP, queries e mensagens de teste (*test data builders*).
- **Evite** em linguagens com **argumentos nomeados e defaults** (Kotlin, Python): eles resolvem o mesmo problema com menos código.

---

## English

### Intent
Separate the **construction** of a complex object from its **representation**, allowing it to be assembled step by step.

### Problem
Objects with many parameters, several of them optional, lead to the *telescoping constructor* (`new Request(url, "GET", null, null, 30, true, ...)`): unreadable and easy to get the order wrong. Setters fix readability but leave the object mutable and possibly invalid halfway through.

### Structure
- **Builder**: accumulates parameters through chainable methods and validates everything in `build()`.
- **Product**: the final object, usually **immutable**.
- **Director** (optional in GoF): encapsulates reusable construction sequences ("standard authenticated JSON request").

### Trade-offs
| Pros | Cons |
|------|------|
| Readable, self-explanatory calls. | Duplicates the product's fields in the builder (boilerplate). |
| Immutable product validated at once in `build()`. | Missing required fields only fail at runtime (unless you use *typestate*). |
| Optional parameters with defaults without a constructor explosion. | Overkill for objects with 2 or 3 fields. |
| The builder can be reused to produce variations. | A mutable builder shared across threads is a source of bugs. |

### When to use / avoid
- **Use** for objects with many optional fields, configurations, HTTP clients, queries, and test messages (*test data builders*).
- **Avoid** in languages with **named arguments and defaults** (Kotlin, Python): they solve the same problem with less code.

---

## Exemplos / Examples

Scenario: building an immutable `HttpRequest` with a required URL and optional method, headers, and timeout.

<details>
<summary><strong>Java</strong></summary>

```java
import java.time.Duration;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;

    private HttpRequest(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.headers = Map.copyOf(b.headers);
        this.timeout = b.timeout;
    }

    static Builder builder(String url) { return new Builder(url); }

    @Override public String toString() {
        return method + " " + url + " headers=" + headers + " timeout=" + timeout;
    }

    static final class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(30);

        private Builder(String url) { this.url = Objects.requireNonNull(url); }

        Builder method(String method) { this.method = method; return this; }
        Builder header(String k, String v) { headers.put(k, v); return this; }
        Builder timeout(Duration t) { this.timeout = t; return this; }

        HttpRequest build() {
            if (timeout.isNegative()) throw new IllegalStateException("timeout must be >= 0");
            return new HttpRequest(this);
        }
    }
}

class Main {
    public static void main(String[] args) {
        HttpRequest req = HttpRequest.builder("https://api.example.com/orders")
                .method("POST")
                .header("Content-Type", "application/json")
                .timeout(Duration.ofSeconds(5))
                .build();
        System.out.println(req);
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
import kotlin.time.Duration
import kotlin.time.Duration.Companion.seconds

// Idiomatic Kotlin: named arguments + defaults replace most builders.
data class HttpRequest(
    val url: String,
    val method: String = "GET",
    val headers: Map<String, String> = emptyMap(),
    val timeout: Duration = 30.seconds,
) {
    init { require(!timeout.isNegative()) { "timeout must be >= 0" } }
}

// A DSL builder is still useful for nested/structured data.
class HttpRequestBuilder(private val url: String) {
    var method = "GET"
    var timeout = 30.seconds
    private val headers = mutableMapOf<String, String>()
    fun header(key: String, value: String) { headers[key] = value }
    fun build() = HttpRequest(url, method, headers.toMap(), timeout)
}

fun httpRequest(url: String, block: HttpRequestBuilder.() -> Unit): HttpRequest =
    HttpRequestBuilder(url).apply(block).build()

fun main() {
    val a = HttpRequest("https://api.example.com/orders", method = "POST", timeout = 5.seconds)
    val b = httpRequest("https://api.example.com/orders") {
        method = "POST"
        header("Content-Type", "application/json")
    }
    println(a)
    println(b)
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

type HttpRequest struct {
	URL     string
	Method  string
	Headers map[string]string
	Timeout time.Duration
}

// Idiomatic Go: functional options instead of a builder object.
type Option func(*HttpRequest)

func WithMethod(m string) Option         { return func(r *HttpRequest) { r.Method = m } }
func WithHeader(k, v string) Option      { return func(r *HttpRequest) { r.Headers[k] = v } }
func WithTimeout(t time.Duration) Option { return func(r *HttpRequest) { r.Timeout = t } }

func NewHttpRequest(url string, opts ...Option) (*HttpRequest, error) {
	r := &HttpRequest{URL: url, Method: "GET", Headers: map[string]string{}, Timeout: 30 * time.Second}
	for _, opt := range opts {
		opt(r)
	}
	if r.Timeout < 0 {
		return nil, errors.New("timeout must be >= 0")
	}
	return r, nil
}

func main() {
	req, err := NewHttpRequest("https://api.example.com/orders",
		WithMethod("POST"),
		WithHeader("Content-Type", "application/json"),
		WithTimeout(5*time.Second),
	)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", *req)
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <stdexcept>
#include <string>

class HttpRequest {
public:
    class Builder;
    void print() const {
        std::cout << method_ << " " << url_ << " timeout=" << timeout_.count() << "s headers=" << headers_.size() << "\n";
    }
private:
    HttpRequest() = default;
    std::string url_, method_;
    std::map<std::string, std::string> headers_;
    std::chrono::seconds timeout_{};
};

class HttpRequest::Builder {
public:
    explicit Builder(std::string url) : url_(std::move(url)) {}
    Builder& method(std::string m) { method_ = std::move(m); return *this; }
    Builder& header(std::string k, std::string v) { headers_[std::move(k)] = std::move(v); return *this; }
    Builder& timeout(std::chrono::seconds t) { timeout_ = t; return *this; }

    HttpRequest build() const {
        if (timeout_.count() < 0) throw std::invalid_argument("timeout must be >= 0");
        HttpRequest r;
        r.url_ = url_;
        r.method_ = method_;
        r.headers_ = headers_;
        r.timeout_ = timeout_;
        return r;
    }
private:
    std::string url_;
    std::string method_ = "GET";
    std::map<std::string, std::string> headers_;
    std::chrono::seconds timeout_{30};
};

int main() {
    using namespace std::chrono_literals;
    auto req = HttpRequest::Builder("https://api.example.com/orders")
                   .method("POST")
                   .header("Content-Type", "application/json")
                   .timeout(5s)
                   .build();
    req.print();
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
use std::collections::HashMap;
use std::time::Duration;

#[allow(dead_code)] // fields are only read through Debug in this demo
#[derive(Debug)]
struct HttpRequest {
    url: String,
    method: String,
    headers: HashMap<String, String>,
    timeout: Duration,
}

// Consuming builder (takes `self` by value): chaining moves ownership, no clones.
struct HttpRequestBuilder {
    url: String,
    method: String,
    headers: HashMap<String, String>,
    timeout: Duration,
}

impl HttpRequestBuilder {
    fn new(url: impl Into<String>) -> Self {
        Self { url: url.into(), method: "GET".into(), headers: HashMap::new(), timeout: Duration::from_secs(30) }
    }
    fn method(mut self, m: &str) -> Self { self.method = m.into(); self }
    fn header(mut self, k: &str, v: &str) -> Self { self.headers.insert(k.into(), v.into()); self }
    fn timeout(mut self, t: Duration) -> Self { self.timeout = t; self }

    fn build(self) -> Result<HttpRequest, String> {
        if self.url.is_empty() {
            return Err("url is required".into());
        }
        Ok(HttpRequest { url: self.url, method: self.method, headers: self.headers, timeout: self.timeout })
    }
}

fn main() {
    let req = HttpRequestBuilder::new("https://api.example.com/orders")
        .method("POST")
        .header("Content-Type", "application/json")
        .timeout(Duration::from_secs(5))
        .build()
        .expect("valid request");
    println!("{req:?}");
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
from dataclasses import dataclass, field


# Idiomatic Python: keyword arguments with defaults + a frozen dataclass.
@dataclass(frozen=True)
class HttpRequest:
    url: str
    method: str = "GET"
    headers: dict[str, str] = field(default_factory=dict)
    timeout: float = 30.0

    def __post_init__(self) -> None:
        if self.timeout < 0:
            raise ValueError("timeout must be >= 0")


# An explicit fluent builder, useful when construction is incremental.
class HttpRequestBuilder:
    def __init__(self, url: str) -> None:
        self._url = url
        self._method = "GET"
        self._headers: dict[str, str] = {}
        self._timeout = 30.0

    def method(self, method: str) -> "HttpRequestBuilder":
        self._method = method
        return self

    def header(self, key: str, value: str) -> "HttpRequestBuilder":
        self._headers[key] = value
        return self

    def timeout(self, seconds: float) -> "HttpRequestBuilder":
        self._timeout = seconds
        return self

    def build(self) -> HttpRequest:
        return HttpRequest(self._url, self._method, dict(self._headers), self._timeout)


if __name__ == "__main__":
    print(HttpRequest("https://api.example.com/orders", method="POST", timeout=5))
    print(
        HttpRequestBuilder("https://api.example.com/orders")
        .method("POST")
        .header("Content-Type", "application/json")
        .build()
    )
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Hand-written or Lombok `@Builder`; records + builder for immutable DTOs. `HttpRequest.newBuilder()` in the JDK. |
| Kotlin | Named/default arguments and `copy()` cover most cases; type-safe DSL builders (`apply`, lambdas with receiver) for nested data. |
| Go | Functional options (`...Option`) or a plain config struct. |
| C++ | Fluent builder returning `Builder&`; designated initializers (C++20) for simple aggregates. |
| Rust | Consuming builders; the *typestate* pattern can make missing required fields a compile error. |
| Python | Keyword arguments and `dataclass`; `dataclasses.replace()` for variations. |

### Referências / References
- *Design Patterns* (GoF), Builder
- *Effective Java*, Joshua Bloch, Item 2
- Dave Cheney, [Functional options for friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)
- [Refactoring.Guru: Builder](https://refactoring.guru/design-patterns/builder)
