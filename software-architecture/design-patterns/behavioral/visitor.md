# Visitor

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, visitor, double-dispatch, pattern-matching, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Representar uma **operação a ser executada sobre os elementos de uma estrutura** de objetos. O Visitor permite definir uma nova operação **sem mudar as classes** dos elementos sobre os quais opera.

### Problema
Um editor de diagramas tem formas (`Circle`, `Rectangle`) e precisa de várias operações: calcular área, exportar para SVG, exportar para JSON, validar. Colocar cada operação como método em cada forma mistura responsabilidades (geometria + serialização + validação) e cada nova operação exige editar todas as classes.

### Estrutura
- **Visitor**: interface com um método `visit` por tipo concreto de elemento (`visitCircle`, `visitRectangle`).
- **ConcreteVisitor**: implementa uma operação para todos os tipos (`AreaVisitor`, `SvgExporter`).
- **Element**: interface com `accept(visitor)`.
- **ConcreteElement**: implementa `accept` chamando `visitor.visitX(this)`. Isso é o **double dispatch**: o método executado depende do tipo do elemento **e** do tipo do visitor.

Em linguagens com **tipos soma + pattern matching exaustivo** (Rust, Kotlin, Java 21+, Scala, Python `match`), o Visitor costuma ser substituído por um simples `match` sobre o tipo, com o compilador garantindo que todos os casos foram tratados.

### Trade-offs
| Prós | Contras |
|------|---------|
| Novas **operações** sem tocar nas classes dos elementos (Open/Closed para operações). | Novos **tipos de elemento** obrigam a mudar todos os visitors (o *expression problem*). |
| Agrupa a lógica de uma operação em um só lugar. | Elementos precisam expor estado para os visitors, enfraquecendo o encapsulamento. |
| Visitors podem acumular estado durante a travessia. | Double dispatch com `accept` é verboso e pouco intuitivo. |
| Ótimo com Composite (ASTs, árvores de documentos). | Pattern matching moderno costuma resolver o mesmo com menos código. |

### Quando usar / evitar
- **Use** quando a hierarquia de elementos é **estável** e as **operações mudam com frequência**: compiladores (ASTs), linters, exportadores, analisadores de documentos.
- **Evite** quando novos tipos de elemento aparecem com frequência, ou quando a linguagem oferece tipos selados com `match` exaustivo.

---

## English

### Intent
Represent an **operation to be performed on the elements of an object structure**. Visitor lets you define a new operation **without changing the classes** of the elements on which it operates.

### Problem
A diagram editor has shapes (`Circle`, `Rectangle`) and needs several operations: compute area, export to SVG, export to JSON, validate. Putting each operation as a method on every shape mixes responsibilities (geometry + serialization + validation), and each new operation requires editing every class.

### Structure
- **Visitor**: an interface with one `visit` method per concrete element type (`visitCircle`, `visitRectangle`).
- **ConcreteVisitor**: implements one operation for all types (`AreaVisitor`, `SvgExporter`).
- **Element**: an interface with `accept(visitor)`.
- **ConcreteElement**: implements `accept` by calling `visitor.visitX(this)`. This is **double dispatch**: the method executed depends on the element's type **and** the visitor's type.

In languages with **sum types + exhaustive pattern matching** (Rust, Kotlin, Java 21+, Scala, Python `match`), Visitor is often replaced by a simple `match` on the type, with the compiler ensuring every case is handled.

### Trade-offs
| Pros | Cons |
|------|------|
| New **operations** without touching element classes (Open/Closed for operations). | New **element types** force changing every visitor (the *expression problem*). |
| Groups one operation's logic in one place. | Elements must expose state to visitors, weakening encapsulation. |
| Visitors can accumulate state during traversal. | Double dispatch with `accept` is verbose and unintuitive. |
| Great with Composite (ASTs, document trees). | Modern pattern matching usually achieves the same with less code. |

### When to use / avoid
- **Use** when the element hierarchy is **stable** and **operations change often**: compilers (ASTs), linters, exporters, document analyzers.
- **Avoid** when new element types appear often, or when the language offers sealed types with exhaustive `match`.

---

## Exemplos / Examples

Scenario: shapes with two operations, `area` and `toSvg`, implemented as visitors (or pattern matching).

