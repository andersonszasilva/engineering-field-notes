# Chain of Responsibility

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, chain-of-responsibility, middleware, pipeline, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Evitar acoplar o remetente de uma requisição ao seu receptor, dando a **vários objetos a chance de tratá-la**. Os handlers são encadeados e a requisição passa pela cadeia até que alguém a trate (ou a interrompa).

### Problema
Uma requisição HTTP precisa passar por autenticação, rate limiting, validação e só então chegar ao handler de negócio. Colocar tudo em um método gigante com `if`s mistura responsabilidades e impede reordenar ou reutilizar etapas.

### Estrutura
- **Handler**: interface com `handle(request)` e referência ao próximo handler.
- **ConcreteHandler**: trata a requisição ou repassa para o próximo; pode também **interromper** a cadeia (ex.: 401).
- **Client**: monta a cadeia e envia a requisição ao primeiro elo.

Duas variantes: (1) GoF clássico, **só um** handler trata; (2) *pipeline/middleware*, **todos** participam e cada um decide se continua.

### Trade-offs
| Prós | Contras |
|------|---------|
| Remetente não conhece quem vai tratar a requisição. | Sem garantia de tratamento: a requisição pode "cair" do fim da cadeia. |
| Etapas reordenáveis, reutilizáveis e testáveis isoladamente. | **A ordem importa** e fica implícita na montagem da cadeia. |
| SRP e Open/Closed: novas etapas sem mexer nas existentes. | Debugging mais difícil: qual handler respondeu? por que parou? |
| Base de middlewares (Servlet filters, Express, `net/http`). | Cadeias longas adicionam latência e profundidade de stack. |

### Quando usar / evitar
- **Use** para middleware HTTP, pipelines de validação, aprovações escalonadas (limite de alçada), tratamento de eventos em UI e interceptors.
- **Evite** quando há sempre exatamente um handler conhecido; um `switch` ou um mapa de handlers é mais claro.

---

## English

### Intent
Avoid coupling the sender of a request to its receiver by giving **multiple objects a chance to handle it**. Handlers are chained and the request travels along the chain until someone handles it (or stops it).

### Problem
An HTTP request must go through authentication, rate limiting, and validation before reaching the business handler. Putting it all in one giant method full of `if`s mixes responsibilities and prevents reordering or reusing steps.

### Structure
- **Handler**: an interface with `handle(request)` and a reference to the next handler.
- **ConcreteHandler**: handles the request or passes it on; it can also **short-circuit** the chain (e.g. 401).
- **Client**: builds the chain and sends the request to the first link.

Two variants: (1) classic GoF, **only one** handler handles it; (2) *pipeline/middleware*, **every** handler takes part and each decides whether to continue.

### Trade-offs
| Pros | Cons |
|------|------|
| The sender doesn't know who will handle the request. | No handling guarantee: the request may "fall off" the end of the chain. |
| Steps can be reordered, reused, and tested in isolation. | **Order matters** and is implicit in how the chain is built. |
| SRP and Open/Closed: new steps without touching existing ones. | Harder debugging: which handler responded? why did it stop? |
| The basis of middleware (Servlet filters, Express, `net/http`). | Long chains add latency and stack depth. |

### When to use / avoid
- **Use** for HTTP middleware, validation pipelines, tiered approvals (spending limits), UI event handling, and interceptors.
- **Avoid** when there's always exactly one known handler; a `switch` or a handler map is clearer.

---

## Exemplos / Examples

Scenario: a request goes through `Auth → RateLimit → Business handler`; any step can short-circuit with an error status.

