# Java: Introdução

**Data:** 2026-10-09
**Área:** programming-languages / java
**Tags:** java, jvm, fundamentos, trade-offs

[Português](#português) | [English](#english)

---

## Português

### Contexto
Java nasceu em 1995 na Sun Microsystems (hoje mantido pela Oracle e pela comunidade via OpenJDK) com a promessa de *"write once, run anywhere"*: o código é compilado para **bytecode** e executado pela **JVM** (Java Virtual Machine), que abstrai o sistema operacional e o hardware.

Trinta anos depois, continua sendo uma das linguagens mais usadas em sistemas de backend, principalmente em ambientes onde confiabilidade, escala e longevidade do código importam mais do que velocidade de prototipação.

### Conceitos-chave
- **JVM**: máquina virtual que executa bytecode. Linguagens como Kotlin, Scala e Clojure também rodam nela.
- **JIT (Just-In-Time compiler)**: compila em tempo de execução os trechos "quentes" do código para código nativo otimizado, usando informações reais de uso.
- **Garbage Collector**: gerenciamento automático de memória, com vários algoritmos (G1, ZGC, Shenandoah, Parallel) para diferentes perfis de latência e throughput.
- **Tipagem estática e forte**: erros pegos em compilação, refatorações seguras em bases de código grandes.
- **Cadência de releases**: uma versão a cada 6 meses desde o Java 10 (2018), com versões **LTS** a cada 2 anos (8, 11, 17, 21, 25).

### Vantagens
| Vantagem | Por que importa |
|----------|-----------------|
| **Performance em regime** | O JIT otimiza com base no comportamento real; aplicações de longa duração chegam perto de C/C++ em muitos cenários. |
| **Ecossistema maduro** | Spring, Quarkus, Micronaut, Hibernate, Kafka, Elasticsearch, Cassandra, Flink: quase todo problema de backend já tem solução testada em produção. |
| **Retrocompatibilidade** | Código de décadas atrás ainda roda em JVMs modernas. Isso reduz o custo de manutenção de longo prazo. |
| **Observabilidade da JVM** | JFR (Java Flight Recorder), JMX, heap dumps, thread dumps: ferramentas de diagnóstico de produção muito ricas. |
| **Concorrência** | `java.util.concurrent`, e desde o Java 21 as **virtual threads**, que permitem milhões de threads leves no modelo bloqueante simples. |
| **Tipagem estática em escala** | Facilita trabalhar com muitos times e muitos serviços; IDEs e ferramentas de análise estática são excelentes. |
| **Mercado e comunidade** | Enorme base de profissionais, conteúdo e suporte, o que reduz o risco de contratação e de dependência de poucas pessoas. |

### Desvantagens
| Desvantagem | Contexto / mitigação |
|-------------|----------------------|
| **Verbosidade** | Historicamente mais cerimoniosa que Python ou Kotlin. Melhorou muito com `var`, records, text blocks, pattern matching e switch expressions. |
| **Startup e consumo de memória** | A JVM demora para "aquecer" e usa mais memória, o que pesa em serverless e CLIs. Mitigações: GraalVM Native Image, CRaC, Project Leyden (AOT cache). |
| **Complexidade de tuning** | Escolher e ajustar GC, heap e flags da JVM exige conhecimento específico. |
| **Null safety** | Não há nulabilidade no sistema de tipos (`NullPointerException` ainda é comum). `Optional` e anotações ajudam, mas não resolvem. |
| **Legado** | Muitas empresas ainda presas no Java 8 com frameworks antigos, o que dá a falsa impressão de que "Java é velho". |
| **Licenciamento confuso** | A mudança de licença do Oracle JDK gerou dúvidas. Na prática há distribuições OpenJDK gratuitas (Temurin, Corretto, Zulu etc.). |
| **Generics com type erasure** | Informação de tipo genérico não existe em runtime, o que limita algumas abstrações. |

### Por que ainda é relevante
1. **Está onde o dinheiro roda**: bancos, fintechs, seguradoras, e-commerce e telecom têm sistemas críticos em Java. Trocar isso é caro e arriscado, então eles evoluem esses sistemas em vez de reescrever.
2. **Base da infraestrutura de dados**: Kafka, Hadoop, Spark, Flink, Elasticsearch, Cassandra e Zookeeper são JVM. Entender a JVM ajuda a operar essas peças.
3. **A linguagem voltou a evoluir rápido**: com releases semestrais, recursos modernos chegam com frequência: records, sealed classes, pattern matching, virtual threads, FFM API (interop nativa sem JNI) e GCs de baixa latência como o ZGC geracional.
4. **Adaptação ao cloud-native**: Quarkus, Micronaut, Spring Boot 3 + GraalVM e o Project Leyden atacam justamente os pontos fracos de startup e memória.
5. **Previsibilidade para o negócio**: LTS com suporte longo, compatibilidade forte e um ecossistema estável tornam Java uma escolha de baixo risco para sistemas que vão viver 10+ anos.

### Quando escolher Java
**Java costuma ser uma boa escolha quando:**
- o serviço é de longa duração, com alto throughput e requisitos de confiabilidade;
- vários times vão manter o código por muitos anos;
- a organização já tem expertise, observabilidade e plataforma montadas em torno da JVM.

**Vale considerar alternativas quando:**
- o workload é serverless ou de vida curta, onde startup domina o custo (a menos que se use native image);
- o foco é ciência de dados ou ML (Python domina o ecossistema);
- é preciso controle fino de memória e latência sem GC (Rust, C++);
- o problema pede binários pequenos e simples de distribuir (Go).

**Armadilhas comuns:**
- Julgar Java pelo Java 8 + Spring XML de 2014.
- Ignorar o custo de *aquecimento* do JIT em benchmarks e testes de carga.
- Deixar o GC e o heap no padrão em containers sem entender os limites de memória.

### Referências
- [OpenJDK](https://openjdk.org/)
- [JEPs: JDK Enhancement Proposals](https://openjdk.org/jeps/0)
- [Inside Java](https://inside.java/)
- *Effective Java*, Joshua Bloch
- *Java Concurrency in Practice*, Brian Goetz

---

## English

### Context
Java was created in 1995 at Sun Microsystems (now maintained by Oracle and the community through OpenJDK) with the promise of *"write once, run anywhere"*: the code is compiled into **bytecode** and executed by the **JVM** (Java Virtual Machine), which abstracts away the operating system and hardware.

Thirty years later, it remains one of the most widely used languages for backend systems, especially in environments where reliability, scalability, and code longevity matter more than prototyping speed.

### Key concepts
- **JVM**: A virtual machine that executes bytecode. Languages such as Kotlin, Scala, and Clojure also run on it.
- **JIT (Just-In-Time compiler)**: Compiles "hot" code paths into optimized native code at runtime, using real-world usage information.
- **Garbage Collector**: Automatic memory management, with several algorithms (G1, ZGC, Shenandoah, Parallel) designed for different latency and throughput profiles.
- **Static and strong typing**: Errors are caught at compile time, enabling safer refactoring in large codebases.
- **Release cadence**: One release every six months since Java 10 (2018), with **LTS** (Long-Term Support) releases every two years (8, 11, 17, 21, 25).

### Advantages
| Advantage | Why it matters |
|-----------|----------------|
| **Steady-state performance** | The JIT optimizes based on actual runtime behavior; long-running applications can approach C/C++ performance in many scenarios. |
| **Mature ecosystem** | Spring, Quarkus, Micronaut, Hibernate, Kafka, Elasticsearch, Cassandra, Flink: almost every backend problem already has a solution that has been tested in production. |
| **Backward compatibility** | Code written decades ago still runs on modern JVMs. This reduces long-term maintenance costs. |
| **JVM observability** | JFR (Java Flight Recorder), JMX, heap dumps, thread dumps: a rich set of tools for diagnosing production issues. |
| **Concurrency** | `java.util.concurrent`, and since Java 21 **virtual threads**, which enable millions of lightweight threads using a simple blocking model. |
| **Static typing at scale** | Makes it easier to work across many teams and many services; IDEs and static analysis tools are excellent. |
| **Market and community** | A large pool of professionals, content, and support reduces hiring risk and dependency on a small number of individuals. |

### Disadvantages
| Disadvantage | Context / mitigation |
|--------------|----------------------|
| **Verbosity** | Historically more verbose than Python or Kotlin. This has improved significantly with `var`, records, text blocks, pattern matching, and switch expressions. |
| **Startup and memory consumption** | The JVM takes time to "warm up" and uses more memory, which matters for serverless and CLIs. Mitigations: GraalVM Native Image, CRaC, Project Leyden (AOT cache). |
| **Tuning complexity** | Choosing and tuning the GC, heap, and JVM flags requires specific knowledge. |
| **Null safety** | There is no nullability in the type system (`NullPointerException` is still common). `Optional` and annotations help, but they do not solve the problem. |
| **Legacy** | Many companies are still stuck on Java 8 with old frameworks, which gives the false impression that "Java is old". |
| **Confusing licensing** | Changes to the Oracle JDK license caused confusion. In practice, free OpenJDK distributions are available (Temurin, Corretto, Zulu, etc.). |
| **Generics with type erasure** | Generic type information does not exist at runtime, which limits some abstractions. |

### Why it still matters
1. **It is where the money flows**: Banks, fintechs, insurance companies, e-commerce businesses, and telecom companies run critical systems on Java. Replacing these systems is expensive and risky, so they evolve them instead of rewriting them.
2. **Foundation of data infrastructure**: Kafka, Hadoop, Spark, Flink, Elasticsearch, Cassandra, and ZooKeeper all run on the JVM. Understanding the JVM helps you operate these pieces.
3. **The language is evolving fast again**: With six-month releases, modern features arrive frequently: records, sealed classes, pattern matching, virtual threads, the FFM API (native interop without JNI), and low-latency GCs such as Generational ZGC.
4. **Adapting to cloud-native**: Quarkus, Micronaut, Spring Boot 3 + GraalVM, and Project Leyden target exactly the weak spots of startup time and memory usage.
5. **Predictability for the business**: Long-term LTS support, strong compatibility, and a stable ecosystem make Java a low-risk choice for systems expected to live 10+ years.

### When to choose Java
**Java is usually a good choice when:**
- the service is long-running, with high throughput and reliability requirements;
- multiple teams will maintain the code for many years;
- the organization already has expertise, observability, and a platform built around the JVM.

**Consider alternatives when:**
- the workload is serverless or short-lived, where startup dominates cost (unless you use native image);
- the focus is data science or ML (Python dominates the ecosystem);
- you need fine-grained control over memory and latency without a GC (Rust, C++);
- the problem calls for small binaries that are simple to distribute (Go).

**Common pitfalls:**
- Judging Java by the Java 8 + Spring XML of 2014.
- Ignoring the JIT *warm-up* cost in benchmarks and load tests.
- Leaving GC and heap settings at their defaults in containers without understanding the memory limits.

### References
- [OpenJDK](https://openjdk.org/)
- [JEPs: JDK Enhancement Proposals](https://openjdk.org/jeps/0)
- [Inside Java](https://inside.java/)
- *Effective Java*, Joshua Bloch
- *Java Concurrency in Practice*, Brian Goetz
