# Template Method

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, template-method, hooks, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Definir o **esqueleto de um algoritmo** em uma operação, delegando **alguns passos** para subclasses. O Template Method permite redefinir certos passos sem mudar a estrutura do algoritmo.

### Problema
Vários jobs de ETL seguem o mesmo fluxo: extrair, validar, transformar, carregar e registrar métricas. Só a extração e a transformação mudam entre fontes (CSV, API, banco). Copiar o fluxo em cada job duplica código e faz com que correções (ex.: tratamento de erro, métricas) precisem ser aplicadas em todos.

### Estrutura
- **AbstractClass**: implementa o **template method** (`run()`), normalmente `final`, que chama os passos em ordem.
- **Passos abstratos**: obrigatórios para as subclasses (`extract()`, `transform()`).
- **Hooks**: passos opcionais com implementação padrão (`beforeLoad()`), que as subclasses podem sobrescrever.
- **ConcreteClass**: implementa os passos específicos.

Princípio de Hollywood: *"don't call us, we'll call you"*. O framework controla o fluxo e chama seu código. Muitos frameworks (JUnit `setUp`/`tearDown`, `HttpServlet.service` → `doGet`) são template methods.

### Trade-offs
| Prós | Contras |
|------|---------|
| Reaproveita o fluxo comum; variações ficam pequenas. | Baseado em **herança**: acoplamento forte entre classe base e subclasses (*fragile base class*). |
| Garante a ordem dos passos e invariantes (logs, métricas, transações). | Fluxo "de cabeça para baixo": difícil entender lendo só a subclasse. |
| Pontos de extensão explícitos (hooks). | Herança única limita combinar variações; tende a hierarquias profundas. |
| | Pode violar Liskov se subclasses mudarem o significado dos passos. |

### Quando usar / evitar
- **Use** em frameworks e bibliotecas onde o fluxo é fixo e só alguns passos variam, especialmente quando invariantes precisam ser garantidos.
- **Prefira** composição (Strategy, funções passadas como parâmetro) quando as variações são independentes ou combináveis; é a abordagem idiomática em Go, Rust e código funcional.

---

## English

### Intent
Define the **skeleton of an algorithm** in an operation, deferring **some steps** to subclasses. Template Method lets you redefine certain steps without changing the algorithm's structure.

### Problem
Several ETL jobs follow the same flow: extract, validate, transform, load, and record metrics. Only extraction and transformation differ between sources (CSV, API, database). Copying the flow into each job duplicates code, and fixes (e.g. error handling, metrics) must then be applied everywhere.

### Structure
- **AbstractClass**: implements the **template method** (`run()`), usually `final`, which calls the steps in order.
- **Abstract steps**: required for subclasses (`extract()`, `transform()`).
- **Hooks**: optional steps with a default implementation (`beforeLoad()`) that subclasses may override.
- **ConcreteClass**: implements the specific steps.

Hollywood principle: *"don't call us, we'll call you"*. The framework controls the flow and calls your code. Many frameworks (JUnit `setUp`/`tearDown`, `HttpServlet.service` → `doGet`) are template methods.

### Trade-offs
| Pros | Cons |
|------|------|
| Reuses the common flow; variations stay small. | **Inheritance-based**: strong coupling between base class and subclasses (*fragile base class*). |
| Guarantees step order and invariants (logging, metrics, transactions). | "Upside-down" flow: hard to understand by reading only the subclass. |
| Explicit extension points (hooks). | Single inheritance limits combining variations; tends toward deep hierarchies. |
| | May violate Liskov if subclasses change what steps mean. |

### When to use / avoid
- **Use** in frameworks and libraries where the flow is fixed and only a few steps vary, especially when invariants must be guaranteed.
- **Prefer** composition (Strategy, functions passed as parameters) when variations are independent or combinable; this is the idiomatic approach in Go, Rust, and functional code.

---

## Exemplos / Examples

Scenario: an ETL job skeleton (`extract → transform → hook → load`) with one concrete CSV job.