### Java
```java
import java.util.List;

// Classic double dispatch.
interface ShapeVisitor<R> {
    R visitCircle(Circle c);
    R visitRectangle(Rectangle r);
}

sealed interface Shape permits Circle, Rectangle {
    <R> R accept(ShapeVisitor<R> v);
}

record Circle(double radius) implements Shape {
    public <R> R accept(ShapeVisitor<R> v) { return v.visitCircle(this); }
}

record Rectangle(double w, double h) implements Shape {
    public <R> R accept(ShapeVisitor<R> v) { return v.visitRectangle(this); }
}

class AreaVisitor implements ShapeVisitor<Double> {
    public Double visitCircle(Circle c) { return Math.PI * c.radius() * c.radius(); }
    public Double visitRectangle(Rectangle r) { return r.w() * r.h(); }
}

class Main {
    // Java 21+: pattern matching over a sealed hierarchy replaces the visitor boilerplate.
    static String toSvg(Shape s) {
        return switch (s) {
            case Circle c -> "<circle r=\"" + c.radius() + "\"/>";
            case Rectangle r -> "<rect width=\"" + r.w() + "\" height=\"" + r.h() + "\"/>";
        }; // exhaustive: adding a new Shape makes this fail to compile
    }

    public static void main(String[] args) {
        List<Shape> shapes = List.of(new Circle(1), new Rectangle(2, 3));
        double total = shapes.stream().mapToDouble(s -> s.accept(new AreaVisitor())).sum();
        System.out.printf("total area = %.2f%n", total);
        shapes.forEach(s -> System.out.println(toSvg(s)));
    }
}
```

### Kotlin
```kotlin
import kotlin.math.PI

sealed interface Shape
data class Circle(val radius: Double) : Shape
data class Rectangle(val w: Double, val h: Double) : Shape

// Each "visitor" is just a function with an exhaustive `when`.
fun Shape.area(): Double = when (this) {
    is Circle -> PI * radius * radius
    is Rectangle -> w * h
}

fun Shape.toSvg(): String = when (this) {
    is Circle -> """<circle r="$radius"/>"""
    is Rectangle -> """<rect width="$w" height="$h"/>"""
}

fun main() {
    val shapes = listOf(Circle(1.0), Rectangle(2.0, 3.0))
    println("total area = %.2f".format(shapes.sumOf { it.area() }))
    shapes.forEach { println(it.toSvg()) }
}
```

### Go
```go
package main

import (
	"fmt"
	"math"
)

type Shape interface {
	Accept(v ShapeVisitor)
}

type ShapeVisitor interface {
	VisitCircle(c Circle)
	VisitRectangle(r Rectangle)
}

type Circle struct{ Radius float64 }
type Rectangle struct{ W, H float64 }

func (c Circle) Accept(v ShapeVisitor)    { v.VisitCircle(c) }
func (r Rectangle) Accept(v ShapeVisitor) { v.VisitRectangle(r) }

// Visitors accumulate results in their own state (Go methods can't be generic).
type AreaVisitor struct{ Total float64 }

func (a *AreaVisitor) VisitCircle(c Circle)       { a.Total += math.Pi * c.Radius * c.Radius }
func (a *AreaVisitor) VisitRectangle(r Rectangle) { a.Total += r.W * r.H }

// Idiomatic alternative: a type switch (not checked for exhaustiveness by the compiler).
func ToSvg(s Shape) string {
	switch s := s.(type) {
	case Circle:
		return fmt.Sprintf(`<circle r="%g"/>`, s.Radius)
	case Rectangle:
		return fmt.Sprintf(`<rect width="%g" height="%g"/>`, s.W, s.H)
	default:
		panic(fmt.Sprintf("unknown shape %T", s))
	}
}

func main() {
	shapes := []Shape{Circle{1}, Rectangle{2, 3}}
	area := &AreaVisitor{}
	for _, s := range shapes {
		s.Accept(area)
		fmt.Println(ToSvg(s))
	}
	fmt.Printf("total area = %.2f\n", area.Total)
}
```

### C++
```cpp
#include <iostream>
#include <numbers>
#include <string>
#include <variant>
#include <vector>

struct Circle { double radius; };
struct Rectangle { double w, h; };
using Shape = std::variant<Circle, Rectangle>;

// "overloaded" helper: build a visitor from lambdas.
template <class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts> overloaded(Ts...) -> overloaded<Ts...>;  // needed in C++17, implicit in C++20

double area(const Shape& s) {
    return std::visit(overloaded{
        [](const Circle& c) { return std::numbers::pi * c.radius * c.radius; },
        [](const Rectangle& r) { return r.w * r.h; },
    }, s);  // compile error if a variant alternative isn't handled
}

std::string toSvg(const Shape& s) {
    return std::visit(overloaded{
        [](const Circle& c) { return "<circle r=\"" + std::to_string(c.radius) + "\"/>"; },
        [](const Rectangle& r) {
            return "<rect width=\"" + std::to_string(r.w) + "\" height=\"" + std::to_string(r.h) + "\"/>";
        },
    }, s);
}

int main() {
    std::vector<Shape> shapes{Circle{1}, Rectangle{2, 3}};
    double total = 0;
    for (const auto& s : shapes) {
        total += area(s);
        std::cout << toSvg(s) << "\n";
    }
    std::cout << "total area = " << total << "\n";
}
```

