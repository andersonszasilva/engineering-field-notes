# Mediator

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / behavioral
**Tags:** gof, design-patterns, behavioral, mediator, decoupling, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Definir um objeto que **encapsula como um conjunto de objetos interage**. O Mediator promove baixo acoplamento evitando que os objetos se refiram uns aos outros explicitamente.

### Problema
Numa sala de chat, cada usuário precisaria conhecer todos os outros para enviar mensagens: N × (N-1) referências. Em um formulário de UI, mudar o campo "país" habilita o "estado", que limpa a "cidade", que desabilita o botão... Componentes que se chamam diretamente formam uma teia difícil de mudar.

### Estrutura
- **Mediator**: interface de comunicação (`send(from, message)`).
- **ConcreteMediator**: conhece os colegas e coordena as interações.
- **Colleagues**: conhecem **apenas** o mediator e o notificam sobre eventos.

Muitos-para-muitos vira um-para-muitos (estrela). Parentes próximos: *event bus*, controller de UI, orquestrador de *sagas*, torre de controle de aeroporto.

Mediator vs Observer: o Observer distribui notificações de um sujeito para muitos observadores sem lógica de coordenação; o Mediator **concentra a lógica** de como os colegas interagem.

### Trade-offs
| Prós | Contras |
|------|---------|
| Colegas desacoplados entre si; mais fáceis de reutilizar e testar. | O mediator tende a virar um **god object** com toda a lógica de interação. |
| Lógica de coordenação centralizada e explícita. | Ponto único de falha/gargalo (inclusive em escala, num *message broker*). |
| Reduz relações N×N para N×1. | Fluxo indireto: mais difícil entender quem reage a quê. |
| Fácil mudar a interação trocando o mediator. | |

### Quando usar / evitar
- **Use** para chats, diálogos/formulários com campos interdependentes, coordenação de workflows (orquestração de sagas) e comunicação entre módulos.
- **Evite** quando poucas classes interagem de forma simples; a indireção não compensa.

---

## English

### Intent
Define an object that **encapsulates how a set of objects interact**. Mediator promotes loose coupling by keeping objects from referring to each other explicitly.

### Problem
In a chat room, each user would need to know every other user to send messages: N × (N-1) references. In a UI form, changing the "country" field enables "state", which clears "city", which disables the button... Components calling each other directly form a web that's hard to change.

### Structure
- **Mediator**: the communication interface (`send(from, message)`).
- **ConcreteMediator**: knows the colleagues and coordinates interactions.
- **Colleagues**: know **only** the mediator and notify it about events.

Many-to-many becomes one-to-many (a star). Close relatives: *event bus*, UI controller, *saga* orchestrator, an airport control tower.

Mediator vs Observer: Observer broadcasts notifications from one subject to many observers with no coordination logic; Mediator **centralizes the logic** of how colleagues interact.

### Trade-offs
| Pros | Cons |
|------|------|
| Colleagues are decoupled from each other; easier to reuse and test. | The mediator tends to become a **god object** holding all interaction logic. |
| Coordination logic is centralized and explicit. | Single point of failure/bottleneck (also at scale, in a *message broker*). |
| Reduces N×N relationships to N×1. | Indirect flow: harder to see who reacts to what. |
| Easy to change the interaction by swapping the mediator. | |

### When to use / avoid
- **Use** for chats, dialogs/forms with interdependent fields, workflow coordination (saga orchestration), and inter-module communication.
- **Avoid** when a few classes interact simply; the indirection doesn't pay off.

---

## Exemplos / Examples

Scenario: a chat room mediates messages between users; users never reference each other.

### Java
```java
import java.util.ArrayList;
import java.util.List;

interface ChatMediator {
    void join(User user);
    void send(User from, String message);
}

class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    public void join(User user) { users.add(user); }

    public void send(User from, String message) {
        // Coordination logic lives here: routing, filtering, muting, logging...
        if (message.isBlank()) return;
        for (User u : users) {
            if (u != from) u.receive(from.name(), message);
        }
    }
}

class User {
    private final String name;
    private final ChatMediator room;

    User(String name, ChatMediator room) {
        this.name = name;
        this.room = room;
        room.join(this);
    }

    String name() { return name; }
    void say(String message) { room.send(this, message); }
    void receive(String from, String message) { System.out.println("[" + name + "] " + from + ": " + message); }
}

class Main {
    public static void main(String[] args) {
        ChatMediator room = new ChatRoom();
        User ana = new User("ana", room);
        new User("bruno", room);
        new User("carla", room);
        ana.say("standup in 5 min");
    }
}
```

### Kotlin
```kotlin
interface ChatMediator {
    fun join(user: User)
    fun send(from: User, message: String)
}

class ChatRoom : ChatMediator {
    private val users = mutableListOf<User>()
    override fun join(user: User) { users += user }
    override fun send(from: User, message: String) {
        if (message.isBlank()) return
        users.filter { it !== from }.forEach { it.receive(from.name, message) }
    }
}

class User(val name: String, private val room: ChatMediator) {
    init { room.join(this) }
    fun say(message: String) = room.send(this, message)
    fun receive(from: String, message: String) = println("[$name] $from: $message")
}

fun main() {
    val room = ChatRoom()
    val ana = User("ana", room)
    User("bruno", room)
    User("carla", room)
    ana.say("standup in 5 min")
}
```