### Java
```java
import java.util.List;

abstract class EtlJob {
    // The template method: final, so subclasses can't change the flow.
    public final void run() {
        long start = System.currentTimeMillis();
        List<String> raw = extract();
        List<String> rows = transform(raw);
        beforeLoad(rows);
        load(rows);
        System.out.printf("%s loaded %d rows in %d ms%n", name(), rows.size(), System.currentTimeMillis() - start);
    }

    protected abstract String name();
    protected abstract List<String> extract();
    protected abstract List<String> transform(List<String> raw);

    protected void beforeLoad(List<String> rows) {}                 // hook with default

    protected void load(List<String> rows) {                         // shared default step
        rows.forEach(r -> System.out.println("  INSERT " + r));
    }
}

class CsvCustomersJob extends EtlJob {
    protected String name() { return "csv-customers"; }
    protected List<String> extract() { return List.of("ana;BR", " bruno;PT ", ""); }
    protected List<String> transform(List<String> raw) {
        return raw.stream().map(String::trim).filter(s -> !s.isEmpty()).map(String::toUpperCase).toList();
    }
    @Override protected void beforeLoad(List<String> rows) { System.out.println("  dedup check on " + rows.size() + " rows"); }
}

class Main {
    public static void main(String[] args) {
        new CsvCustomersJob().run();
    }
}
```

### Kotlin
```kotlin
abstract class EtlJob {
    // Kotlin methods are final by default; only `abstract`/`open` ones can be overridden.
    fun run() {
        val start = System.currentTimeMillis()
        val rows = transform(extract())
        beforeLoad(rows)
        load(rows)
        println("$name loaded ${rows.size} rows in ${System.currentTimeMillis() - start} ms")
    }

    protected abstract val name: String
    protected abstract fun extract(): List<String>
    protected abstract fun transform(raw: List<String>): List<String>
    protected open fun beforeLoad(rows: List<String>) {}
    protected open fun load(rows: List<String>) = rows.forEach { println("  INSERT $it") }
}

class CsvCustomersJob : EtlJob() {
    override val name = "csv-customers"
    override fun extract() = listOf("ana;BR", " bruno;PT ", "")
    override fun transform(raw: List<String>) = raw.map(String::trim).filter(String::isNotEmpty).map(String::uppercase)
}

// Composition alternative: the "template" is a function taking the steps.
fun runEtl(name: String, extract: () -> List<String>, transform: (List<String>) -> List<String>) {
    val rows = transform(extract())
    rows.forEach { println("  INSERT $it") }
    println("$name loaded ${rows.size} rows")
}

fun main() {
    CsvCustomersJob().run()
    runEtl("api-orders", { listOf("o1", "o2") }, { it.map { r -> "order:$r" } })
}
```

### Go
```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// Go has no inheritance: the "abstract steps" become an interface,
// and the template method is a function that drives it.
type Steps interface {
	Name() string
	Extract() []string
	Transform(raw []string) []string
}

// Optional hook, detected with a type assertion.
type BeforeLoader interface {
	BeforeLoad(rows []string)
}

func RunETL(job Steps) {
	start := time.Now()
	rows := job.Transform(job.Extract())
	if h, ok := job.(BeforeLoader); ok {
		h.BeforeLoad(rows)
	}
	for _, r := range rows {
		fmt.Println("  INSERT", r)
	}
	fmt.Printf("%s loaded %d rows in %s\n", job.Name(), len(rows), time.Since(start))
}

type CsvCustomers struct{}

func (CsvCustomers) Name() string      { return "csv-customers" }
func (CsvCustomers) Extract() []string { return []string{"ana;BR", " bruno;PT ", ""} }
func (CsvCustomers) Transform(raw []string) []string {
	var out []string
	for _, r := range raw {
		if r = strings.TrimSpace(r); r != "" {
			out = append(out, strings.ToUpper(r))
		}
	}
	return out
}
func (CsvCustomers) BeforeLoad(rows []string) { fmt.Println("  dedup check on", len(rows), "rows") }

func main() {
	RunETL(CsvCustomers{})
}
```

