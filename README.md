# mSTAandCC: Mini Static Timing Analyzer & Circuit Compiler

[![CI](https://github.com/antoniofsnascimento/mSTAandCC/actions/workflows/ci.yml/badge.svg)](https://github.com/antoniofsnascimento/mSTAandCC/actions/workflows/ci.yml)

`mSTAandCC` is a high-performance, in-memory Static Timing Analysis (STA) engine and gate-level circuit compiler written in modern **C++20**. Designed with principles derived from production electronic design automation (EDA) workflows, it optimizes cache locality and algorithmic throughput across large-scale logic network traversals.

## Architectural Highlights

* **ISCAS85 Benchmark Ingestion:** Lexical and syntactic parsing of standard `.bench` structural netlists using zero-allocation C++20 `std::string_view` pipelines.
* **Data-Oriented DAG Engine:** The circuit graph strictly avoids Object-Oriented heap-scattered pointers. It implements a Structure of Arrays (SoA) and Flat Compressed Sparse Row (CSR) topology to maximize CPU L1/L2 cache line prefetching.
* **Topological Dependency Resolution:** Implements Kahn's topological sort algorithm to validate acyclic constraints ($O(V + E)$) and schedule forward/backward evaluation passes.
* **Accurate Slack & Critical Path Diagnostics:** Calculates signal Arrival Times (AT), Required Times (RT) anchored to Virtual Clocks, Slack margins, and isolates worst-case delay chains.
* **Engine Decoupling:** The core graph engine (`libmsta`) is completely isolated from the CLI, ensuring future readiness for distributed RPC/gRPC wrappers.
* **Deterministic Resource Management:** Strict adherence to RAII, Zero-Warning compilation policies (`-Wall -Wextra -Werror`), and continuous memory sanitization via ASan/UBSan.

## Building and Testing

### Prerequisites
* CMake 3.20+
* C++20 compliant compiler: Clang 15+ (macOS / Linux) or GCC 12+ (Linux)
* Ninja build system (optional, recommended)

### Build Instructions
```bash
# Configure the build tree with debug symbols and sanitizers enabled
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DENABLE_SANITIZERS=ON

# Compile the static library and executables
cmake --build build --parallel

# Execute unit, functional (ISCAS85), and stress test suites
ctest --test-dir build --output-on-failure