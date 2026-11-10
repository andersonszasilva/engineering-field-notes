# Rust: Introdução

**Data:** 2026-11-03
**Área:** programming-languages / rust
**Tags:** rust, ownership, memory-safety, sistemas, performance, fundamentos, trade-offs

[Português](#português) | [English](#english)

---

## Português

### Contexto
Rust começou em 2006 como projeto pessoal de **Graydon Hoare**, passou a ser patrocinado pela **Mozilla** em 2009 e chegou à versão 1.0 em 2015. O objetivo era ambicioso: ter a performance e o controle de C e C++ **sem** as classes de bugs de memória (use-after-free, buffer overflow, data races) que causam boa parte das vulnerabilidades de segurança em software de sistemas.

A solução foi um sistema de tipos com **ownership e borrowing**, verificado em tempo de compilação, sem garbage collector. Desde 2021 a linguagem é mantida pela **Rust Foundation** (AWS, Google, Microsoft, Huawei, Mozilla e outros). Hoje Rust está no kernel do Linux, no Android, no Windows, no Chromium, em infraestrutura de cloud (AWS Firecracker, Cloudflare) e em uma nova geração de ferramentas de desenvolvimento (uv, ruff, Polars, Deno).

### Conceitos-chave
- **Ownership**: cada valor tem um único dono; quando o dono sai de escopo, o valor é liberado (drop). Não há GC nem `free` manual.
- **Borrowing**: é possível emprestar referências: várias imutáveis (`&T`) **ou** uma mutável (`&mut T`), nunca as duas ao mesmo tempo. Isso elimina data races em compilação.
- **Lifetimes**: o compilador garante que nenhuma referência vive mais que o valor que ela aponta. Na maioria dos casos são inferidas.
- **Borrow checker**: a parte do compilador que verifica essas regras. É a principal fonte de segurança e também da curva de aprendizado.
- **Sem null e sem exceções**: ausência de valor é `Option<T>` e erros são `Result<T, E>`, com o operador `?` para propagação.
- **Enums algébricos e pattern matching**: `enum` com dados e `match` exaustivo para modelar estados de forma segura.
- **Traits e generics**: polimorfismo por traits, resolvido em compilação (monomorfização) ou dinamicamente (`dyn Trait`).
- **Fearless concurrency**: os traits `Send` e `Sync` fazem o compilador rejeitar código concorrente inseguro.
- **`unsafe`**: um escape explícito e localizado para operações que o compilador não consegue provar seguras (FFI, ponteiros crus), normalmente encapsulado atrás de APIs seguras.
- **Cargo e editions**: Cargo é o build system e gerenciador de pacotes oficial (crates.io). *Editions* (2015, 2018, 2021, 2024) permitem evoluir a linguagem sem quebrar código antigo. Releases estáveis a cada 6 semanas.

### Vantagens
| Vantagem | Por que importa |
|----------|-----------------|
| **Segurança de memória sem GC** | Elimina, em código safe, use-after-free, double free, buffer overflow e data races, que respondem pela maioria das vulnerabilidades graves em C/C++. |
| **Performance comparável a C/C++** | Abstrações zero-cost, sem runtime pesado nem pausas de GC; latência previsível. |
| **"Se compila, provavelmente funciona"** | O sistema de tipos pega muitos bugs de lógica e concorrência antes da execução, o que reduz incidentes em produção. |
| **Tooling excelente** | Cargo, rustfmt, clippy, rust-analyzer, testes e docs (`cargo doc`) integrados e padronizados. |
| **Mensagens de erro do compilador** | Entre as melhores do mercado, muitas vezes já sugerindo a correção. |
| **Concorrência segura** | Paralelismo real com garantias de compilação; ecossistema assíncrono maduro com Tokio. |
| **Interop com C** | FFI direto, o que permite substituir componentes críticos de sistemas C/C++ aos poucos. |
| **Alvos variados** | Servidores, CLIs, embarcados (`no_std`), WebAssembly, kernels e drivers. |
| **Binários nativos** | Fáceis de distribuir, com startup rápido e baixo consumo de memória. |

### Desvantagens
| Desvantagem | Contexto / mitigação |
|-------------|----------------------|
| **Curva de aprendizado íngreme** | Ownership, borrowing, lifetimes e traits exigem uma nova forma de pensar; os primeiros meses costumam ser de "brigar com o borrow checker". |
| **Tempo de compilação** | Builds são lentos, principalmente com muitos generics e dependências. Mitigações: build incremental, dividir em crates menores, `sccache`, linker mais rápido (mold, lld). |
| **Complexidade do async** | `Pin`, `Send` em futures, escolha de runtime e a separação entre funções sync e async tornam o async Rust mais difícil que goroutines ou coroutines. |
| **Produtividade inicial menor** | Para protótipos e serviços CRUD, Go, Kotlin ou Python entregam mais rápido. |
| **Ecossistema mais jovem** | Excelente em infraestrutura e CLIs, mas com menos opções maduras em GUI, ORMs, ML e frameworks corporativos; muitas crates ainda estão em versão 0.x. |
| **Mercado de trabalho** | Demanda crescente, mas o pool de profissionais experientes ainda é pequeno, o que pesa em contratação. |
| **`unsafe` ainda existe** | FFI e código de baixo nível exigem `unsafe`; a segurança depende de encapsular e revisar bem esses trechos (Miri ajuda a testar). |
| **Árvore de dependências** | Projetos puxam muitas crates pequenas; é preciso cuidar de supply chain (`cargo audit`, `cargo deny`, `cargo vet`). |

### Por que é relevante
1. **Segurança de memória virou requisito**: governos (CISA, NSA, Casa Branca) e grandes empresas recomendam linguagens memory-safe para código novo, e Rust é a principal opção quando GC não é aceitável.
2. **Adoção em sistemas críticos**: Linux (drivers desde o kernel 6.1), Android, Windows, Chromium, AWS (Firecracker, partes de S3 e Lambda) e Cloudflare usam Rust em produção, com quedas mensuráveis em vulnerabilidades de memória.
3. **Uma nova geração de ferramentas**: uv, ruff, Polars, Deno, Turbopack, Biome e ripgrep mostram como Rust deixa ferramentas de desenvolvedor ordens de grandeza mais rápidas.
4. **Infraestrutura de dados e cloud**: bancos e engines como TiKV, Materialize, DataFusion e Qdrant, proxies e runtimes serverless apostam em Rust por performance e confiabilidade.
5. **Substituição incremental de C/C++**: a interop com C permite reescrever primeiro os componentes mais expostos (parsers, codecs, rede) sem reescrever o sistema inteiro.

### Quando escolher Rust
**Rust costuma ser uma boa escolha quando:**
- o sistema precisa de performance de C/C++ e segurança de memória ao mesmo tempo;
- o código processa entradas não confiáveis (parsers, rede, codecs, criptografia);
- latência previsível importa e pausas de GC não são aceitáveis;
- o projeto é infraestrutura, CLI, embarcado, WebAssembly ou extensão nativa de Python/Node.

**Vale considerar alternativas quando:**
- o objetivo é entregar rápido um serviço de negócio ou CRUD (Go, Java, Kotlin, Python);
- o time não tem tempo para absorver a curva de aprendizado;
- o domínio depende de um ecossistema que Rust ainda não tem maduro (ML de pesquisa, GUIs desktop complexas, frameworks corporativos);
- a base de código existente em C++ é enorme e a integração custaria mais que os ganhos.

**Armadilhas comuns:**
- Espalhar `.clone()`, `.unwrap()` e `Rc<RefCell<T>>` para "calar" o compilador em vez de rever o design de ownership.
- Bloquear threads dentro de código async (I/O síncrono ou CPU pesada no runtime do Tokio) sem `spawn_blocking`.
- Usar `unsafe` sem documentar as invariantes nem testar com Miri.
- Tentar modelar hierarquias de herança de linguagens OO em vez de usar enums e traits.
- Começar pelo async e por lifetimes complexos antes de dominar ownership básico.

### Referências
- [The Rust Programming Language ("The Book")](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) (unsafe Rust)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
- [This Week in Rust](https://this-week-in-rust.org/)
- *Programming Rust*, Jim Blandy, Jason Orendorff & Leonora Tindall
- *Rust for Rustaceans*, Jon Gjengset
- *Rust Atomics and Locks*, Mara Bos

---

## English

### Context
Rust started in 2006 as a personal project by **Graydon Hoare**, was sponsored by **Mozilla** starting in 2009, and reached version 1.0 in 2015. The goal was ambitious: deliver the performance and control of C and C++ **without** the classes of memory bugs (use-after-free, buffer overflows, data races) behind a large share of security vulnerabilities in systems software.

The solution was a type system with **ownership and borrowing**, checked at compile time, with no garbage collector. Since 2021 the language has been stewarded by the **Rust Foundation** (AWS, Google, Microsoft, Huawei, Mozilla, and others). Today Rust is in the Linux kernel, Android, Windows, Chromium, cloud infrastructure (AWS Firecracker, Cloudflare), and a new generation of developer tools (uv, ruff, Polars, Deno).

### Key concepts
- **Ownership**: Every value has a single owner; when the owner goes out of scope, the value is freed (dropped). No GC and no manual `free`.
- **Borrowing**: You can lend references: many immutable ones (`&T`) **or** one mutable one (`&mut T`), never both at the same time. This eliminates data races at compile time.
- **Lifetimes**: The compiler guarantees that no reference outlives the value it points to. In most cases they are inferred.
- **Borrow checker**: The part of the compiler that enforces these rules. It is the main source of safety and also of the learning curve.
- **No null and no exceptions**: Absence of a value is `Option<T>` and errors are `Result<T, E>`, with the `?` operator for propagation.
- **Algebraic enums and pattern matching**: Enums that carry data and exhaustive `match` for modeling states safely.
- **Traits and generics**: Polymorphism through traits, resolved at compile time (monomorphization) or dynamically (`dyn Trait`).
- **Fearless concurrency**: The `Send` and `Sync` traits make the compiler reject unsafe concurrent code.
- **`unsafe`**: An explicit, localized escape hatch for operations the compiler cannot prove safe (FFI, raw pointers), usually wrapped behind safe APIs.
- **Cargo and editions**: Cargo is the official build system and package manager (crates.io). *Editions* (2015, 2018, 2021, 2024) let the language evolve without breaking old code. Stable releases every six weeks.

### Advantages
| Advantage | Why it matters |
|-----------|----------------|
| **Memory safety without a GC** | In safe code it eliminates use-after-free, double free, buffer overflows, and data races, which account for most serious vulnerabilities in C/C++. |
| **Performance comparable to C/C++** | Zero-cost abstractions, no heavy runtime, no GC pauses; predictable latency. |
| **"If it compiles, it probably works"** | The type system catches many logic and concurrency bugs before runtime, reducing production incidents. |
| **Excellent tooling** | Cargo, rustfmt, clippy, rust-analyzer, tests, and docs (`cargo doc`) are integrated and standardized. |
| **Compiler error messages** | Among the best available, often suggesting the fix directly. |
| **Safe concurrency** | Real parallelism with compile-time guarantees; a mature async ecosystem with Tokio. |
| **C interop** | Direct FFI, which allows critical components of C/C++ systems to be replaced gradually. |
| **Many targets** | Servers, CLIs, embedded (`no_std`), WebAssembly, kernels, and drivers. |
| **Native binaries** | Easy to distribute, with fast startup and low memory usage. |

### Disadvantages
| Disadvantage | Context / mitigation |
|--------------|----------------------|
| **Steep learning curve** | Ownership, borrowing, lifetimes, and traits require a new way of thinking; the first months are often spent "fighting the borrow checker". |
| **Compilation time** | Builds are slow, especially with heavy generics and many dependencies. Mitigations: incremental builds, splitting into smaller crates, `sccache`, a faster linker (mold, lld). |
| **Async complexity** | `Pin`, `Send` futures, choosing a runtime, and the split between sync and async functions make async Rust harder than goroutines or coroutines. |
| **Lower initial productivity** | For prototypes and CRUD services, Go, Kotlin, or Python ship faster. |
| **Younger ecosystem** | Excellent for infrastructure and CLIs, but fewer mature options for GUIs, ORMs, ML, and enterprise frameworks; many crates are still at 0.x. |
| **Job market** | Demand is growing, but the pool of experienced engineers is still small, which affects hiring. |
| **`unsafe` still exists** | FFI and low-level code require `unsafe`; safety depends on encapsulating and reviewing those sections carefully (Miri helps test them). |
| **Dependency trees** | Projects pull in many small crates; supply chain needs attention (`cargo audit`, `cargo deny`, `cargo vet`). |

### Why it matters
1. **Memory safety has become a requirement**: Governments (CISA, NSA, the White House) and large companies recommend memory-safe languages for new code, and Rust is the main option when a GC is not acceptable.
2. **Adoption in critical systems**: Linux (drivers since kernel 6.1), Android, Windows, Chromium, AWS (Firecracker, parts of S3 and Lambda), and Cloudflare run Rust in production, with measurable drops in memory-safety vulnerabilities.
3. **A new generation of tools**: uv, ruff, Polars, Deno, Turbopack, Biome, and ripgrep show how Rust makes developer tools orders of magnitude faster.
4. **Data and cloud infrastructure**: Databases and engines such as TiKV, Materialize, DataFusion, and Qdrant, along with proxies and serverless runtimes, bet on Rust for performance and reliability.
5. **Incremental replacement of C/C++**: C interop allows rewriting the most exposed components first (parsers, codecs, networking) without rewriting the whole system.

### When to choose Rust
**Rust is usually a good choice when:**
- the system needs C/C++ performance and memory safety at the same time;
- the code processes untrusted input (parsers, networking, codecs, cryptography);
- predictable latency matters and GC pauses are unacceptable;
- the project is infrastructure, a CLI, embedded, WebAssembly, or a native extension for Python/Node.

**Consider alternatives when:**
- the goal is to ship a business or CRUD service quickly (Go, Java, Kotlin, Python);
- the team does not have time to absorb the learning curve;
- the domain depends on an ecosystem Rust does not yet have in mature form (research ML, complex desktop GUIs, enterprise frameworks);
- the existing C++ codebase is huge and integration would cost more than the gains.

**Common pitfalls:**
- Sprinkling `.clone()`, `.unwrap()`, and `Rc<RefCell<T>>` to "silence" the compiler instead of revisiting the ownership design.
- Blocking threads inside async code (synchronous I/O or heavy CPU work on the Tokio runtime) without `spawn_blocking`.
- Using `unsafe` without documenting the invariants or testing with Miri.
- Trying to model OO inheritance hierarchies instead of using enums and traits.
- Starting with async and complex lifetimes before mastering basic ownership.

### References
- [The Rust Programming Language ("The Book")](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) (unsafe Rust)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
- [This Week in Rust](https://this-week-in-rust.org/)
- *Programming Rust*, Jim Blandy, Jason Orendorff & Leonora Tindall
- *Rust for Rustaceans*, Jon Gjengset
- *Rust Atomics and Locks*, Mara Bos