### Java
```java
import java.util.HashMap;
import java.util.Map;

record Request(String user, String token, String path) {}
record Response(int status, String body) {}

abstract class Handler {
    private Handler next;

    Handler then(Handler next) { this.next = next; return next; }

    Response handle(Request req) {
        return next != null ? next.handle(req) : new Response(404, "no handler");
    }
}

class AuthHandler extends Handler {
    @Override Response handle(Request req) {
        if (!"secret".equals(req.token())) return new Response(401, "unauthorized");
        return super.handle(req);
    }
}

class RateLimitHandler extends Handler {
    private final Map<String, Integer> counts = new HashMap<>();
    private final int limit;
    RateLimitHandler(int limit) { this.limit = limit; }

    @Override Response handle(Request req) {
        int n = counts.merge(req.user(), 1, Integer::sum);
        if (n > limit) return new Response(429, "too many requests");
        return super.handle(req);
    }
}

class OrdersHandler extends Handler {
    @Override Response handle(Request req) { return new Response(200, "orders of " + req.user()); }
}

class Main {
    public static void main(String[] args) {
        Handler chain = new AuthHandler();
        chain.then(new RateLimitHandler(2)).then(new OrdersHandler());

        System.out.println(chain.handle(new Request("ana", "wrong", "/orders")));  // 401
        for (int i = 0; i < 3; i++)
            System.out.println(chain.handle(new Request("ana", "secret", "/orders"))); // 200, 200, 429
    }
}
```

### Kotlin
```kotlin
data class Request(val user: String, val token: String, val path: String)
data class Response(val status: Int, val body: String)

// Functional middleware: each step receives the request and the rest of the chain.
typealias Handler = (Request) -> Response
typealias Middleware = (Handler) -> Handler

val auth: Middleware = { next -> { req -> if (req.token != "secret") Response(401, "unauthorized") else next(req) } }

fun rateLimit(limit: Int): Middleware {
    val counts = mutableMapOf<String, Int>()
    return { next ->
        { req ->
            val n = (counts[req.user] ?: 0) + 1
            counts[req.user] = n
            if (n > limit) Response(429, "too many requests") else next(req)
        }
    }
}

val orders: Handler = { req -> Response(200, "orders of ${req.user}") }

fun chain(handler: Handler, vararg middlewares: Middleware): Handler =
    middlewares.foldRight(handler) { mw, acc -> mw(acc) }

fun main() {
    val app = chain(orders, auth, rateLimit(2))
    println(app(Request("ana", "wrong", "/orders")))
    repeat(3) { println(app(Request("ana", "secret", "/orders"))) }
}
```

### Go
```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync"
)

// Idiomatic Go: middleware is func(http.Handler) http.Handler.
func Auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer secret" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return // short-circuit
		}
		next.ServeHTTP(w, r)
	})
}

func RateLimit(limit int) func(http.Handler) http.Handler {
	var mu sync.Mutex
	counts := map[string]int{}
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			user := r.Header.Get("X-User")
			mu.Lock()
			counts[user]++
			n := counts[user]
			mu.Unlock()
			if n > limit {
				http.Error(w, "too many requests", http.StatusTooManyRequests)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

func Chain(h http.Handler, mws ...func(http.Handler) http.Handler) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

func main() {
	orders := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "orders of %s", r.Header.Get("X-User"))
	})
	app := Chain(orders, Auth, RateLimit(2))

	for _, token := range []string{"wrong", "secret", "secret", "secret"} {
		req := httptest.NewRequest("GET", "/orders", nil)
		req.Header.Set("Authorization", "Bearer "+token)
		req.Header.Set("X-User", "ana")
		rec := httptest.NewRecorder()
		app.ServeHTTP(rec, req)
		fmt.Println(rec.Code, rec.Body.String())
	}
}
```

### C++
```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>

struct Request { std::string user, token, path; };
struct Response { int status; std::string body; };

class Handler {
public:
    virtual ~Handler() = default;
    Handler* then(std::unique_ptr<Handler> next) { next_ = std::move(next); return next_.get(); }
    virtual Response handle(const Request& req) {
        return next_ ? next_->handle(req) : Response{404, "no handler"};
    }
private:
    std::unique_ptr<Handler> next_;
};

class AuthHandler : public Handler {
public:
    Response handle(const Request& req) override {
        if (req.token != "secret") return {401, "unauthorized"};
        return Handler::handle(req);
    }
};

class RateLimitHandler : public Handler {
public:
    explicit RateLimitHandler(int limit) : limit_(limit) {}
    Response handle(const Request& req) override {
        if (++counts_[req.user] > limit_) return {429, "too many requests"};
        return Handler::handle(req);
    }
private:
    int limit_;
    std::map<std::string, int> counts_;
};

class OrdersHandler : public Handler {
public:
    Response handle(const Request& req) override { return {200, "orders of " + req.user}; }
};

int main() {
    AuthHandler chain;
    chain.then(std::make_unique<RateLimitHandler>(2))->then(std::make_unique<OrdersHandler>());

    for (const char* token : {"wrong", "secret", "secret", "secret"}) {
        auto res = chain.handle({"ana", token, "/orders"});
        std::cout << res.status << " " << res.body << "\n";
    }
}
```