### C++
```cpp
#include <algorithm>
#include <cctype>
#include <iostream>
#include <string>
#include <vector>

class EtlJob {
public:
    virtual ~EtlJob() = default;

    // Non-virtual interface (NVI) idiom: public non-virtual template, private virtual steps.
    void run() {
        auto rows = transform(extract());
        beforeLoad(rows);
        for (const auto& r : rows) std::cout << "  INSERT " << r << "\n";
        std::cout << name() << " loaded " << rows.size() << " rows\n";
    }

private:
    virtual std::string name() const = 0;
    virtual std::vector<std::string> extract() = 0;
    virtual std::vector<std::string> transform(std::vector<std::string> raw) = 0;
    virtual void beforeLoad(const std::vector<std::string>&) {}  // hook
};

class CsvCustomersJob : public EtlJob {
    std::string name() const override { return "csv-customers"; }
    std::vector<std::string> extract() override { return {"ana;BR", " bruno;PT ", ""}; }
    std::vector<std::string> transform(std::vector<std::string> raw) override {
        std::vector<std::string> out;
        for (auto r : raw) {
            r.erase(0, r.find_first_not_of(' '));
            r.erase(r.find_last_not_of(' ') + 1);
            if (r.empty()) continue;
            std::transform(r.begin(), r.end(), r.begin(), [](unsigned char c) { return std::toupper(c); });
            out.push_back(r);
        }
        return out;
    }
    void beforeLoad(const std::vector<std::string>& rows) override {
        std::cout << "  dedup check on " << rows.size() << " rows\n";
    }
};

int main() {
    CsvCustomersJob job;
    job.run();
}
```

### Rust
```rust
// Trait default methods act as the template; required methods are the abstract steps.
trait EtlJob {
    fn name(&self) -> &str;
    fn extract(&self) -> Vec<String>;
    fn transform(&self, raw: Vec<String>) -> Vec<String>;

    fn before_load(&self, _rows: &[String]) {} // hook with default

    // Template method. Rust can't mark it `final`; by convention, implementors don't override it.
    fn run(&self) {
        let rows = self.transform(self.extract());
        self.before_load(&rows);
        for r in &rows {
            println!("  INSERT {r}");
        }
        println!("{} loaded {} rows", self.name(), rows.len());
    }
}

struct CsvCustomers;

impl EtlJob for CsvCustomers {
    fn name(&self) -> &str { "csv-customers" }

    fn extract(&self) -> Vec<String> {
        vec!["ana;BR".into(), " bruno;PT ".into(), "".into()]
    }

    fn transform(&self, raw: Vec<String>) -> Vec<String> {
        raw.iter().map(|s| s.trim()).filter(|s| !s.is_empty()).map(str::to_uppercase).collect()
    }

    fn before_load(&self, rows: &[String]) {
        println!("  dedup check on {} rows", rows.len());
    }
}

fn main() {
    CsvCustomers.run();
}
```

### Python
```python
import time
from abc import ABC, abstractmethod
from typing import final


class EtlJob(ABC):
    name: str = "etl"

    @final  # type checkers flag overrides; not enforced at runtime
    def run(self) -> None:
        start = time.perf_counter()
        rows = self.transform(self.extract())
        self.before_load(rows)
        self.load(rows)
        print(f"{self.name} loaded {len(rows)} rows in {(time.perf_counter() - start) * 1000:.1f} ms")

    @abstractmethod
    def extract(self) -> list[str]: ...

    @abstractmethod
    def transform(self, raw: list[str]) -> list[str]: ...

    def before_load(self, rows: list[str]) -> None:  # hook
        pass

    def load(self, rows: list[str]) -> None:
        for r in rows:
            print(f"  INSERT {r}")


class CsvCustomersJob(EtlJob):
    name = "csv-customers"

    def extract(self) -> list[str]:
        return ["ana;BR", " bruno;PT ", ""]

    def transform(self, raw: list[str]) -> list[str]:
        return [r.strip().upper() for r in raw if r.strip()]

    def before_load(self, rows: list[str]) -> None:
        print(f"  dedup check on {len(rows)} rows")


if __name__ == "__main__":
    CsvCustomersJob().run()
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `AbstractList`, `HttpServlet`, Spring `JdbcTemplate` (which actually uses callbacks: template + strategy). Mark the template `final`. |
| Kotlin | Classes and methods are final by default, which makes the template safe; consider higher-order functions instead. |
| Go | No inheritance: a driver function over an interface; optional hooks via type assertion. |
| C++ | Non-Virtual Interface (NVI) idiom; CRTP for compile-time templates. |
| Rust | Trait default methods; no `final`, so document that the template shouldn't be overridden. |
| Python | ABC + `@abstractmethod`; `typing.final` is checked only by type checkers. |

### Referências / References
- *Design Patterns* (GoF), Template Method
- Herb Sutter, [Virtuality](http://www.gotw.ca/publications/mill18.htm) (NVI idiom)
- [Refactoring.Guru: Template Method](https://refactoring.guru/design-patterns/template-method)
