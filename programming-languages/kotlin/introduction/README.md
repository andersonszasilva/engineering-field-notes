# Kotlin: Introdução

**Data:** 2026-11-03
**Área:** programming-languages / kotlin
**Tags:** kotlin, jvm, android, coroutines, fundamentos, trade-offs

[Português](#português) | [English](#english)

---

## Português

### Contexto
Kotlin foi criado pela **JetBrains** (a empresa do IntelliJ IDEA), anunciado em 2011 e lançado na versão 1.0 em 2016. O objetivo era pragmático: uma linguagem para a JVM mais concisa e segura que Java, mas com **interoperabilidade total** com o código e as bibliotecas Java existentes, para que times pudessem adotá-la arquivo por arquivo.

A virada veio em 2017, quando o Google anunciou suporte oficial no Android, e em 2019, quando o Android passou a ser *Kotlin-first*. Hoje Kotlin é a linguagem padrão no Android e uma alternativa sólida a Java no backend, com suporte de primeira classe no Spring e frameworks próprios como o Ktor.

### Conceitos-chave
- **Interoperabilidade com Java**: Kotlin compila para bytecode da JVM e chama (e é chamado por) código Java sem camadas de adaptação.
- **Null safety no sistema de tipos**: `String` e `String?` são tipos diferentes. O compilador obriga a tratar valores nulos (`?.`, `?:`, smart casts).
- **Coroutines**: modelo de concorrência baseado em funções `suspend` e **concorrência estruturada** (`CoroutineScope`, `launch`, `async`), com `Flow` para streams assíncronos.
- **Concisão expressiva**: `data class`, `sealed class/interface`, `when` exaustivo, extension functions, inferência de tipos, parâmetros nomeados e com valor padrão.
- **Multiplataforma**: além da JVM, compila para JavaScript, WebAssembly e nativo (via LLVM). O **Kotlin Multiplatform (KMP)** permite compartilhar lógica entre Android, iOS, backend e web.
- **Compilador K2**: novo frontend do compilador, estável desde o Kotlin 2.0 (2024), com compilação mais rápida e base para a evolução da linguagem.

### Vantagens
| Vantagem | Por que importa |
|----------|-----------------|
| **Null safety** | Elimina a maior parte dos `NullPointerException` em tempo de compilação, que é uma das classes de bug mais comuns em Java. |
| **Concisão sem perder clareza** | Menos boilerplate (data classes, propriedades, inferência) significa menos código para revisar e manter. |
| **Interop com Java** | Adoção incremental: dá para introduzir Kotlin num projeto Java existente e usar todo o ecossistema da JVM (Spring, Hibernate, Kafka clients etc.). |
| **Coroutines** | Código assíncrono escrito de forma sequencial, com cancelamento e escopo estruturado, sem *callback hell* nem cadeias reativas complexas. |
| **Modelagem de domínio** | `sealed` + `when` exaustivo + data classes tornam estados e erros explícitos e verificados pelo compilador. |
| **Padrão no Android** | Jetpack Compose, bibliotecas e documentação oficial do Android são Kotlin-first. |
| **Tooling** | Suporte excelente no IntelliJ IDEA e Android Studio (a mesma empresa cria a linguagem e a IDE). |
| **Compartilhamento de código** | KMP permite compartilhar regras de negócio entre mobile e backend mantendo UI nativa. |

### Desvantagens
| Desvantagem | Contexto / mitigação |
|-------------|----------------------|
| **Tempo de compilação** | Historicamente mais lento que Java. O compilador K2, o build incremental e o KSP (no lugar do kapt) reduziram bastante o problema. |
| **Platform types na interop** | Valores vindos de Java têm nulabilidade desconhecida (`String!`), então a null safety pode "vazar". Mitigação: anotações de nulabilidade (JSpecify) no código Java. |
| **Curva das coroutines** | Concorrência estruturada, dispatchers, cancelamento e tratamento de exceções exigem estudo; uso incorreto gera vazamentos e bugs sutis. |
| **Classes `final` por padrão** | Conflita com frameworks que usam proxies (Spring, JPA). Mitigação: plugins `kotlin-spring` / `kotlin-allopen` e `kotlin-jpa`. |
| **Mercado de backend menor** | Muito forte no Android, mas no backend o pool de profissionais ainda é menor que o de Java. |
| **Dependência de um fornecedor** | A evolução da linguagem e o melhor tooling estão concentrados na JetBrains (com o Google como parceiro via Kotlin Foundation). Suporte fora do IntelliJ é mais limitado. |
| **Java está diminuindo a distância** | Records, sealed classes, pattern matching e virtual threads reduziram parte da vantagem de Kotlin no backend. |
| **Liberdade demais** | Extension functions, DSLs e operadores customizados podem gerar código "esperto" e difícil de ler sem convenções de time. |

### Por que é relevante
1. **É a linguagem do Android**: praticamente todo desenvolvimento Android novo é feito em Kotlin, com Jetpack Compose como toolkit de UI padrão.
2. **Uma alternativa moderna na JVM**: entrega segurança e expressividade sem abrir mão do ecossistema, da performance e da observabilidade da JVM.
3. **Coroutines mudaram a forma de escrever código assíncrono**: o modelo de concorrência estruturada influenciou outras linguagens e o próprio Java (Structured Concurrency).
4. **Multiplataforma real**: KMP e Compose Multiplatform viabilizam compartilhar código entre Android, iOS, desktop, web e backend sem abandonar UI nativa.
5. **Suporte de primeira classe no backend**: Spring Boot, Ktor, Quarkus e Micronaut suportam Kotlin oficialmente; empresas grandes usam Kotlin em produção no servidor.

### Quando escolher Kotlin
**Kotlin costuma ser uma boa escolha quando:**
- o projeto é Android (ou mobile multiplataforma com KMP);
- o time quer os benefícios da JVM com mais segurança contra null e menos boilerplate;
- o domínio tem muitos estados e regras que se beneficiam de `sealed` + `when` exaustivo;
- há muito I/O assíncrono e o time prefere coroutines a programação reativa.

**Vale considerar alternativas quando:**
- a organização tem forte padronização em Java e o ganho não justifica uma segunda linguagem no stack;
- o foco é ciência de dados ou ML (Python domina o ecossistema);
- é preciso controle fino de memória e latência sem GC (Rust, C++);
- o problema pede binários pequenos e simples de distribuir com tooling mínimo (Go).

**Armadilhas comuns:**
- Usar `!!` para "calar" o compilador, jogando fora a null safety.
- Usar `GlobalScope` ou ignorar a concorrência estruturada, gerando coroutines órfãs.
- Bloquear threads (`Thread.sleep`, JDBC) dentro de coroutines sem trocar para `Dispatchers.IO`.
- Escrever "Java com sintaxe Kotlin" ou, no extremo oposto, abusar de DSLs e extensions até o código ficar ilegível.
- Esquecer os plugins `allopen`/`jpa` ao usar Spring e JPA.

### Referências
- [Kotlin: documentação oficial](https://kotlinlang.org/docs/home.html)
- [Kotlin Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [KEEP: Kotlin Evolution and Enhancement Process](https://github.com/Kotlin/KEEP)
- [Kotlin Multiplatform](https://www.jetbrains.com/kotlin-multiplatform/)
- *Kotlin in Action*, Dmitry Jemerov & Svetlana Isakova
- *Effective Kotlin*, Marcin Moskała

---

## English

### Context
Kotlin was created by **JetBrains** (the company behind IntelliJ IDEA), announced in 2011, and released as version 1.0 in 2016. The goal was pragmatic: a JVM language that is more concise and safer than Java, but with **full interoperability** with existing Java code and libraries, so teams could adopt it one file at a time.

The turning point came in 2017, when Google announced official support on Android, and in 2019, when Android became *Kotlin-first*. Today Kotlin is the default language for Android and a solid alternative to Java on the backend, with first-class support in Spring and its own frameworks such as Ktor.

### Key concepts
- **Java interoperability**: Kotlin compiles to JVM bytecode and calls (and is called by) Java code without adapter layers.
- **Null safety in the type system**: `String` and `String?` are different types. The compiler forces you to handle null values (`?.`, `?:`, smart casts).
- **Coroutines**: A concurrency model based on `suspend` functions and **structured concurrency** (`CoroutineScope`, `launch`, `async`), with `Flow` for asynchronous streams.
- **Expressive conciseness**: `data class`, `sealed class/interface`, exhaustive `when`, extension functions, type inference, named and default parameters.
- **Multiplatform**: Besides the JVM, it compiles to JavaScript, WebAssembly, and native code (via LLVM). **Kotlin Multiplatform (KMP)** lets you share logic across Android, iOS, backend, and web.
- **K2 compiler**: The new compiler frontend, stable since Kotlin 2.0 (2024), with faster compilation and a foundation for future language evolution.

### Advantages
| Advantage | Why it matters |
|-----------|----------------|
| **Null safety** | Eliminates most `NullPointerException`s at compile time, one of the most common classes of bugs in Java. |
| **Conciseness without losing clarity** | Less boilerplate (data classes, properties, inference) means less code to review and maintain. |
| **Java interop** | Incremental adoption: you can introduce Kotlin into an existing Java project and use the entire JVM ecosystem (Spring, Hibernate, Kafka clients, etc.). |
| **Coroutines** | Asynchronous code written sequentially, with cancellation and structured scoping, without callback hell or complex reactive chains. |
| **Domain modeling** | `sealed` + exhaustive `when` + data classes make states and errors explicit and compiler-checked. |
| **Android standard** | Jetpack Compose, libraries, and official Android documentation are Kotlin-first. |
| **Tooling** | Excellent support in IntelliJ IDEA and Android Studio (the same company builds the language and the IDE). |
| **Code sharing** | KMP lets you share business logic between mobile and backend while keeping native UI. |

### Disadvantages
| Disadvantage | Context / mitigation |
|--------------|----------------------|
| **Compilation time** | Historically slower than Java. The K2 compiler, incremental builds, and KSP (instead of kapt) have reduced the problem significantly. |
| **Platform types in interop** | Values coming from Java have unknown nullability (`String!`), so null safety can "leak". Mitigation: nullability annotations (JSpecify) in Java code. |
| **Coroutines learning curve** | Structured concurrency, dispatchers, cancellation, and exception handling require study; misuse leads to leaks and subtle bugs. |
| **`final` classes by default** | Conflicts with frameworks that rely on proxies (Spring, JPA). Mitigation: the `kotlin-spring` / `kotlin-allopen` and `kotlin-jpa` plugins. |
| **Smaller backend market** | Very strong on Android, but on the backend the talent pool is still smaller than Java's. |
| **Single-vendor dependency** | Language evolution and the best tooling are concentrated at JetBrains (with Google as a partner through the Kotlin Foundation). Support outside IntelliJ is more limited. |
| **Java is closing the gap** | Records, sealed classes, pattern matching, and virtual threads have reduced part of Kotlin's advantage on the backend. |
| **Too much freedom** | Extension functions, DSLs, and custom operators can lead to "clever" code that is hard to read without team conventions. |

### Why it matters
1. **It is the language of Android**: Virtually all new Android development is done in Kotlin, with Jetpack Compose as the default UI toolkit.
2. **A modern alternative on the JVM**: It delivers safety and expressiveness without giving up the JVM's ecosystem, performance, and observability.
3. **Coroutines changed how async code is written**: The structured concurrency model has influenced other languages and Java itself (Structured Concurrency).
4. **Real multiplatform**: KMP and Compose Multiplatform make it possible to share code across Android, iOS, desktop, web, and backend without abandoning native UI.
5. **First-class backend support**: Spring Boot, Ktor, Quarkus, and Micronaut officially support Kotlin; large companies run Kotlin in production on the server.

### When to choose Kotlin
**Kotlin is usually a good choice when:**
- the project is Android (or cross-platform mobile with KMP);
- the team wants the benefits of the JVM with stronger null safety and less boilerplate;
- the domain has many states and rules that benefit from `sealed` + exhaustive `when`;
- there is a lot of asynchronous I/O and the team prefers coroutines over reactive programming.

**Consider alternatives when:**
- the organization is heavily standardized on Java and the gain does not justify a second language in the stack;
- the focus is data science or ML (Python dominates the ecosystem);
- you need fine-grained control over memory and latency without a GC (Rust, C++);
- the problem calls for small binaries that are simple to distribute with minimal tooling (Go).

**Common pitfalls:**
- Using `!!` to "silence" the compiler, throwing away null safety.
- Using `GlobalScope` or ignoring structured concurrency, leaving orphaned coroutines.
- Blocking threads (`Thread.sleep`, JDBC) inside coroutines without switching to `Dispatchers.IO`.
- Writing "Java with Kotlin syntax" or, at the other extreme, overusing DSLs and extensions until the code becomes unreadable.
- Forgetting the `allopen`/`jpa` plugins when using Spring and JPA.

### References
- [Kotlin: official documentation](https://kotlinlang.org/docs/home.html)
- [Kotlin Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [KEEP: Kotlin Evolution and Enhancement Process](https://github.com/Kotlin/KEEP)
- [Kotlin Multiplatform](https://www.jetbrains.com/kotlin-multiplatform/)
- *Kotlin in Action*, Dmitry Jemerov & Svetlana Isakova
- *Effective Kotlin*, Marcin Moskała
