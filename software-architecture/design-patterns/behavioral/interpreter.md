# Interpreter

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, interpreter, dsl, ast, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Dada uma linguagem, definir uma **representação para sua gramática** junto com um interpretador que usa essa representação para **avaliar sentenças** da linguagem.

### Problema
Regras de negócio que mudam com frequência, como "habilitar a feature se `country == BR` **e** (`plan == pro` **ou** `beta == true`)", não deveriam exigir deploy. Elas podem ser expressas numa pequena linguagem (DSL), guardadas como dados e avaliadas em runtime.

### Estrutura
- **AbstractExpression**: interface com `interpret(context)`.
- **TerminalExpression**: folhas da gramática (literais, variáveis, comparações simples).
- **NonterminalExpression**: regras compostas (`And`, `Or`, `Not`), que contêm outras expressões. É um **Composite** especializado.
- **Context**: informações globais para a avaliação (atributos do usuário, variáveis).
- O **parser** (texto → árvore) não faz parte do padrão; costuma ser escrito à mão (*recursive descent*) ou gerado (ANTLR).

### Trade-offs
| Prós | Contras |
|------|---------|
| Gramática simples vira código direto: uma classe por regra. | Gramáticas grandes geram dezenas de classes difíceis de manter. |
| Fácil estender com novas expressões. | Performance ruim comparada a compilação/bytecode para uso intenso. |
| Regras como dados: configuráveis em runtime, sem deploy. | Você está criando uma linguagem: precisa de erros claros, ferramentas, documentação, versionamento. |
| Combina com Visitor para várias operações (avaliar, imprimir, otimizar). | Risco de segurança se a DSL permitir execução arbitrária. |

### Quando usar / evitar
- **Use** para DSLs pequenas e estáveis: regras de feature flags, filtros de busca, regras de roteamento/alerta, expressões de política.
- **Evite** quando já existe uma linguagem pronta (CEL, JSONLogic, SpEL, Rego, SQL, expressões regulares) ou quando a gramática é grande.

---

## English

### Intent
Given a language, define a **representation for its grammar** along with an interpreter that uses the representation to **evaluate sentences** in the language.

### Problem
Frequently changing business rules, such as "enable the feature if `country == BR` **and** (`plan == pro` **or** `beta == true`)", shouldn't require a deploy. They can be expressed in a small language (DSL), stored as data, and evaluated at runtime.

### Structure
- **AbstractExpression**: an interface with `interpret(context)`.
- **TerminalExpression**: grammar leaves (literals, variables, simple comparisons).
- **NonterminalExpression**: composite rules (`And`, `Or`, `Not`) that contain other expressions. A specialized **Composite**.
- **Context**: global information for evaluation (user attributes, variables).
- The **parser** (text → tree) is not part of the pattern; it's usually hand-written (*recursive descent*) or generated (ANTLR).

### Trade-offs
| Pros | Cons |
|------|------|
| A simple grammar maps directly to code: one class per rule. | Large grammars produce dozens of classes that are hard to maintain. |
| Easy to extend with new expressions. | Poor performance compared to compiling/bytecode under heavy use. |
| Rules as data: configurable at runtime, no deploy. | You are building a language: it needs clear errors, tooling, docs, versioning. |
| Pairs with Visitor for several operations (evaluate, print, optimize). | Security risk if the DSL allows arbitrary execution. |

### When to use / avoid
- **Use** for small, stable DSLs: feature-flag rules, search filters, routing/alerting rules, policy expressions.
- **Avoid** when a ready-made language exists (CEL, JSONLogic, SpEL, Rego, SQL, regular expressions) or when the grammar is large.

---

## Exemplos / Examples

Scenario: feature-flag rule `country == "BR" AND (plan == "pro" OR NOT beta == "false")`, evaluated against user attributes. (The parser is omitted; the AST is built directly.)

