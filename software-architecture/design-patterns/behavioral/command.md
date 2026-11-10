# Command

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, command, undo, cqrs, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
**Encapsular uma requisição como um objeto**, permitindo parametrizar clientes com diferentes requisições, enfileirá-las, registrá-las em log e suportar **desfazer** (undo).

### Problema
Um editor precisa de undo/redo. Um sistema de jobs precisa enfileirar operações para executar depois, com retry. Uma API precisa registrar cada operação para auditoria. Se as operações são só chamadas de método, não há "coisa" para guardar, enfileirar ou desfazer.

### Estrutura
- **Command**: interface com `execute()` (e opcionalmente `undo()`).
- **ConcreteCommand**: guarda os parâmetros e a referência ao **Receiver**, e implementa a ação.
- **Receiver**: objeto que realmente faz o trabalho (`Document`, `Account`).
- **Invoker**: dispara comandos sem saber o que fazem (botão, fila, scheduler); pode manter histórico.
- **Client**: cria os comandos e os entrega ao invoker.

Conceitos relacionados: *commands* em CQRS, mensagens em filas (o comando serializado), *event sourcing* (log de operações), *transaction scripts*.

### Trade-offs
| Prós | Contras |
|------|---------|
| Undo/redo, filas, agendamento, retry e auditoria ficam naturais. | Uma classe por operação: muito boilerplate em linguagens sem lambdas. |
| Desacopla quem dispara de quem executa. | Undo correto é difícil quando há efeitos externos (e-mail enviado não volta). |
| Comandos compostos (macros) e transações. | Comandos serializados exigem versionamento de esquema. |
| Facilita testes: comando é um valor inspecionável. | Indireção: seguir o fluxo "quem executa isto?" fica mais difícil. |

### Quando usar / evitar
- **Use** para undo/redo, filas de trabalho, jobs agendados, macros, CQRS e logs de auditoria.
- **Evite** quando uma simples função/callback resolve; em linguagens com closures, um lambda já é um command sem undo.

---

## English

### Intent
**Encapsulate a request as an object**, letting you parameterize clients with different requests, queue them, log them, and support **undo**.

### Problem
An editor needs undo/redo. A job system needs to queue operations to run later, with retries. An API needs to record every operation for auditing. If operations are just method calls, there is no "thing" to store, queue, or undo.

### Structure
- **Command**: interface with `execute()` (and optionally `undo()`).
- **ConcreteCommand**: stores parameters and a reference to the **Receiver**, and implements the action.
- **Receiver**: the object that actually does the work (`Document`, `Account`).
- **Invoker**: triggers commands without knowing what they do (button, queue, scheduler); may keep history.
- **Client**: creates commands and hands them to the invoker.

Related ideas: *commands* in CQRS, messages in queues (the serialized command), *event sourcing* (an operation log), *transaction scripts*.

