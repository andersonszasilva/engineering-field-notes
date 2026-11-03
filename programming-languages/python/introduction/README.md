# Python: Introdução

**Data:** 2026-11-03
**Área:** programming-languages / python
**Tags:** python, cpython, dados, ml, automação, fundamentos, trade-offs

[Português](#português) | [English](#english)

---

## Português

### Contexto
Python foi criado por **Guido van Rossum** no CWI (Holanda), com desenvolvimento iniciado no fim de 1989 e primeiro lançamento público em 1991. A filosofia sempre foi **legibilidade**: código que se lê quase como pseudocódigo, resumida no *Zen of Python* (`import this`): *"Readability counts"*, *"There should be one obvious way to do it"*.

A migração do Python 2 para o Python 3 (lançado em 2008) foi longa e dolorosa, e só terminou de fato com o fim do suporte ao Python 2 em 2020. Desde então a linguagem vive uma fase de evolução acelerada: releases anuais, ganhos grandes de performance (projeto *Faster CPython*) e o início do fim do GIL. Hoje Python é a linguagem dominante em ciência de dados, machine learning e IA, além de muito usada em backend, automação, scripting e ensino.

### Conceitos-chave
- **Interpretada e dinâmica**: o CPython (implementação de referência) compila o código para bytecode e o executa numa VM; tipos são verificados em tempo de execução.
- **Tipagem dinâmica e forte**: variáveis não têm tipo fixo, mas objetos têm (`"1" + 1` gera erro). **Type hints** (PEP 484) permitem checagem estática opcional com mypy ou pyright.
- **GIL (Global Interpreter Lock)**: no CPython tradicional, só uma thread executa bytecode Python por vez. Desde o 3.13 existe um build **free-threaded** (sem GIL), oficialmente suportado a partir do 3.14.
- **Gerenciamento de memória**: contagem de referências mais um garbage collector para ciclos.
- **Concorrência**: `threading` para I/O, `multiprocessing` para CPU, `asyncio` para I/O assíncrono com `async`/`await`.
- **"Baterias incluídas"**: biblioteca padrão extensa (`json`, `pathlib`, `sqlite3`, `http`, `unittest`, `concurrent.futures` etc.).
- **Extensões nativas**: bibliotecas como NumPy, pandas e PyTorch são escritas em C, C++, Rust ou CUDA; Python é a "cola" que orquestra código de alta performance.
- **Cadência de releases**: uma versão nova por ano, em outubro (PEP 602), com cerca de 5 anos de suporte cada.

### Vantagens
| Vantagem | Por que importa |
|----------|-----------------|
| **Produtividade e legibilidade** | Pouco código para resolver muito; ideal para prototipar, validar ideias e entregar rápido. |
| **Ecossistema de dados e IA** | NumPy, pandas, Polars, scikit-learn, PyTorch, JAX, Hugging Face, Jupyter: é o padrão de fato em ciência de dados, ML e LLMs. |
| **Curva de aprendizado suave** | Usada no ensino e por profissionais de outras áreas (cientistas, analistas, engenheiros de dados), o que amplia quem consegue contribuir. |
| **Versatilidade** | Backend (Django, FastAPI, Flask), automação, scripting, DevOps, testes, pipelines de dados e ferramentas internas. |
| **Interop com código nativo** | C extensions, Cython, pybind11, PyO3 (Rust) e `ctypes` permitem mover só os trechos críticos para código nativo. |
| **Biblioteca padrão e PyPI** | Centenas de milhares de pacotes: quase sempre já existe uma biblioteca para o problema. |
| **Tooling moderno** | `uv` (gerenciamento de pacotes e ambientes muito rápido), `ruff` (lint e format), mypy/pyright e pytest elevaram muito a experiência de desenvolvimento. |
| **Comunidade e mercado** | Uma das linguagens mais populares do mundo, com enorme volume de conteúdo e profissionais. |

### Desvantagens
| Desvantagem | Contexto / mitigação |
|-------------|----------------------|
| **Performance em CPU** | Python puro é bem mais lento que linguagens compiladas. Mitigações: bibliotecas nativas (NumPy, Polars), Cython, Rust via PyO3, PyPy, e os ganhos do *Faster CPython* e do JIT experimental. |
| **GIL e paralelismo** | No build padrão, threads não executam código Python em paralelo. Mitigações: `multiprocessing`, `asyncio` para I/O, extensões que liberam o GIL e o build free-threaded (ainda amadurecendo no ecossistema). |
| **Erros em tempo de execução** | Tipagem dinâmica deixa passar erros que só aparecem em produção. Mitigação: type hints com checagem no CI, testes e validação com Pydantic. |
| **Empacotamento e ambientes** | Historicamente confuso (pip, virtualenv, conda, poetry, setup.py vs pyproject.toml). O `uv` e o `pyproject.toml` simplificaram bastante. |
| **Consumo de memória** | Objetos Python têm overhead alto; estruturas grandes em Python puro consomem muita memória. Usar arrays NumPy/Arrow, `__slots__` e geradores. |
| **Distribuição** | Não gera um binário único nativamente; distribuir apps para usuários finais exige ferramentas como PyInstaller ou containers. |
| **Manutenção em bases grandes** | Sem disciplina de tipos e arquitetura, bases de código grandes ficam difíceis de refatorar com segurança. |

### Por que é relevante
1. **É a linguagem da IA**: treinamento, inferência, avaliação e orquestração de modelos e agentes acontecem majoritariamente em Python. SDKs de LLMs chegam primeiro (e às vezes só) em Python.
2. **Domina dados**: análise, engenharia de dados (Airflow, Dagster, PySpark, dbt Python models) e notebooks são Python.
3. **Está ficando muito mais rápida**: o 3.11 trouxe ganhos de 10–60%, e as versões seguintes adicionaram um JIT experimental e o build free-threaded, atacando os dois pontos fracos históricos.
4. **Backend moderno**: FastAPI, Django e Pydantic tornam Python uma opção sólida para APIs, principalmente quando o serviço está próximo de dados e ML.
5. **Linguagem franca da engenharia**: scripts de automação, ferramentas de infraestrutura, testes e glue code em praticamente toda empresa de tecnologia.

### Quando escolher Python
**Python costuma ser uma boa escolha quando:**
- o projeto envolve dados, ML, IA ou pesquisa;
- velocidade de desenvolvimento e prototipação importam mais que performance bruta;
- o trabalho é automação, scripting, ferramentas internas ou glue code entre sistemas;
- a API está próxima de modelos e pipelines de dados (FastAPI + Pydantic).

**Vale considerar alternativas quando:**
- o serviço é CPU-bound, de alto throughput e baixa latência (Go, Java, Kotlin, Rust, C++);
- é preciso paralelismo real em CPU com memória compartilhada e o build free-threaded ainda não atende o ecossistema usado;
- o software precisa ser distribuído como binário simples (Go, Rust);
- a base de código é grande, com muitos times, e a organização não consegue impor tipagem e testes (Java, Kotlin, TypeScript).

**Armadilhas comuns:**
- Usar valores mutáveis como default de função (`def f(x=[])`).
- Achar que `threading` acelera código CPU-bound no build padrão.
- Misturar código bloqueante dentro de `async def` e travar o event loop.
- Instalar pacotes no Python do sistema em vez de usar ambientes isolados (`uv`, `venv`).
- Escrever loops em Python puro sobre grandes volumes de dados em vez de usar operações vetorizadas.
- Ignorar type hints em projetos que vão crescer.

### Referências
- [Python documentation](https://docs.python.org/3/)
- [PEP Index](https://peps.python.org/)
- [What's New in Python](https://docs.python.org/3/whatsnew/index.html)
- [Python Packaging User Guide](https://packaging.python.org/)
- [Real Python](https://realpython.com/)
- *Fluent Python*, Luciano Ramalho
- *Effective Python*, Brett Slatkin
- *High Performance Python*, Micha Gorelick & Ian Ozsvald

---

## English

### Context
Python was created by **Guido van Rossum** at CWI (Netherlands), with development starting in late 1989 and the first public release in 1991. Its philosophy has always been **readability**: code that reads almost like pseudocode, summarized in the *Zen of Python* (`import this`): *"Readability counts"*, *"There should be one obvious way to do it"*.

The migration from Python 2 to Python 3 (released in 2008) was long and painful, and only truly ended when Python 2 reached end of life in 2020. Since then the language has been evolving quickly: yearly releases, major performance gains (the *Faster CPython* project), and the beginning of the end of the GIL. Today Python is the dominant language for data science, machine learning, and AI, and is widely used for backend, automation, scripting, and teaching.

### Key concepts
- **Interpreted and dynamic**: CPython (the reference implementation) compiles code to bytecode and runs it on a VM; types are checked at runtime.
- **Dynamic and strong typing**: Variables have no fixed type, but objects do (`"1" + 1` raises an error). **Type hints** (PEP 484) enable optional static checking with mypy or pyright.
- **GIL (Global Interpreter Lock)**: In traditional CPython, only one thread executes Python bytecode at a time. Since 3.13 there is a **free-threaded** build (no GIL), officially supported starting with 3.14.
- **Memory management**: Reference counting plus a garbage collector for cycles.
- **Concurrency**: `threading` for I/O, `multiprocessing` for CPU, `asyncio` for asynchronous I/O with `async`/`await`.
- **"Batteries included"**: An extensive standard library (`json`, `pathlib`, `sqlite3`, `http`, `unittest`, `concurrent.futures`, etc.).
- **Native extensions**: Libraries such as NumPy, pandas, and PyTorch are written in C, C++, Rust, or CUDA; Python is the "glue" that orchestrates high-performance code.
- **Release cadence**: One new version per year, in October (PEP 602), each with about five years of support.

### Advantages
| Advantage | Why it matters |
|-----------|----------------|
| **Productivity and readability** | Little code solves a lot; ideal for prototyping, validating ideas, and shipping fast. |
| **Data and AI ecosystem** | NumPy, pandas, Polars, scikit-learn, PyTorch, JAX, Hugging Face, Jupyter: it is the de facto standard for data science, ML, and LLMs. |
| **Gentle learning curve** | Used in education and by professionals from other fields (scientists, analysts, data engineers), which widens who can contribute. |
| **Versatility** | Backend (Django, FastAPI, Flask), automation, scripting, DevOps, testing, data pipelines, and internal tools. |
| **Native code interop** | C extensions, Cython, pybind11, PyO3 (Rust), and `ctypes` let you move only the critical paths to native code. |
| **Standard library and PyPI** | Hundreds of thousands of packages: there is almost always a library for the problem. |
| **Modern tooling** | `uv` (very fast package and environment management), `ruff` (lint and format), mypy/pyright, and pytest have greatly improved the developer experience. |
| **Community and market** | One of the most popular languages in the world, with a huge amount of content and professionals. |

### Disadvantages
| Disadvantage | Context / mitigation |
|--------------|----------------------|
| **CPU performance** | Pure Python is much slower than compiled languages. Mitigations: native libraries (NumPy, Polars), Cython, Rust via PyO3, PyPy, and the gains from *Faster CPython* and the experimental JIT. |
| **GIL and parallelism** | In the default build, threads do not run Python code in parallel. Mitigations: `multiprocessing`, `asyncio` for I/O, extensions that release the GIL, and the free-threaded build (still maturing across the ecosystem). |
| **Runtime errors** | Dynamic typing lets errors slip through that only appear in production. Mitigation: type hints checked in CI, tests, and validation with Pydantic. |
| **Packaging and environments** | Historically confusing (pip, virtualenv, conda, poetry, setup.py vs pyproject.toml). `uv` and `pyproject.toml` have simplified things considerably. |
| **Memory usage** | Python objects carry high overhead; large structures in pure Python use a lot of memory. Use NumPy/Arrow arrays, `__slots__`, and generators. |
| **Distribution** | It does not natively produce a single binary; shipping apps to end users requires tools such as PyInstaller or containers. |
| **Maintainability at scale** | Without typing and architectural discipline, large codebases become hard to refactor safely. |

### Why it matters
1. **It is the language of AI**: Training, inference, evaluation, and orchestration of models and agents happen mostly in Python. LLM SDKs ship first (and sometimes only) in Python.
2. **It dominates data**: Analysis, data engineering (Airflow, Dagster, PySpark, dbt Python models), and notebooks are Python.
3. **It is getting much faster**: 3.11 brought 10–60% speedups, and later versions added an experimental JIT and the free-threaded build, tackling the two historical weak spots.
4. **Modern backend**: FastAPI, Django, and Pydantic make Python a solid option for APIs, especially when the service sits close to data and ML.
5. **The lingua franca of engineering**: Automation scripts, infrastructure tooling, tests, and glue code in virtually every technology company.

### When to choose Python
**Python is usually a good choice when:**
- the project involves data, ML, AI, or research;
- development speed and prototyping matter more than raw performance;
- the work is automation, scripting, internal tools, or glue code between systems;
- the API sits close to models and data pipelines (FastAPI + Pydantic).

**Consider alternatives when:**
- the service is CPU-bound, high-throughput, and low-latency (Go, Java, Kotlin, Rust, C++);
- you need real CPU parallelism with shared memory and the free-threaded build does not yet support the ecosystem you rely on;
- the software must ship as a simple binary (Go, Rust);
- the codebase is large, with many teams, and the organization cannot enforce typing and tests (Java, Kotlin, TypeScript).

**Common pitfalls:**
- Using mutable values as function defaults (`def f(x=[])`).
- Assuming `threading` speeds up CPU-bound code in the default build.
- Mixing blocking code into `async def` and stalling the event loop.
- Installing packages into the system Python instead of using isolated environments (`uv`, `venv`).
- Writing pure-Python loops over large datasets instead of using vectorized operations.
- Skipping type hints in projects that are going to grow.

### References
- [Python documentation](https://docs.python.org/3/)
- [PEP Index](https://peps.python.org/)
- [What's New in Python](https://docs.python.org/3/whatsnew/index.html)
- [Python Packaging User Guide](https://packaging.python.org/)
- [Real Python](https://realpython.com/)
- *Fluent Python*, Luciano Ramalho
- *Effective Python*, Brett Slatkin
- *High Performance Python*, Micha Gorelick & Ian Ozsvald