### Rust
```rust
use std::f64::consts::PI;

enum Shape {
    Circle { radius: f64 },
    Rectangle { w: f64, h: f64 },
}

// Operations are functions over the enum; `match` is exhaustive.
fn area(s: &Shape) -> f64 {
    match s {
        Shape::Circle { radius } => PI * radius * radius,
        Shape::Rectangle { w, h } => w * h,
    }
}

fn to_svg(s: &Shape) -> String {
    match s {
        Shape::Circle { radius } => format!(r#"<circle r="{radius}"/>"#),
        Shape::Rectangle { w, h } => format!(r#"<rect width="{w}" height="{h}"/>"#),
    }
}

// A trait-based visitor is still useful for open traversals (e.g. syn::visit for ASTs).
trait ShapeVisitor {
    fn visit_circle(&mut self, radius: f64);
    fn visit_rectangle(&mut self, w: f64, h: f64);
}

impl Shape {
    fn accept(&self, v: &mut impl ShapeVisitor) {
        match *self {
            Shape::Circle { radius } => v.visit_circle(radius),
            Shape::Rectangle { w, h } => v.visit_rectangle(w, h),
        }
    }
}

struct Counter { circles: u32, rectangles: u32 }
impl ShapeVisitor for Counter {
    fn visit_circle(&mut self, _: f64) { self.circles += 1; }
    fn visit_rectangle(&mut self, _: f64, _: f64) { self.rectangles += 1; }
}

fn main() {
    let shapes = vec![Shape::Circle { radius: 1.0 }, Shape::Rectangle { w: 2.0, h: 3.0 }];
    let total: f64 = shapes.iter().map(area).sum();
    println!("total area = {total:.2}");
    shapes.iter().for_each(|s| println!("{}", to_svg(s)));

    let mut counter = Counter { circles: 0, rectangles: 0 };
    shapes.iter().for_each(|s| s.accept(&mut counter));
    println!("{} circles, {} rectangles", counter.circles, counter.rectangles);
}
```

### Python
```python
import math
from dataclasses import dataclass
from functools import singledispatch


@dataclass(frozen=True)
class Circle:
    radius: float


@dataclass(frozen=True)
class Rectangle:
    w: float
    h: float


Shape = Circle | Rectangle


# Option 1: structural pattern matching.
def area(shape: Shape) -> float:
    match shape:
        case Circle(radius=r):
            return math.pi * r * r
        case Rectangle(w=w, h=h):
            return w * h
    raise TypeError(shape)


# Option 2: singledispatch = external, open-ended double dispatch on the argument type.
@singledispatch
def to_svg(shape: object) -> str:
    raise TypeError(f"unsupported: {type(shape).__name__}")


@to_svg.register
def _(shape: Circle) -> str:
    return f'<circle r="{shape.radius}"/>'


@to_svg.register
def _(shape: Rectangle) -> str:
    return f'<rect width="{shape.w}" height="{shape.h}"/>'


if __name__ == "__main__":
    shapes: list[Shape] = [Circle(1), Rectangle(2, 3)]
    print(f"total area = {sum(area(s) for s in shapes):.2f}")
    for s in shapes:
        print(to_svg(s))
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Classic `accept`/`visit` (e.g. `javax.lang.model`, `FileVisitor`); Java 21 sealed types + `switch` patterns often replace it. |
| Kotlin | Sealed classes + exhaustive `when` as extension functions. |
| Go | Visitor interfaces (`go/ast.Visitor`, `ast.Walk`) or type switches (no exhaustiveness check). |
| C++ | `std::variant` + `std::visit` with the `overloaded` idiom; classic double dispatch for open hierarchies. |
| Rust | Enums + `match`; trait-based visitors for large ASTs (`syn::visit`, `serde::de::Visitor`). |
| Python | `match` statements or `functools.singledispatch`; `ast.NodeVisitor` in the standard library. |

### Referências / References
- *Design Patterns* (GoF), Visitor
- Philip Wadler, [The Expression Problem](https://homepages.inf.ed.ac.uk/wadler/papers/expression/expression.txt)
- [Refactoring.Guru: Visitor](https://refactoring.guru/design-patterns/visitor)