### Trade-offs
| Pros | Cons |
|------|------|
| Undo/redo, queues, scheduling, retries, and auditing become natural. | One class per operation: lots of boilerplate in languages without lambdas. |
| Decouples the trigger from the executor. | Correct undo is hard with external side effects (a sent email can't be unsent). |
| Composite commands (macros) and transactions. | Serialized commands require schema versioning. |
| Eases testing: a command is an inspectable value. | Indirection: following "who runs this?" gets harder. |

### When to use / avoid
- **Use** for undo/redo, work queues, scheduled jobs, macros, CQRS, and audit logs.
- **Avoid** when a plain function/callback does the job; in languages with closures, a lambda is already a command without undo.

---

## Exemplos / Examples

Scenario: a text editor with `Insert` and `Delete` commands and an undo history.

<details>
<summary><strong>Java</strong></summary>

```java
import java.util.ArrayDeque;
import java.util.Deque;

// Receiver
class Document {
    final StringBuilder text = new StringBuilder();
}

interface Command {
    void execute();
    void undo();
}

record InsertCommand(Document doc, int pos, String value) implements Command {
    public void execute() { doc.text.insert(pos, value); }
    public void undo() { doc.text.delete(pos, pos + value.length()); }
}

final class DeleteCommand implements Command {
    private final Document doc;
    private final int start, end;
    private String removed = "";

    DeleteCommand(Document doc, int start, int end) { this.doc = doc; this.start = start; this.end = end; }

    public void execute() {
        removed = doc.text.substring(start, end); // remember state needed for undo
        doc.text.delete(start, end);
    }
    public void undo() { doc.text.insert(start, removed); }
}

// Invoker with history
class Editor {
    private final Deque<Command> history = new ArrayDeque<>();
    void run(Command c) { c.execute(); history.push(c); }
    void undo() { if (!history.isEmpty()) history.pop().undo(); }
}

class Main {
    public static void main(String[] args) {
        var doc = new Document();
        var editor = new Editor();
        editor.run(new InsertCommand(doc, 0, "Hello world"));
        editor.run(new DeleteCommand(doc, 5, 11));
        System.out.println(doc.text); // Hello
        editor.undo();
        System.out.println(doc.text); // Hello world
        editor.undo();
        System.out.println("[" + doc.text + "]"); // []
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
class Document { val text = StringBuilder() }

// Sealed commands are plain data: easy to log, serialize, and test.
sealed interface Command {
    fun execute(doc: Document)
    fun undo(doc: Document)
}

data class Insert(val pos: Int, val value: String) : Command {
    override fun execute(doc: Document) { doc.text.insert(pos, value) }
    override fun undo(doc: Document) { doc.text.delete(pos, pos + value.length) }
}

class Delete(private val start: Int, private val end: Int) : Command {
    private var removed = ""
    override fun execute(doc: Document) {
        removed = doc.text.substring(start, end)
        doc.text.delete(start, end)
    }
    override fun undo(doc: Document) { doc.text.insert(start, removed) }
}

class Editor(private val doc: Document) {
    private val history = ArrayDeque<Command>()
    fun run(cmd: Command) { cmd.execute(doc); history.addLast(cmd) }
    fun undo() { history.removeLastOrNull()?.undo(doc) }
}

fun main() {
    val doc = Document()
    val editor = Editor(doc)
    editor.run(Insert(0, "Hello world"))
    editor.run(Delete(5, 11))
    println(doc.text)
    editor.undo()
    println(doc.text)
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import "fmt"

type Document struct{ Text string }

type Command interface {
	Execute(d *Document)
	Undo(d *Document)
}

type Insert struct {
	Pos   int
	Value string
}

func (c Insert) Execute(d *Document) { d.Text = d.Text[:c.Pos] + c.Value + d.Text[c.Pos:] }
func (c Insert) Undo(d *Document)    { d.Text = d.Text[:c.Pos] + d.Text[c.Pos+len(c.Value):] }

type Delete struct {
	Start, End int
	removed    string
}

// Pointer receiver: Execute stores state that Undo needs.
func (c *Delete) Execute(d *Document) {
	c.removed = d.Text[c.Start:c.End]
	d.Text = d.Text[:c.Start] + d.Text[c.End:]
}
func (c *Delete) Undo(d *Document) { d.Text = d.Text[:c.Start] + c.removed + d.Text[c.Start:] }

type Editor struct {
	doc     *Document
	history []Command
}

func (e *Editor) Run(c Command) {
	c.Execute(e.doc)
	e.history = append(e.history, c)
}

func (e *Editor) Undo() {
	if n := len(e.history); n > 0 {
		e.history[n-1].Undo(e.doc)
		e.history = e.history[:n-1]
	}
}

func main() {
	doc := &Document{}
	ed := &Editor{doc: doc}
	ed.Run(Insert{Pos: 0, Value: "Hello world"})
	ed.Run(&Delete{Start: 5, End: 11})
	fmt.Println(doc.Text)
	ed.Undo()
	fmt.Println(doc.Text)
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Document { std::string text; };

struct Command {
    virtual ~Command() = default;
    virtual void execute(Document& d) = 0;
    virtual void undo(Document& d) = 0;
};

class Insert : public Command {
public:
    Insert(std::size_t pos, std::string v) : pos_(pos), value_(std::move(v)) {}
    void execute(Document& d) override { d.text.insert(pos_, value_); }
    void undo(Document& d) override { d.text.erase(pos_, value_.size()); }
private:
    std::size_t pos_;
    std::string value_;
};

class Delete : public Command {
public:
    Delete(std::size_t start, std::size_t end) : start_(start), end_(end) {}
    void execute(Document& d) override {
        removed_ = d.text.substr(start_, end_ - start_);
        d.text.erase(start_, end_ - start_);
    }
    void undo(Document& d) override { d.text.insert(start_, removed_); }
private:
    std::size_t start_, end_;
    std::string removed_;
};

class Editor {
public:
    explicit Editor(Document& d) : doc_(d) {}
    void run(std::unique_ptr<Command> c) { c->execute(doc_); history_.push_back(std::move(c)); }
    void undo() {
        if (history_.empty()) return;
        history_.back()->undo(doc_);
        history_.pop_back();
    }
private:
    Document& doc_;
    std::vector<std::unique_ptr<Command>> history_;
};

int main() {
    Document doc;
    Editor editor(doc);
    editor.run(std::make_unique<Insert>(0, "Hello world"));
    editor.run(std::make_unique<Delete>(5, 11));
    std::cout << doc.text << "\n";  // Hello
    editor.undo();
    std::cout << doc.text << "\n";  // Hello world
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
// Commands as an enum: data-only, serializable (e.g. with serde), exhaustive matching.
#[derive(Debug)]
enum Command {
    Insert { pos: usize, value: String },
    Delete { start: usize, end: usize, removed: Option<String> },
}

struct Editor {
    text: String,
    history: Vec<Command>,
}

impl Editor {
    fn run(&mut self, mut cmd: Command) {
        match &mut cmd {
            Command::Insert { pos, value } => self.text.insert_str(*pos, value),
            Command::Delete { start, end, removed } => {
                *removed = Some(self.text[*start..*end].to_string());
                self.text.replace_range(*start..*end, "");
            }
        }
        self.history.push(cmd);
    }

    fn undo(&mut self) {
        match self.history.pop() {
            Some(Command::Insert { pos, value }) => self.text.replace_range(pos..pos + value.len(), ""),
            Some(Command::Delete { start, removed: Some(r), .. }) => self.text.insert_str(start, &r),
            _ => {}
        }
    }
}

fn main() {
    let mut editor = Editor { text: String::new(), history: Vec::new() };
    editor.run(Command::Insert { pos: 0, value: "Hello world".into() });
    editor.run(Command::Delete { start: 5, end: 11, removed: None });
    println!("{}", editor.text);
    editor.undo();
    println!("{}", editor.text);
    println!("{:?}", editor.history);
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
from dataclasses import dataclass, field
from typing import Protocol


class Document:
    def __init__(self) -> None:
        self.text = ""


class Command(Protocol):
    def execute(self, doc: Document) -> None: ...
    def undo(self, doc: Document) -> None: ...


@dataclass
class Insert:
    pos: int
    value: str

    def execute(self, doc: Document) -> None:
        doc.text = doc.text[: self.pos] + self.value + doc.text[self.pos :]

    def undo(self, doc: Document) -> None:
        doc.text = doc.text[: self.pos] + doc.text[self.pos + len(self.value) :]


@dataclass
class Delete:
    start: int
    end: int
    removed: str = field(default="", init=False)

    def execute(self, doc: Document) -> None:
        self.removed = doc.text[self.start : self.end]
        doc.text = doc.text[: self.start] + doc.text[self.end :]

    def undo(self, doc: Document) -> None:
        doc.text = doc.text[: self.start] + self.removed + doc.text[self.start :]


class Editor:
    def __init__(self, doc: Document) -> None:
        self.doc = doc
        self.history: list[Command] = []

    def run(self, cmd: Command) -> None:
        cmd.execute(self.doc)
        self.history.append(cmd)

    def undo(self) -> None:
        if self.history:
            self.history.pop().undo(self.doc)


if __name__ == "__main__":
    doc = Document()
    editor = Editor(doc)
    editor.run(Insert(0, "Hello world"))
    editor.run(Delete(5, 11))
    print(doc.text)  # Hello
    editor.undo()
    print(doc.text)  # Hello world
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | `Runnable`/`Callable` submitted to an `ExecutorService` are commands; records make immutable command objects cheap. |
| Kotlin | Sealed interfaces + data classes model commands as data; `() -> Unit` when undo isn't needed. |
| Go | `func()` values or small structs implementing an interface; commands serialized to queues (Kafka, SQS). |
| C++ | `std::function<void()>` for simple commands; classes when undo/state is needed. |
| Rust | Enums are ideal: data-only, `serde`-serializable, exhaustive `match` for execute/undo. |
| Python | Functions/`functools.partial` as commands; dataclasses when undo/serialization matter. |

### Referências / References
- *Design Patterns* (GoF), Command
- Martin Fowler, [CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Refactoring.Guru: Command](https://refactoring.guru/design-patterns/command)
