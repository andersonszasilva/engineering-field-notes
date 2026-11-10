# Singleton

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / creational
**Tags:** gof, design-patterns, creational, singleton, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Garantir que uma classe tenha **uma única instância** e fornecer um **ponto de acesso global** a ela.

### Problema
Alguns recursos devem existir uma vez só por processo: um pool de conexões, um registry de métricas, a configuração carregada na inicialização. Criar várias instâncias desperdiça recursos ou gera estado inconsistente.

### Estrutura
- Construtor privado (ou equivalente) impede `new` de fora.
- Um campo estático guarda a instância.
- Um método/propriedade estático entrega a instância, criando-a de forma **lazy** e **thread-safe** se necessário.

### Trade-offs
| Prós | Contras |
|------|---------|
| Uma instância só, controle explícito do ciclo de vida. | É **estado global** disfarçado: acoplamento oculto entre módulos. |
| Inicialização lazy: só paga o custo se usar. | Dificulta testes: não dá para trocar por um mock sem hacks. |
| Ponto de acesso simples e conhecido. | Inicialização lazy concorrente é fácil de errar (double-checked locking). |
| | Viola o SRP: a classe cuida da lógica *e* do próprio ciclo de vida. |
| | "Um por processo" vira "um por classloader/pod/réplica" em sistemas distribuídos. |

### Quando usar / evitar
- **Use** para recursos realmente únicos e sem estado de negócio (logger, registry de métricas, cache do processo).
- **Prefira** injeção de dependência com escopo singleton (Spring, Guice, Koin, wiring manual no `main`): você ganha a instância única **sem** o acesso global.
- **Evite** como atalho para passar dependências, ou para estado mutável compartilhado entre requisições.

---

## English

### Intent
Ensure a class has **only one instance** and provide a **global access point** to it.

### Problem
Some resources should exist only once per process: a connection pool, a metrics registry, configuration loaded at startup. Creating several instances wastes resources or leads to inconsistent state.

### Structure
- A private constructor (or equivalent) prevents `new` from outside.
- A static field holds the instance.
- A static method/property returns the instance, creating it **lazily** and **thread-safely** if needed.

### Trade-offs
| Pros | Cons |
|------|------|
| One instance, explicit lifecycle control. | It is **global state** in disguise: hidden coupling between modules. |
| Lazy initialization: you only pay if you use it. | Hard to test: you can't swap it for a mock without hacks. |
| Simple, well-known access point. | Concurrent lazy initialization is easy to get wrong (double-checked locking). |
| | Violates SRP: the class handles its logic *and* its own lifecycle. |
| | "One per process" becomes "one per classloader/pod/replica" in distributed systems. |

### When to use / avoid
- **Use** for truly unique resources without business state (logger, metrics registry, process-wide cache).
- **Prefer** dependency injection with singleton scope (Spring, Guice, Koin, manual wiring in `main`): you get a single instance **without** global access.
- **Avoid** as a shortcut for passing dependencies, or for mutable state shared across requests.

---

## Exemplos / Examples

Scenario: an application-wide configuration object loaded once.

### Java
```java
// Initialization-on-demand holder: lazy and thread-safe, guaranteed by the JVM class loading.
final class AppConfig {
    private final String env;

    private AppConfig() {
        this.env = System.getenv().getOrDefault("APP_ENV", "dev");
    }

    private static final class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }

    static AppConfig getInstance() {
        return Holder.INSTANCE;
    }

    String env() { return env; }
}

// Alternative: `enum AppConfig { INSTANCE; }` is also serialization- and reflection-safe.

class Main {
    public static void main(String[] args) {
        System.out.println(AppConfig.getInstance() == AppConfig.getInstance()); // true
        System.out.println(AppConfig.getInstance().env());
    }
}
```

### Kotlin
```kotlin
// `object` is a language-level singleton: lazy and thread-safe on first access.
object AppConfig {
    val env: String = System.getenv("APP_ENV") ?: "dev"
}

fun main() {
    println(AppConfig === AppConfig) // true
    println(AppConfig.env)
}
```

### Go
```go
package main

import (
	"fmt"
	"os"
	"sync"
)

type AppConfig struct {
	Env string
}

var (
	instance *AppConfig
	once     sync.Once
)

// GetConfig initializes the instance exactly once, even under concurrent calls.
func GetConfig() *AppConfig {
	once.Do(func() {
		env := os.Getenv("APP_ENV")
		if env == "" {
			env = "dev"
		}
		instance = &AppConfig{Env: env}
	})
	return instance
}

func main() {
	fmt.Println(GetConfig() == GetConfig()) // true
	fmt.Println(GetConfig().Env)
}
```

### C++
```cpp
#include <cstdlib>
#include <iostream>
#include <string>

// Meyers singleton: function-local statics are initialized thread-safely since C++11.
class AppConfig {
public:
    static AppConfig& instance() {
        static AppConfig config;
        return config;
    }

    AppConfig(const AppConfig&) = delete;
    AppConfig& operator=(const AppConfig&) = delete;

    const std::string& env() const { return env_; }

private:
    AppConfig() {
        const char* e = std::getenv("APP_ENV");
        env_ = e ? e : "dev";
    }
    std::string env_;
};

int main() {
    std::cout << std::boolalpha << (&AppConfig::instance() == &AppConfig::instance()) << "\n";
    std::cout << AppConfig::instance().env() << "\n";
}
```

### Rust
```rust
use std::sync::OnceLock;

struct AppConfig {
    env: String,
}

// OnceLock (std since 1.70) gives lazy, thread-safe, one-time initialization.
fn config() -> &'static AppConfig {
    static INSTANCE: OnceLock<AppConfig> = OnceLock::new();
    INSTANCE.get_or_init(|| AppConfig {
        env: std::env::var("APP_ENV").unwrap_or_else(|_| "dev".to_string()),
    })
}

fn main() {
    println!("{}", std::ptr::eq(config(), config())); // true
    println!("{}", config().env);
}
```

### Python
```python
import os
from functools import cache


class AppConfig:
    def __init__(self) -> None:
        self.env = os.getenv("APP_ENV", "dev")


# Idiomatic Python: a module is already a singleton; a cached factory is the explicit version.
@cache
def get_config() -> AppConfig:
    return AppConfig()


if __name__ == "__main__":
    print(get_config() is get_config())  # True
    print(get_config().env)
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Holder idiom or `enum`. Avoid hand-written double-checked locking (needs `volatile`). In Spring, beans are singletons by default. |
| Kotlin | `object` declaration; `by lazy` for lazily computed values. |
| Go | `sync.Once` (or `sync.OnceValue` since Go 1.21). Package-level variables are the usual "singleton". |
| C++ | Meyers singleton. Beware the *static initialization order fiasco* across translation units. |
| Rust | `OnceLock` / `LazyLock` (1.80+). Global mutable state requires `Mutex`, making the cost visible. |
| Python | Modules are singletons. `@cache` on a factory is simple; metaclass-based singletons are rarely worth it. |

### Referências / References
- *Design Patterns: Elements of Reusable Object-Oriented Software*, Gamma, Helm, Johnson, Vlissides (GoF)
- *Effective Java*, Joshua Bloch, Item 3
- [Refactoring.Guru: Singleton](https://refactoring.guru/design-patterns/singleton)