### Rust
```rust
use std::collections::HashMap;

struct Request<'a> { user: &'a str, token: &'a str }
#[allow(dead_code)] // fields are only read through Debug in this demo
#[derive(Debug)]
struct Response { status: u16, body: String }

// Each handler either answers or delegates to `next`.
trait Handler {
    fn handle(&mut self, req: &Request) -> Response;
}

struct Auth<N: Handler> { next: N }
impl<N: Handler> Handler for Auth<N> {
    fn handle(&mut self, req: &Request) -> Response {
        if req.token != "secret" {
            return Response { status: 401, body: "unauthorized".into() };
        }
        self.next.handle(req)
    }
}

struct RateLimit<N: Handler> { next: N, limit: u32, counts: HashMap<String, u32> }
impl<N: Handler> Handler for RateLimit<N> {
    fn handle(&mut self, req: &Request) -> Response {
        let n = self.counts.entry(req.user.to_string()).or_insert(0);
        *n += 1;
        if *n > self.limit {
            return Response { status: 429, body: "too many requests".into() };
        }
        self.next.handle(req)
    }
}

struct Orders;
impl Handler for Orders {
    fn handle(&mut self, req: &Request) -> Response {
        Response { status: 200, body: format!("orders of {}", req.user) }
    }
}

fn main() {
    let mut app = Auth { next: RateLimit { next: Orders, limit: 2, counts: HashMap::new() } };
    for token in ["wrong", "secret", "secret", "secret"] {
        println!("{:?}", app.handle(&Request { user: "ana", token }));
    }
}
```

### Python
```python
from collections import Counter
from collections.abc import Callable
from dataclasses import dataclass


@dataclass
class Request:
    user: str
    token: str
    path: str


@dataclass
class Response:
    status: int
    body: str


Handler = Callable[[Request], Response]
Middleware = Callable[[Handler], Handler]


def auth(next_: Handler) -> Handler:
    def handle(req: Request) -> Response:
        if req.token != "secret":
            return Response(401, "unauthorized")
        return next_(req)

    return handle


def rate_limit(limit: int) -> Middleware:
    counts: Counter[str] = Counter()

    def middleware(next_: Handler) -> Handler:
        def handle(req: Request) -> Response:
            counts[req.user] += 1
            if counts[req.user] > limit:
                return Response(429, "too many requests")
            return next_(req)

        return handle

    return middleware


def orders(req: Request) -> Response:
    return Response(200, f"orders of {req.user}")


def chain(handler: Handler, *middlewares: Middleware) -> Handler:
    for mw in reversed(middlewares):
        handler = mw(handler)
    return handler


if __name__ == "__main__":
    app = chain(orders, auth, rate_limit(2))
    for token in ["wrong", "secret", "secret", "secret"]:
        print(app(Request("ana", token, "/orders")))
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Servlet `FilterChain`, Spring Security filter chain, OkHttp interceptors. |
| Kotlin | Higher-order functions (`(Handler) -> Handler`); Ktor plugins/interceptors. |
| Go | `func(http.Handler) http.Handler` middleware; gRPC interceptors. |
| C++ | Linked handler objects or a `std::vector<std::function<...>>` pipeline. |
| Rust | Nested generic handlers (tower `Layer`s in axum/tonic) compile to a static chain. |
| Python | WSGI/ASGI middleware, Django middleware, decorator stacks. |

### Referências / References
- *Design Patterns* (GoF), Chain of Responsibility
- [Refactoring.Guru: Chain of Responsibility](https://refactoring.guru/design-patterns/chain-of-responsibility)
