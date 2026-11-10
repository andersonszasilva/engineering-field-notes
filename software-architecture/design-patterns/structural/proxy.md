# Proxy

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / structural
**Tags:** gof, design-patterns, structural, proxy, lazy-loading, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Fornecer um **substituto** para outro objeto, com a mesma interface, para **controlar o acesso** a ele.

### Problema
Carregar uma imagem de alta resolução do storage é caro e nem sempre ela será exibida. Em outros casos o objeto real está em outra máquina (RPC), precisa de checagem de permissão antes de cada chamada, ou suas respostas deveriam ser cacheadas. O cliente não deveria precisar saber disso.

### Estrutura
- **Subject**: interface comum (`Image`).
- **RealSubject**: o objeto real e caro (`HighResImage`).
- **Proxy**: mesma interface; guarda (ou cria sob demanda) o RealSubject e controla o acesso.

Tipos comuns:
- **Virtual proxy**: criação lazy de objetos caros (lazy loading do Hibernate).
- **Remote proxy**: representa um objeto em outro processo (stubs gRPC, Java RMI).
- **Protection proxy**: checa permissões antes de delegar.
- **Caching / smart proxy**: cache, contagem de referências, logging, rate limiting.

Proxy vs Decorator: estrutura quase idêntica. O Decorator **adiciona comportamento** e é composto pelo cliente; o Proxy **controla acesso** e normalmente gerencia o ciclo de vida do objeto real.

### Trade-offs
| Prós | Contras |
|------|---------|
| Controle de acesso, lazy loading e cache transparentes ao cliente. | Latência e falhas escondidas: uma chamada "local" pode ser de rede (*fallacies of distributed computing*). |
| Open/Closed: adiciona controle sem mudar o objeto real. | Transparência demais gera surpresas (N+1 queries com lazy loading, `LazyInitializationException`). |
| Proxies dinâmicos permitem AOP genérico (transações, segurança). | Mais uma indireção; proxies dinâmicos dificultam debugging. |
| | Inicialização lazy concorrente exige sincronização. |

### Quando usar / evitar
- **Use** para lazy loading de recursos caros, acesso remoto, controle de acesso, cache e rate limiting de clientes.
- **Evite** esconder chamadas de rede atrás de uma interface que parece local sem expor timeouts e erros.

---

## English

### Intent
Provide a **surrogate** for another object, with the same interface, to **control access** to it.

### Problem
Loading a high-resolution image from storage is expensive and it may never be displayed. In other cases the real object lives on another machine (RPC), needs a permission check before each call, or its responses should be cached. The client shouldn't need to know.

### Structure
- **Subject**: common interface (`Image`).
- **RealSubject**: the real, expensive object (`HighResImage`).
- **Proxy**: same interface; holds (or lazily creates) the RealSubject and controls access.

Common kinds:
- **Virtual proxy**: lazy creation of expensive objects (Hibernate lazy loading).
- **Remote proxy**: represents an object in another process (gRPC stubs, Java RMI).
- **Protection proxy**: checks permissions before delegating.
- **Caching / smart proxy**: caching, reference counting, logging, rate limiting.

Proxy vs Decorator: nearly identical structure. A Decorator **adds behavior** and is composed by the client; a Proxy **controls access** and usually manages the real object's lifecycle.

### Trade-offs
| Pros | Cons |
|------|------|
| Access control, lazy loading, and caching transparent to the client. | Hidden latency and failures: a "local" call may be a network call (*fallacies of distributed computing*). |
| Open/Closed: adds control without changing the real object. | Too much transparency surprises people (N+1 queries with lazy loading, `LazyInitializationException`). |
| Dynamic proxies enable generic AOP (transactions, security). | One more indirection; dynamic proxies make debugging harder. |
| | Concurrent lazy initialization requires synchronization. |

### When to use / avoid
- **Use** for lazy loading expensive resources, remote access, access control, caching, and client-side rate limiting.
- **Avoid** hiding network calls behind an interface that looks local without exposing timeouts and errors.

---

## Exemplos / Examples

Scenario: a **virtual proxy** that loads a heavy image only on first `display()`, plus a **protection** check.

### Java
```java
import java.util.Set;

interface Image {
    void display(String user);
}

class HighResImage implements Image {
    private final String path;
    HighResImage(String path) {
        this.path = path;
        System.out.println("loading " + path + " from storage (expensive)");
    }
    public void display(String user) { System.out.println("displaying " + path + " to " + user); }
}

class ImageProxy implements Image {
    private final String path;
    private final Set<String> allowedUsers;
    private HighResImage real; // created lazily

    ImageProxy(String path, Set<String> allowedUsers) {
        this.path = path;
        this.allowedUsers = allowedUsers;
    }

    @Override public void display(String user) {
        if (!allowedUsers.contains(user)) throw new SecurityException(user + " is not allowed");
        if (real == null) real = new HighResImage(path); // not thread-safe; synchronize if shared
        real.display(user);
    }
}

class Main {
    public static void main(String[] args) {
        Image image = new ImageProxy("photos/aurora.raw", Set.of("ana"));
        System.out.println("proxy created, nothing loaded yet");
        image.display("ana"); // loads, then displays
        image.display("ana"); // already loaded
        // java.lang.reflect.Proxy creates proxies at runtime for any interface.
    }
}
```

### Kotlin
```kotlin
interface Image {
    fun display(user: String)
}

class HighResImage(private val path: String) : Image {
    init { println("loading $path from storage (expensive)") }
    override fun display(user: String) = println("displaying $path to $user")
}

class ImageProxy(private val path: String, private val allowedUsers: Set<String>) : Image {
    // `by lazy` is a thread-safe virtual proxy for a value.
    private val real by lazy { HighResImage(path) }

    override fun display(user: String) {
        if (user !in allowedUsers) throw SecurityException("$user is not allowed")
        real.display(user)
    }
}

fun main() {
    val image: Image = ImageProxy("photos/aurora.raw", setOf("ana"))
    println("proxy created, nothing loaded yet")
    image.display("ana")
    image.display("ana")
}
```