### Java
```java
import java.util.Map;

// Grammar as a sealed hierarchy of records.
sealed interface Expr permits Eq, And, Or, Not {
    boolean eval(Map<String, String> ctx);
}

record Eq(String attr, String value) implements Expr {          // terminal
    public boolean eval(Map<String, String> ctx) { return value.equals(ctx.get(attr)); }
}
record And(Expr left, Expr right) implements Expr {              // nonterminal
    public boolean eval(Map<String, String> ctx) { return left.eval(ctx) && right.eval(ctx); }
}
record Or(Expr left, Expr right) implements Expr {
    public boolean eval(Map<String, String> ctx) { return left.eval(ctx) || right.eval(ctx); }
}
record Not(Expr inner) implements Expr {
    public boolean eval(Map<String, String> ctx) { return !inner.eval(ctx); }
}

class Main {
    public static void main(String[] args) {
        Expr rule = new And(
                new Eq("country", "BR"),
                new Or(new Eq("plan", "pro"), new Not(new Eq("beta", "false"))));

        System.out.println(rule.eval(Map.of("country", "BR", "plan", "free", "beta", "true")));  // true
        System.out.println(rule.eval(Map.of("country", "BR", "plan", "free", "beta", "false"))); // false
        System.out.println(rule); // records give a readable AST for free
    }
}
```

### Kotlin
```kotlin
sealed interface Expr
data class Eq(val attr: String, val value: String) : Expr
data class And(val left: Expr, val right: Expr) : Expr
data class Or(val left: Expr, val right: Expr) : Expr
data class Not(val inner: Expr) : Expr

// The interpreter as one exhaustive `when` (functional style) instead of a method per class.
fun Expr.eval(ctx: Map<String, String>): Boolean = when (this) {
    is Eq -> ctx[attr] == value
    is And -> left.eval(ctx) && right.eval(ctx)
    is Or -> left.eval(ctx) || right.eval(ctx)
    is Not -> !inner.eval(ctx)
}

// Small infix DSL to build rules readably.
infix fun String.eq(v: String) = Eq(this, v)
infix fun Expr.and(o: Expr) = And(this, o)
infix fun Expr.or(o: Expr) = Or(this, o)

fun main() {
    val rule = ("country" eq "BR") and (("plan" eq "pro") or Not("beta" eq "false"))
    println(rule.eval(mapOf("country" to "BR", "plan" to "free", "beta" to "true")))  // true
    println(rule.eval(mapOf("country" to "BR", "plan" to "free", "beta" to "false"))) // false
}
```

### Go
```go
package main

import "fmt"

type Context map[string]string

type Expr interface {
	Eval(ctx Context) bool
}

type Eq struct{ Attr, Value string }
type And struct{ Left, Right Expr }
type Or struct{ Left, Right Expr }
type Not struct{ Inner Expr }

func (e Eq) Eval(ctx Context) bool  { return ctx[e.Attr] == e.Value }
func (e And) Eval(ctx Context) bool { return e.Left.Eval(ctx) && e.Right.Eval(ctx) }
func (e Or) Eval(ctx Context) bool  { return e.Left.Eval(ctx) || e.Right.Eval(ctx) }
func (e Not) Eval(ctx Context) bool { return !e.Inner.Eval(ctx) }

func main() {
	rule := And{
		Eq{"country", "BR"},
		Or{Eq{"plan", "pro"}, Not{Eq{"beta", "false"}}},
	}
	fmt.Println(rule.Eval(Context{"country": "BR", "plan": "free", "beta": "true"}))  // true
	fmt.Println(rule.Eval(Context{"country": "BR", "plan": "free", "beta": "false"})) // false
}
```

### C++
```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>
#include <variant>

using Context = std::map<std::string, std::string>;

// Grammar as a std::variant; recursion through unique_ptr.
struct Eq; struct And; struct Or; struct Not;
using Expr = std::variant<Eq, And, Or, Not>;
using ExprPtr = std::unique_ptr<Expr>;

struct Eq { std::string attr, value; };
struct And { ExprPtr left, right; };
struct Or { ExprPtr left, right; };
struct Not { ExprPtr inner; };

bool eval(const Expr& e, const Context& ctx);

struct Evaluator {
    const Context& ctx;
    bool operator()(const Eq& e) const {
        auto it = ctx.find(e.attr);
        return it != ctx.end() && it->second == e.value;
    }
    bool operator()(const And& e) const { return eval(*e.left, ctx) && eval(*e.right, ctx); }
    bool operator()(const Or& e) const { return eval(*e.left, ctx) || eval(*e.right, ctx); }
    bool operator()(const Not& e) const { return !eval(*e.inner, ctx); }
};

bool eval(const Expr& e, const Context& ctx) { return std::visit(Evaluator{ctx}, e); }

template <class T, class... Args>
ExprPtr make(Args&&... args) { return std::make_unique<Expr>(T{std::forward<Args>(args)...}); }

int main() {
    auto rule = make<And>(
        make<Eq>("country", "BR"),
        make<Or>(make<Eq>("plan", "pro"), make<Not>(make<Eq>("beta", "false"))));

    std::cout << std::boolalpha
              << eval(*rule, {{"country", "BR"}, {"plan", "free"}, {"beta", "true"}}) << "\n"    // true
              << eval(*rule, {{"country", "BR"}, {"plan", "free"}, {"beta", "false"}}) << "\n";  // false
}
```

