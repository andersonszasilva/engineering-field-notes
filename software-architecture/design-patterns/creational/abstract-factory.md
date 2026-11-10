# Abstract Factory

**Data:** 2026-11-10
**Área:** software-architecture / design-patterns / creational
**Tags:** gof, design-patterns, creational, abstract-factory, trade-offs

[Português](#português) | [English](#english) | [Exemplos / Examples](#exemplos--examples)

---

## Português

### Intenção
Fornecer uma interface para criar **famílias de objetos relacionados** sem especificar suas classes concretas.

### Problema
Uma aplicação roda em mais de um provedor de nuvem e precisa de um *storage* de objetos e de uma *fila*. Misturar um storage da AWS com uma fila do GCP no mesmo ambiente é um bug. O código de negócio não deveria saber qual provedor está em uso, mas os objetos criados precisam ser **consistentes entre si**.

### Estrutura
- **AbstractFactory**: interface com um método de criação por produto (`createStorage()`, `createQueue()`).
- **ConcreteFactory**: uma por família (`AwsFactory`, `GcpFactory`).
- **AbstractProduct / ConcreteProduct**: interfaces e implementações de cada produto.
- **Client**: recebe uma fábrica e só conhece as interfaces.

Diferença para o Factory Method: o Factory Method cria **um** produto; a Abstract Factory agrupa **vários** métodos fábrica que precisam produzir objetos compatíveis.

### Trade-offs
| Prós | Contras |
|------|---------|
| Garante consistência entre produtos da mesma família. | Adicionar um **novo tipo de produto** obriga a mudar a interface e todas as fábricas. |
| Troca a família inteira em um ponto só (config, ambiente, testes). | Muitas interfaces e classes; pode parecer burocrático. |
| Isola o código cliente dos SDKs de cada fornecedor. | Abstrair o mínimo denominador comum pode esconder recursos úteis de cada provedor. |
| Facilita testes com uma fábrica in-memory. | Com uma só família real, é abstração especulativa. |

### Quando usar / evitar
- **Use** quando há famílias de objetos que precisam combinar (provedores de nuvem, temas de UI, drivers de banco, ambientes prod/teste).
- **Evite** quando os produtos variam de forma independente; nesse caso, fábricas separadas (ou injeção direta) são mais simples.

---

## English

### Intent
Provide an interface for creating **families of related objects** without specifying their concrete classes.

### Problem
An application runs on more than one cloud provider and needs an object *storage* and a *queue*. Mixing AWS storage with a GCP queue in the same environment is a bug. Business code shouldn't know which provider is in use, but the created objects must be **consistent with each other**.

### Structure
- **AbstractFactory**: interface with one creation method per product (`createStorage()`, `createQueue()`).
- **ConcreteFactory**: one per family (`AwsFactory`, `GcpFactory`).
- **AbstractProduct / ConcreteProduct**: interfaces and implementations of each product.
- **Client**: receives a factory and only knows the interfaces.

Difference from Factory Method: Factory Method creates **one** product; Abstract Factory groups **several** factory methods that must produce compatible objects.

### Trade-offs
| Pros | Cons |
|------|------|
| Guarantees consistency among products of the same family. | Adding a **new product type** requires changing the interface and every factory. |
| Swaps the whole family in one place (config, environment, tests). | Many interfaces and classes; can feel bureaucratic. |
| Isolates client code from each vendor's SDK. | Abstracting to the lowest common denominator can hide useful provider features. |
| Eases testing with an in-memory factory. | With a single real family, it's speculative abstraction. |

### When to use / avoid
- **Use** when there are families of objects that must match (cloud providers, UI themes, database drivers, prod/test environments).
- **Avoid** when products vary independently; separate factories (or direct injection) are simpler then.

---

## Exemplos / Examples

Scenario: a cloud factory creates a matching `BlobStorage` and `Queue` for AWS or GCP.

<details>
<summary><strong>Java</strong></summary>

```java
interface BlobStorage { void put(String key, byte[] data); }
interface Queue { void publish(String message); }

interface CloudFactory {
    BlobStorage createStorage();
    Queue createQueue();
}

class S3Storage implements BlobStorage {
    public void put(String key, byte[] data) { System.out.println("S3 put " + key); }
}
class SqsQueue implements Queue {
    public void publish(String message) { System.out.println("SQS publish " + message); }
}
class GcsStorage implements BlobStorage {
    public void put(String key, byte[] data) { System.out.println("GCS put " + key); }
}
class PubSubQueue implements Queue {
    public void publish(String message) { System.out.println("Pub/Sub publish " + message); }
}

class AwsFactory implements CloudFactory {
    public BlobStorage createStorage() { return new S3Storage(); }
    public Queue createQueue() { return new SqsQueue(); }
}
class GcpFactory implements CloudFactory {
    public BlobStorage createStorage() { return new GcsStorage(); }
    public Queue createQueue() { return new PubSubQueue(); }
}

// The client only knows the abstract types.
class ReportExporter {
    private final BlobStorage storage;
    private final Queue queue;

    ReportExporter(CloudFactory factory) {
        this.storage = factory.createStorage();
        this.queue = factory.createQueue();
    }

    void export(String id) {
        storage.put("reports/" + id, new byte[0]);
        queue.publish("report-ready:" + id);
    }
}

class Main {
    public static void main(String[] args) {
        CloudFactory factory = "gcp".equals(System.getenv("CLOUD")) ? new GcpFactory() : new AwsFactory();
        new ReportExporter(factory).export("42");
    }
}
```

</details>

<details>
<summary><strong>Kotlin</strong></summary>

```kotlin
interface BlobStorage { fun put(key: String, data: ByteArray) }
interface Queue { fun publish(message: String) }

interface CloudFactory {
    fun createStorage(): BlobStorage
    fun createQueue(): Queue
}

object AwsFactory : CloudFactory {
    override fun createStorage() = object : BlobStorage {
        override fun put(key: String, data: ByteArray) = println("S3 put $key")
    }
    override fun createQueue() = object : Queue {
        override fun publish(message: String) = println("SQS publish $message")
    }
}

object GcpFactory : CloudFactory {
    override fun createStorage() = object : BlobStorage {
        override fun put(key: String, data: ByteArray) = println("GCS put $key")
    }
    override fun createQueue() = object : Queue {
        override fun publish(message: String) = println("Pub/Sub publish $message")
    }
}

class ReportExporter(factory: CloudFactory) {
    private val storage = factory.createStorage()
    private val queue = factory.createQueue()

    fun export(id: String) {
        storage.put("reports/$id", ByteArray(0))
        queue.publish("report-ready:$id")
    }
}

fun main() {
    val factory: CloudFactory = if (System.getenv("CLOUD") == "gcp") GcpFactory else AwsFactory
    ReportExporter(factory).export("42")
}
```

</details>

<details>
<summary><strong>Go</strong></summary>

```go
package main

import (
	"fmt"
	"os"
)

type BlobStorage interface{ Put(key string, data []byte) }
type Queue interface{ Publish(msg string) }

type CloudFactory interface {
	NewStorage() BlobStorage
	NewQueue() Queue
}

type s3Storage struct{}

func (s3Storage) Put(key string, _ []byte) { fmt.Println("S3 put", key) }

type sqsQueue struct{}

func (sqsQueue) Publish(msg string) { fmt.Println("SQS publish", msg) }

type gcsStorage struct{}

func (gcsStorage) Put(key string, _ []byte) { fmt.Println("GCS put", key) }

type pubSubQueue struct{}

func (pubSubQueue) Publish(msg string) { fmt.Println("Pub/Sub publish", msg) }

type AwsFactory struct{}

func (AwsFactory) NewStorage() BlobStorage { return s3Storage{} }
func (AwsFactory) NewQueue() Queue         { return sqsQueue{} }

type GcpFactory struct{}

func (GcpFactory) NewStorage() BlobStorage { return gcsStorage{} }
func (GcpFactory) NewQueue() Queue         { return pubSubQueue{} }

func ExportReport(f CloudFactory, id string) {
	f.NewStorage().Put("reports/"+id, nil)
	f.NewQueue().Publish("report-ready:" + id)
}

func main() {
	var f CloudFactory = AwsFactory{}
	if os.Getenv("CLOUD") == "gcp" {
		f = GcpFactory{}
	}
	ExportReport(f, "42")
}
```

</details>

<details>
<summary><strong>C++</strong></summary>

```cpp
#include <cstdlib>
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct BlobStorage {
    virtual ~BlobStorage() = default;
    virtual void put(const std::string& key, const std::vector<char>& data) = 0;
};
struct Queue {
    virtual ~Queue() = default;
    virtual void publish(const std::string& msg) = 0;
};

struct CloudFactory {
    virtual ~CloudFactory() = default;
    virtual std::unique_ptr<BlobStorage> createStorage() const = 0;
    virtual std::unique_ptr<Queue> createQueue() const = 0;
};

struct S3Storage : BlobStorage {
    void put(const std::string& key, const std::vector<char>&) override { std::cout << "S3 put " << key << "\n"; }
};
struct SqsQueue : Queue {
    void publish(const std::string& msg) override { std::cout << "SQS publish " << msg << "\n"; }
};
struct GcsStorage : BlobStorage {
    void put(const std::string& key, const std::vector<char>&) override { std::cout << "GCS put " << key << "\n"; }
};
struct PubSubQueue : Queue {
    void publish(const std::string& msg) override { std::cout << "Pub/Sub publish " << msg << "\n"; }
};

struct AwsFactory : CloudFactory {
    std::unique_ptr<BlobStorage> createStorage() const override { return std::make_unique<S3Storage>(); }
    std::unique_ptr<Queue> createQueue() const override { return std::make_unique<SqsQueue>(); }
};
struct GcpFactory : CloudFactory {
    std::unique_ptr<BlobStorage> createStorage() const override { return std::make_unique<GcsStorage>(); }
    std::unique_ptr<Queue> createQueue() const override { return std::make_unique<PubSubQueue>(); }
};

void exportReport(const CloudFactory& f, const std::string& id) {
    f.createStorage()->put("reports/" + id, {});
    f.createQueue()->publish("report-ready:" + id);
}

int main() {
    const char* cloud = std::getenv("CLOUD");
    std::unique_ptr<CloudFactory> f;
    if (cloud && std::string(cloud) == "gcp") f = std::make_unique<GcpFactory>();
    else f = std::make_unique<AwsFactory>();
    exportReport(*f, "42");
}
```

</details>

<details>
<summary><strong>Rust</strong></summary>

```rust
trait BlobStorage { fn put(&self, key: &str, data: &[u8]); }
trait Queue { fn publish(&self, msg: &str); }

// Associated types tie each family's products together at compile time.
trait CloudFactory {
    type Storage: BlobStorage;
    type Q: Queue;
    fn storage(&self) -> Self::Storage;
    fn queue(&self) -> Self::Q;
}

struct S3; impl BlobStorage for S3 { fn put(&self, k: &str, _: &[u8]) { println!("S3 put {k}"); } }
struct Sqs; impl Queue for Sqs { fn publish(&self, m: &str) { println!("SQS publish {m}"); } }
struct Gcs; impl BlobStorage for Gcs { fn put(&self, k: &str, _: &[u8]) { println!("GCS put {k}"); } }
struct PubSub; impl Queue for PubSub { fn publish(&self, m: &str) { println!("Pub/Sub publish {m}"); } }

struct Aws;
impl CloudFactory for Aws {
    type Storage = S3;
    type Q = Sqs;
    fn storage(&self) -> S3 { S3 }
    fn queue(&self) -> Sqs { Sqs }
}

struct Gcp;
impl CloudFactory for Gcp {
    type Storage = Gcs;
    type Q = PubSub;
    fn storage(&self) -> Gcs { Gcs }
    fn queue(&self) -> PubSub { PubSub }
}

// Static dispatch: the compiler generates one version per family, no vtables.
fn export_report<F: CloudFactory>(factory: &F, id: &str) {
    factory.storage().put(&format!("reports/{id}"), &[]);
    factory.queue().publish(&format!("report-ready:{id}"));
}

fn main() {
    if std::env::var("CLOUD").as_deref() == Ok("gcp") {
        export_report(&Gcp, "42");
    } else {
        export_report(&Aws, "42");
    }
}
```

</details>

<details>
<summary><strong>Python</strong></summary>

```python
import os
from typing import Protocol


class BlobStorage(Protocol):
    def put(self, key: str, data: bytes) -> None: ...


class Queue(Protocol):
    def publish(self, message: str) -> None: ...


class CloudFactory(Protocol):
    def create_storage(self) -> BlobStorage: ...
    def create_queue(self) -> Queue: ...


class S3Storage:
    def put(self, key: str, data: bytes) -> None:
        print(f"S3 put {key}")


class SqsQueue:
    def publish(self, message: str) -> None:
        print(f"SQS publish {message}")


class GcsStorage:
    def put(self, key: str, data: bytes) -> None:
        print(f"GCS put {key}")


class PubSubQueue:
    def publish(self, message: str) -> None:
        print(f"Pub/Sub publish {message}")


class AwsFactory:
    def create_storage(self) -> BlobStorage:
        return S3Storage()

    def create_queue(self) -> Queue:
        return SqsQueue()


class GcpFactory:
    def create_storage(self) -> BlobStorage:
        return GcsStorage()

    def create_queue(self) -> Queue:
        return PubSubQueue()


def export_report(factory: CloudFactory, report_id: str) -> None:
    factory.create_storage().put(f"reports/{report_id}", b"")
    factory.create_queue().publish(f"report-ready:{report_id}")


if __name__ == "__main__":
    factory: CloudFactory = GcpFactory() if os.getenv("CLOUD") == "gcp" else AwsFactory()
    export_report(factory, "42")
```

</details>

### Notas idiomáticas / Idiomatic notes
| Language | Note |
|----------|------|
| Java | Common in frameworks (JDBC `Connection` creates matching `Statement`s; `DocumentBuilderFactory`). DI profiles often replace hand-written factories. |
| Kotlin | `object` factories; anonymous `object :` implementations keep families compact. |
| Go | Small interfaces and constructor functions; often a struct of funcs instead of a factory interface. |
| C++ | Runtime polymorphism as above, or templates/policies for compile-time families. |
| Rust | Associated types enforce family consistency at compile time; `Box<dyn Trait>` when chosen at runtime. |
| Python | `Protocol` for structural typing; a module per provider exposing the same functions also works. |

### Referências / References
- *Design Patterns* (GoF), Abstract Factory
- [Refactoring.Guru: Abstract Factory](https://refactoring.guru/design-patterns/abstract-factory)