### Go
```go
package main

import (
	"fmt"
	"sync"
)

type Image interface {
	Display(user string) error
}

type highResImage struct{ path string }

func loadHighRes(path string) *highResImage {
	fmt.Println("loading", path, "from storage (expensive)")
	return &highResImage{path: path}
}

func (i *highResImage) Display(user string) error {
	fmt.Println("displaying", i.path, "to", user)
	return nil
}

type ImageProxy struct {
	path    string
	allowed map[string]bool
	once    sync.Once
	real    *highResImage
}

func (p *ImageProxy) Display(user string) error {
	if !p.allowed[user] {
		return fmt.Errorf("%s is not allowed", user)
	}
	p.once.Do(func() { p.real = loadHighRes(p.path) }) // lazy and goroutine-safe
	return p.real.Display(user)
}

func main() {
	var img Image = &ImageProxy{path: "photos/aurora.raw", allowed: map[string]bool{"ana": true}}
	fmt.Println("proxy created, nothing loaded yet")
	_ = img.Display("ana")
	_ = img.Display("ana")
	fmt.Println(img.Display("bob"))
}
```

### C++
```cpp
#include <iostream>
#include <memory>
#include <set>
#include <stdexcept>
#include <string>

struct Image {
    virtual ~Image() = default;
    virtual void display(const std::string& user) = 0;
};

class HighResImage : public Image {
public:
    explicit HighResImage(std::string path) : path_(std::move(path)) {
        std::cout << "loading " << path_ << " from storage (expensive)\n";
    }
    void display(const std::string& user) override { std::cout << "displaying " << path_ << " to " << user << "\n"; }
private:
    std::string path_;
};

class ImageProxy : public Image {
public:
    ImageProxy(std::string path, std::set<std::string> allowed) : path_(std::move(path)), allowed_(std::move(allowed)) {}

    void display(const std::string& user) override {
        if (!allowed_.count(user)) throw std::runtime_error(user + " is not allowed");
        if (!real_) real_ = std::make_unique<HighResImage>(path_);
        real_->display(user);
    }
private:
    std::string path_;
    std::set<std::string> allowed_;
    std::unique_ptr<HighResImage> real_;
};

int main() {
    ImageProxy image("photos/aurora.raw", {"ana"});
    std::cout << "proxy created, nothing loaded yet\n";
    image.display("ana");
    image.display("ana");
}
```

### Rust
```rust
use std::cell::OnceCell;
use std::collections::HashSet;

trait Image {
    fn display(&self, user: &str) -> Result<(), String>;
}

struct HighResImage {
    path: String,
}

impl HighResImage {
    fn load(path: &str) -> Self {
        println!("loading {path} from storage (expensive)");
        Self { path: path.to_string() }
    }
}

impl Image for HighResImage {
    fn display(&self, user: &str) -> Result<(), String> {
        println!("displaying {} to {user}", self.path);
        Ok(())
    }
}

struct ImageProxy {
    path: String,
    allowed: HashSet<String>,
    real: OnceCell<HighResImage>, // use OnceLock when shared across threads
}

impl Image for ImageProxy {
    fn display(&self, user: &str) -> Result<(), String> {
        if !self.allowed.contains(user) {
            return Err(format!("{user} is not allowed"));
        }
        self.real.get_or_init(|| HighResImage::load(&self.path)).display(user)
    }
}

fn main() {
    let image = ImageProxy {
        path: "photos/aurora.raw".into(),
        allowed: HashSet::from(["ana".to_string()]),
        real: OnceCell::new(),
    };
    println!("proxy created, nothing loaded yet");
    image.display("ana").unwrap();
    image.display("ana").unwrap();
    println!("{:?}", image.display("bob"));
}
```

### Python
```python
from functools import cached_property


class HighResImage:
    def __init__(self, path: str) -> None:
        print(f"loading {path} from storage (expensive)")
        self.path = path

    def display(self, user: str) -> None:
        print(f"displaying {self.path} to {user}")


class ImageProxy:
    def __init__(self, path: str, allowed_users: set[str]) -> None:
        self._path = path
        self._allowed = allowed_users

    @cached_property  # computed on first access, then stored on the instance
    def _real(self) -> HighResImage:
        return HighResImage(self._path)

    def display(self, user: str) -> None:
        if user not in self._allowed:
            raise PermissionError(f"{user} is not allowed")
        self._real.display(user)


if __name__ == "__main__":
    image = ImageProxy("photos/aurora.raw", {"ana"})
    print("proxy created, nothing loaded yet")
    image.display("ana")
    image.display("ana")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `java.lang.reflect.Proxy`, CGLIB/ByteBuddy; Spring `@Transactional`/`@Cacheable` and Hibernate lazy entities are proxies. |
| Kotlin | `by lazy` (virtual proxy) and delegated properties (`by Delegates.observable`). |
| Go | `sync.Once` for lazy init; gRPC client stubs are remote proxies; `httputil.ReverseProxy`. |
| C++ | Smart pointers are proxies (`shared_ptr` does reference counting); `operator->` overloading. |
| Rust | `OnceCell`/`LazyLock`, `Deref` for smart pointers, `Rc`/`Arc`. |
| Python | `cached_property`, `__getattr__`-based proxies, `weakref.proxy`. |

### Referências / References
- *Design Patterns* (GoF), Proxy
- [Fallacies of distributed computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)
- [Refactoring.Guru: Proxy](https://refactoring.guru/design-patterns/proxy)