### Rust
```rust
use std::collections::HashMap;

// Enums + Box are the natural AST representation in Rust.
enum Expr {
    Eq(String, String),
    And(Box<Expr>, Box<Expr>),
    Or(Box<Expr>, Box<Expr>),
    Not(Box<Expr>),
}

impl Expr {
    fn eval(&self, ctx: &HashMap<&str, &str>) -> bool {
        match self {
            Expr::Eq(attr, value) => ctx.get(attr.as_str()) == Some(&value.as_str()),
            Expr::And(l, r) => l.eval(ctx) && r.eval(ctx),
            Expr::Or(l, r) => l.eval(ctx) || r.eval(ctx),
            Expr::Not(e) => !e.eval(ctx),
        }
    }
}

fn eq(a: &str, v: &str) -> Expr { Expr::Eq(a.into(), v.into()) }

fn main() {
    use Expr::*;
    let rule = And(
        Box::new(eq("country", "BR")),
        Box::new(Or(Box::new(eq("plan", "pro")), Box::new(Not(Box::new(eq("beta", "false")))))),
    );

    let user1 = HashMap::from([("country", "BR"), ("plan", "free"), ("beta", "true")]);
    let user2 = HashMap::from([("country", "BR"), ("plan", "free"), ("beta", "false")]);
    println!("{}", rule.eval(&user1)); // true
    println!("{}", rule.eval(&user2)); // false
}
```

### Python
```python
from __future__ import annotations

from dataclasses import dataclass

Context = dict[str, str]


@dataclass(frozen=True)
class Eq:
    attr: str
    value: str


@dataclass(frozen=True)
class And:
    left: Expr
    right: Expr


@dataclass(frozen=True)
class Or:
    left: Expr
    right: Expr


@dataclass(frozen=True)
class Not:
    inner: Expr


Expr = Eq | And | Or | Not


# Structural pattern matching (3.10+) makes the interpreter a single function.
def evaluate(expr: Expr, ctx: Context) -> bool:
    match expr:
        case Eq(attr, value):
            return ctx.get(attr) == value
        case And(left, right):
            return evaluate(left, ctx) and evaluate(right, ctx)
        case Or(left, right):
            return evaluate(left, ctx) or evaluate(right, ctx)
        case Not(inner):
            return not evaluate(inner, ctx)
    raise TypeError(f"unknown expression: {expr!r}")


if __name__ == "__main__":
    rule = And(Eq("country", "BR"), Or(Eq("plan", "pro"), Not(Eq("beta", "false"))))
    print(evaluate(rule, {"country": "BR", "plan": "free", "beta": "true"}))   # True
    print(evaluate(rule, {"country": "BR", "plan": "free", "beta": "false"}))  # False
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Sealed records + pattern-matching `switch`; `java.util.regex.Pattern` and SpEL are interpreters. |
| Kotlin | Sealed classes + `when`; infix functions and lambdas with receivers for internal DSLs. |
| Go | Interfaces per node; `text/template` and `go/ast` show the pattern in the standard library. |
| C++ | `std::variant` + `std::visit` for closed ASTs; expression templates for compile-time DSLs. |
| Rust | Enums + `Box` for ASTs; `nom`/`pest` for parsing. |
| Python | Dataclasses + `match`; the `ast` module exposes Python's own interpreter tree. |

### Referências / References
- *Design Patterns* (GoF), Interpreter
- Robert Nystrom, [*Crafting Interpreters*](https://craftinginterpreters.com/)
- [CEL: Common Expression Language](https://cel.dev/)
- Martin Fowler, *Domain-Specific Languages*