### Go
```go
package main

import "fmt"

type Message struct {
	From, Text string
}

// Mediator implemented with channels: users only know the room's inbox.
type ChatRoom struct {
	inbox   chan Message
	members map[string]chan Message
}

func NewChatRoom() *ChatRoom {
	return &ChatRoom{inbox: make(chan Message), members: map[string]chan Message{}}
}

func (r *ChatRoom) Join(name string) chan Message {
	ch := make(chan Message, 10)
	r.members[name] = ch
	return ch
}

// Run is the coordination loop; it is the only place that knows every member.
func (r *ChatRoom) Run() {
	for msg := range r.inbox {
		for name, ch := range r.members {
			if name != msg.From {
				ch <- msg
			}
		}
	}
}

func main() {
	room := NewChatRoom()
	bruno := room.Join("bruno")
	carla := room.Join("carla")
	room.Join("ana")
	go room.Run()

	room.inbox <- Message{From: "ana", Text: "standup in 5 min"}
	fmt.Printf("[bruno] %+v\n", <-bruno)
	fmt.Printf("[carla] %+v\n", <-carla)
}
```

### C++
```cpp
#include <iostream>
#include <string>
#include <vector>

class User;

struct ChatMediator {
    virtual ~ChatMediator() = default;
    virtual void join(User* user) = 0;
    virtual void send(const User& from, const std::string& msg) = 0;
};

class User {
public:
    User(std::string name, ChatMediator& room) : name_(std::move(name)), room_(room) { room_.join(this); }
    const std::string& name() const { return name_; }
    void say(const std::string& msg) { room_.send(*this, msg); }
    void receive(const std::string& from, const std::string& msg) const {
        std::cout << "[" << name_ << "] " << from << ": " << msg << "\n";
    }
private:
    std::string name_;
    ChatMediator& room_;
};

class ChatRoom : public ChatMediator {
public:
    void join(User* user) override { users_.push_back(user); }  // non-owning pointers
    void send(const User& from, const std::string& msg) override {
        if (msg.empty()) return;
        for (const User* u : users_)
            if (u != &from) u->receive(from.name(), msg);
    }
private:
    std::vector<User*> users_;
};

int main() {
    ChatRoom room;
    User ana("ana", room), bruno("bruno", room), carla("carla", room);
    ana.say("standup in 5 min");
}
```

### Rust
```rust
// Rust discourages cyclic references (user -> room -> user).
// Idiomatic version: the mediator owns the colleagues and users are addressed by id.
struct User {
    name: String,
    inbox: Vec<String>,
}

#[derive(Default)]
struct ChatRoom {
    users: Vec<User>,
}

type UserId = usize;

impl ChatRoom {
    fn join(&mut self, name: &str) -> UserId {
        self.users.push(User { name: name.into(), inbox: Vec::new() });
        self.users.len() - 1
    }

    fn send(&mut self, from: UserId, message: &str) {
        if message.trim().is_empty() {
            return;
        }
        let sender = self.users[from].name.clone();
        for (id, user) in self.users.iter_mut().enumerate() {
            if id != from {
                user.inbox.push(format!("{sender}: {message}"));
            }
        }
    }
}

fn main() {
    let mut room = ChatRoom::default();
    let ana = room.join("ana");
    room.join("bruno");
    room.join("carla");
    room.send(ana, "standup in 5 min");

    for u in &room.users {
        println!("[{}] {:?}", u.name, u.inbox);
    }
}
```

### Python
```python
from __future__ import annotations


class ChatRoom:
    def __init__(self) -> None:
        self._users: list[User] = []

    def join(self, user: User) -> None:
        self._users.append(user)

    def send(self, sender: User, message: str) -> None:
        if not message.strip():
            return
        for user in self._users:
            if user is not sender:
                user.receive(sender.name, message)


class User:
    def __init__(self, name: str, room: ChatRoom) -> None:
        self.name = name
        self._room = room
        room.join(self)

    def say(self, message: str) -> None:
        self._room.send(self, message)

    def receive(self, sender: str, message: str) -> None:
        print(f"[{self.name}] {sender}: {message}")


if __name__ == "__main__":
    room = ChatRoom()
    ana = User("ana", room)
    User("bruno", room)
    User("carla", room)
    ana.say("standup in 5 min")
```

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | MediatR-style command buses, Spring `ApplicationEventPublisher`, `java.util.concurrent.Executor` coordinating tasks. |
| Kotlin | Coroutines + `Channel`/`SharedFlow` as an in-process mediator. |
| Go | A goroutine owning state and talking through channels is a natural mediator ("share memory by communicating"). |
| C++ | Non-owning pointers/references to colleagues; watch lifetimes (dangling pointers). |
| Rust | Avoid reference cycles: the mediator owns colleagues and uses ids/handles, or uses channels (`mpsc`). |
| Python | Simple object references; `asyncio.Queue` for async mediators. |

### Referências / References
- *Design Patterns* (GoF), Mediator
- Chris Richardson, [Saga pattern: orchestration](https://microservices.io/patterns/data/saga.html)
- [Refactoring.Guru: Mediator](https://refactoring.guru/design-patterns/mediator)
