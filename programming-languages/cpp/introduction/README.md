# C++: Introdução

**Data:** 2026-11-03
**Área:** programming-languages / cpp
**Tags:** cpp, c++, performance, sistemas, memória, fundamentos, trade-offs

[Português](#português) | [English](#english)

---

## Português

### Contexto
C++ foi criado por **Bjarne Stroustrup** no Bell Labs a partir de 1979, inicialmente como *"C with Classes"*, e renomeado para C++ em 1983. A ideia era unir a eficiência e o acesso ao hardware do C com abstrações de alto nível (classes, e depois templates) sem custo extra em tempo de execução.

O primeiro padrão ISO saiu em 1998 (C++98). Depois de um longo intervalo, o **C++11** modernizou a linguagem (move semantics, smart pointers, lambdas, `auto`) e, desde então, um novo padrão sai a cada 3 anos: C++14, 17, 20, 23 e 26. Hoje C++ continua sendo a base de sistemas onde performance e controle de recursos são inegociáveis: engines de jogos, bancos de dados, browsers, trading de alta frequência, runtimes de ML e sistemas embarcados.

### Conceitos-chave
- **Zero-cost abstractions**: *"o que você não usa, você não paga; o que você usa, não daria para escrever melhor à mão"*. Abstrações devem compilar para código tão eficiente quanto o equivalente manual.
- **Compilação para código nativo**: sem VM nem garbage collector; o compilador (GCC, Clang, MSVC) gera código de máquina otimizado para a arquitetura alvo.
- **RAII (Resource Acquisition Is Initialization)**: recursos (memória, arquivos, locks, sockets) são liberados de forma determinística no destrutor quando o objeto sai de escopo.
- **Smart pointers**: `std::unique_ptr` (posse exclusiva) e `std::shared_ptr` (posse compartilhada) substituem `new`/`delete` manuais no C++ moderno.
- **Move semantics**: transferência de recursos sem cópia (`std::move`, rvalue references), essencial para performance com tipos grandes.
- **Templates e concepts**: programação genérica resolvida em compilação; *concepts* (C++20) restringem templates e melhoram as mensagens de erro.
- **Undefined Behavior (UB)**: operações inválidas (acesso fora dos limites, uso após free, overflow de inteiro com sinal) não têm comportamento definido, e o compilador pode otimizar assumindo que nunca acontecem.
- **Padronização ISO**: a linguagem é definida por um comitê (WG21), com múltiplos compiladores e implementações independentes.

### Vantagens
| Vantagem | Por que importa |
|----------|-----------------|
| **Performance máxima e previsível** | Sem pausas de GC nem warm-up de JIT. Latência determinística, essencial em trading, jogos, áudio e sistemas de tempo real. |
| **Controle fino de memória e hardware** | Layout de dados, alocação, alinhamento, cache locality, SIMD: dá para otimizar no nível que o problema exige. |
| **Abstrações sem custo** | Templates, RAII e classes permitem código expressivo sem sacrificar eficiência. |
| **Interop com C** | Acesso direto a APIs de sistema operacional, drivers e a décadas de bibliotecas C. |
| **Portabilidade** | Compila para praticamente qualquer plataforma: servidores, desktops, consoles, mobile, microcontroladores. |
| **Ecossistema de alta performance** | Boost, Abseil, Folly, Eigen, Qt, Unreal Engine, LLVM, CUDA: bibliotecas maduras e otimizadas. |
| **Longevidade e compatibilidade** | Código de décadas continua compilando; padrões ISO dão estabilidade e evolução contínua. |
| **Base de sistemas críticos** | Chrome, Firefox, HotSpot (a JVM), V8, MySQL, MongoDB, ClickHouse, RocksDB, PyTorch e TensorFlow têm núcleo em C++. |

### Desvantagens
| Desvantagem | Contexto / mitigação |
|-------------|----------------------|
| **Segurança de memória** | Buffer overflows, use-after-free e data races são fontes clássicas de vulnerabilidades. Mitigação: C++ moderno (smart pointers, `std::span`, containers), sanitizers (ASan, UBSan, TSan), análise estática e hardening do compilador. |
| **Complexidade da linguagem** | 40+ anos de recursos acumulados; existem muitas formas de fazer a mesma coisa, e várias delas são armadilhas. Exige guidelines claras (C++ Core Guidelines). |
| **Undefined Behavior** | Bugs sutis que aparecem só com certas otimizações ou compiladores. Difícil de diagnosticar sem sanitizers e testes. |
| **Build e gerenciamento de dependências** | Não há gerenciador de pacotes nem build system padrão. CMake, vcpkg e Conan ajudam, mas a configuração ainda é trabalhosa. |
| **Tempo de compilação** | Headers e templates deixam builds grandes lentos. Mitigações: modules (C++20), precompiled headers, ccache, builds distribuídos. |
| **Mensagens de erro** | Erros de template podem ser longos e confusos; concepts melhoraram, mas não resolveram tudo. |
| **Produtividade** | Para serviços de negócio típicos, Java, Kotlin, Go ou Python entregam mais rápido com menos risco. |
| **Pressão regulatória e de mercado** | Governos e empresas recomendam linguagens memory-safe para código novo, o que empurra parte dos projetos para Rust. |

### Por que é relevante
1. **Está na base de tudo**: browsers, sistemas operacionais (partes do Windows, macOS), a própria JVM, V8, compiladores (LLVM) e bancos de dados são C++. Quase toda linguagem de alto nível roda em cima de C ou C++.
2. **Performance ainda é diferencial competitivo**: em trading de alta frequência, engines de jogos, bancos de dados e processamento de vídeo, microssegundos viram dinheiro.
3. **Infraestrutura de IA**: PyTorch, TensorFlow, ONNX Runtime, llama.cpp e kernels CUDA são C++. Python é a interface; C++ é onde o trabalho pesado acontece.
4. **A linguagem continua evoluindo**: C++20 trouxe concepts, ranges, coroutines e modules; C++23 e C++26 seguem adicionando recursos como `std::expected`, reflexão estática e contracts, além de esforços de *safety profiles* e hardening da biblioteca padrão.
5. **Enorme base instalada**: bilhões de linhas de código C++ em produção precisam ser mantidas, evoluídas e integradas por décadas.

### Quando escolher C++
**C++ costuma ser uma boa escolha quando:**
- latência e throughput são requisitos centrais e pausas de GC não são aceitáveis;
- o projeto precisa de controle fino de memória, hardware ou layout de dados (jogos, HFT, bancos de dados, engines);
- o domínio já é dominado por C++ (Unreal, CUDA, Qt, sistemas embarcados, HPC);
- é preciso integrar ou estender uma base de código C++ existente.

**Vale considerar alternativas quando:**
- o código novo tem requisitos fortes de segurança de memória e a performance é parecida (Rust);
- é um serviço de negócio, API ou microsserviço comum (Java, Kotlin, Go);
- o foco é ciência de dados ou prototipação rápida (Python, usando bibliotecas C++ por baixo);
- o time não tem experiência sólida em C++ e não há como investir em ferramentas e revisão.

**Armadilhas comuns:**
- Escrever "C com classes" em 2026: `new`/`delete` manuais, ponteiros crus com posse, arrays C em vez de containers.
- Não rodar sanitizers e análise estática no CI.
- Otimizar sem medir: usar profiler (perf, VTune, Tracy) antes de complicar o código.
- Abusar de herança e de template metaprogramming quando uma solução simples resolve.
- Ignorar warnings do compilador (compile com `-Wall -Wextra -Werror` ou equivalente).

### Referências
- [cppreference.com](https://en.cppreference.com/)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/)
- [isocpp.org: Standard C++ Foundation](https://isocpp.org/)
- [Compiler Explorer](https://godbolt.org/)
- *A Tour of C++*, Bjarne Stroustrup
- *Effective Modern C++*, Scott Meyers
- *C++ Concurrency in Action*, Anthony Williams

---

## English

### Context
C++ was created by **Bjarne Stroustrup** at Bell Labs starting in 1979, initially as *"C with Classes"*, and renamed C++ in 1983. The idea was to combine C's efficiency and hardware access with high-level abstractions (classes, and later templates) at no extra runtime cost.

The first ISO standard was released in 1998 (C++98). After a long gap, **C++11** modernized the language (move semantics, smart pointers, lambdas, `auto`), and since then a new standard has shipped every three years: C++14, 17, 20, 23, and 26. Today C++ remains the foundation of systems where performance and resource control are non-negotiable: game engines, databases, browsers, high-frequency trading, ML runtimes, and embedded systems.

### Key concepts
- **Zero-cost abstractions**: *"What you don't use, you don't pay for; what you do use, you couldn't hand-code any better."* Abstractions should compile to code as efficient as the manual equivalent.
- **Native compilation**: No VM and no garbage collector; the compiler (GCC, Clang, MSVC) generates machine code optimized for the target architecture.
- **RAII (Resource Acquisition Is Initialization)**: Resources (memory, files, locks, sockets) are released deterministically in the destructor when an object goes out of scope.
- **Smart pointers**: `std::unique_ptr` (exclusive ownership) and `std::shared_ptr` (shared ownership) replace manual `new`/`delete` in modern C++.
- **Move semantics**: Transferring resources without copying (`std::move`, rvalue references), essential for performance with large types.
- **Templates and concepts**: Generic programming resolved at compile time; *concepts* (C++20) constrain templates and improve error messages.
- **Undefined Behavior (UB)**: Invalid operations (out-of-bounds access, use-after-free, signed integer overflow) have no defined behavior, and the compiler may optimize assuming they never happen.
- **ISO standardization**: The language is defined by a committee (WG21), with multiple independent compilers and implementations.

### Advantages
| Advantage | Why it matters |
|-----------|----------------|
| **Maximum, predictable performance** | No GC pauses or JIT warm-up. Deterministic latency, essential for trading, games, audio, and real-time systems. |
| **Fine-grained control of memory and hardware** | Data layout, allocation, alignment, cache locality, SIMD: you can optimize at whatever level the problem demands. |
| **Zero-cost abstractions** | Templates, RAII, and classes allow expressive code without sacrificing efficiency. |
| **C interop** | Direct access to operating system APIs, drivers, and decades of C libraries. |
| **Portability** | Compiles to virtually any platform: servers, desktops, consoles, mobile, microcontrollers. |
| **High-performance ecosystem** | Boost, Abseil, Folly, Eigen, Qt, Unreal Engine, LLVM, CUDA: mature, optimized libraries. |
| **Longevity and compatibility** | Decades-old code still compiles; ISO standards provide stability and continuous evolution. |
| **Foundation of critical systems** | Chrome, Firefox, HotSpot (the JVM), V8, MySQL, MongoDB, ClickHouse, RocksDB, PyTorch, and TensorFlow have C++ cores. |

### Disadvantages
| Disadvantage | Context / mitigation |
|--------------|----------------------|
| **Memory safety** | Buffer overflows, use-after-free, and data races are classic sources of vulnerabilities. Mitigation: modern C++ (smart pointers, `std::span`, containers), sanitizers (ASan, UBSan, TSan), static analysis, and compiler hardening. |
| **Language complexity** | 40+ years of accumulated features; there are many ways to do the same thing, and several of them are traps. Requires clear guidelines (C++ Core Guidelines). |
| **Undefined Behavior** | Subtle bugs that only show up with certain optimizations or compilers. Hard to diagnose without sanitizers and tests. |
| **Build and dependency management** | No standard package manager or build system. CMake, vcpkg, and Conan help, but setup is still laborious. |
| **Compilation time** | Headers and templates make large builds slow. Mitigations: modules (C++20), precompiled headers, ccache, distributed builds. |
| **Error messages** | Template errors can be long and confusing; concepts improved this, but did not solve everything. |
| **Productivity** | For typical business services, Java, Kotlin, Go, or Python deliver faster with less risk. |
| **Regulatory and market pressure** | Governments and companies recommend memory-safe languages for new code, pushing some projects toward Rust. |

### Why it matters
1. **It is at the foundation of everything**: Browsers, operating systems (parts of Windows and macOS), the JVM itself, V8, compilers (LLVM), and databases are C++. Almost every high-level language runs on top of C or C++.
2. **Performance is still a competitive edge**: In high-frequency trading, game engines, databases, and video processing, microseconds turn into money.
3. **AI infrastructure**: PyTorch, TensorFlow, ONNX Runtime, llama.cpp, and CUDA kernels are C++. Python is the interface; C++ is where the heavy lifting happens.
4. **The language keeps evolving**: C++20 brought concepts, ranges, coroutines, and modules; C++23 and C++26 keep adding features such as `std::expected`, static reflection, and contracts, along with efforts around *safety profiles* and standard library hardening.
5. **A huge installed base**: Billions of lines of C++ in production need to be maintained, evolved, and integrated for decades.

### When to choose C++
**C++ is usually a good choice when:**
- latency and throughput are core requirements and GC pauses are unacceptable;
- the project needs fine-grained control over memory, hardware, or data layout (games, HFT, databases, engines);
- the domain is already dominated by C++ (Unreal, CUDA, Qt, embedded systems, HPC);
- you need to integrate with or extend an existing C++ codebase.

**Consider alternatives when:**
- new code has strong memory-safety requirements and comparable performance is achievable (Rust);
- it is a typical business service, API, or microservice (Java, Kotlin, Go);
- the focus is data science or fast prototyping (Python, using C++ libraries underneath);
- the team lacks solid C++ experience and cannot invest in tooling and review.

**Common pitfalls:**
- Writing "C with classes" in 2026: manual `new`/`delete`, owning raw pointers, C arrays instead of containers.
- Not running sanitizers and static analysis in CI.
- Optimizing without measuring: use a profiler (perf, VTune, Tracy) before making the code more complex.
- Overusing inheritance and template metaprogramming when a simple solution would do.
- Ignoring compiler warnings (compile with `-Wall -Wextra -Werror` or the equivalent).

### References
- [cppreference.com](https://en.cppreference.com/)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/)
- [isocpp.org: Standard C++ Foundation](https://isocpp.org/)
- [Compiler Explorer](https://godbolt.org/)
- *A Tour of C++*, Bjarne Stroustrup
- *Effective Modern C++*, Scott Meyers
- *C++ Concurrency in Action*, Anthony Williams
