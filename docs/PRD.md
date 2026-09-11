# Product Requirements Document (PRD): mSTAandCC

## 1. Executive Summary
`mSTAandCC` (Mini Static Timing Analyzer & Circuit Compiler) is an in-memory gate-level static timing analysis engine and circuit graph compiler implemented in ISO C++20. It models digital combinatorial circuits as Directed Acyclic Graphs (DAGs) and executes linear-time topological traversals to evaluate propagation delays, detect setup timing violations, and report worst-case critical delay paths.

## 2. Problem Statement & Industrial Context
In electronic design automation (EDA), complex application-specific integrated circuits (ASICs) contain billions of logic gates. Simulating their dynamic transient behavior across every input permutation is computationally intractable. Static Timing Analysis solves this by abstracting logic gates into delay-annotated graph vertices and interconnects into directed edges, validating timing constraints statically without requiring functional vector simulation.

## 3. System Architecture & Capabilities

### 3.1 Structural Netlist Ingestion
The engine performs lexical and syntactic analysis of gate-level circuit topologies.
* **Input Contract (v1.0):** The ingestion engine must strictly support the **ISCAS85 benchmark format (`.bench`)**. This is the universal standard for combinatorial circuit validation (`INPUT(...)`, `OUTPUT(...)`, `gate_id = AND(...)`).
* **Delay Rules:** It must also support a supplementary, simple delay rules file specifying intrinsic delays per gate type (e.g., `AND=1.5ns`, `NOT=0.5ns`, `NAND=1.2ns`).
* **Constraint:** The parser must be built using high-performance C++20 `std::string_view` without relying on heavy external generators like Flex/Bison.

### 3.2 Cache-Conscious Graph Model (DOD)
The graph representation is engineered around **Data-Oriented Design (DOD)** to eliminate pointer chasing.
* **In-Memory Model:** Structure of Arrays (SoA) combined with Flat Compressed Sparse Row (CSR).
    * `std::vector<uint32_t> node_head_offsets` // Starting offset for successors
    * `std::vector<uint32_t> edge_destinations` // Compacted adjacency edges
    * `std::vector<float> intrinsic_delays`     // Contiguous delays for auto-vectorization
    * `std::vector<float> arrival_times`        // Contiguous buffer for AT
    * `std::vector<float> required_times`       // Contiguous buffer for RT
* **Constraint:** The use of `std::vector<Node>` containing pointers or nested vectors per node is strictly forbidden to guarantee 64-byte alignment and maximize L1/L2 cache line utilization.

### 3.3 Topological Sorter & Cycle Detection
Implementation of Kahn's algorithm tracking in-degrees to detect non-combinatorial feedback cycles and produce a linear processing order ($O(V + E)$).

### 3.4 Timing Propagation Engine & Boundary Constraints
The engine calculates propagation delays anchored to strict boundary timing constraints:
* **Primary Inputs (PIs):**
  $AT(PI) = T_{\text{input\_delay}} \quad (\text{Default: } 0.0\,\text{ns})$
* **Primary Outputs (POs):** The required time is anchored to a Virtual Clock with period $T_{\text{clk}}$:
  $RT(PO) = T_{\text{clk}} - T_{\text{output\_delay}}$
* **Forward Pass (Arrival Time):** For any internal node $v$:
  $AT(v) = \max_{u \in \text{pred}(v)} \big(AT(u) + \text{delay}(u \to v)\big)$
* **Backward Pass (Required Time):** For any internal node $u$:
  $RT(u) = \min_{v \in \text{succ}(u)} \big(RT(v) - \text{delay}(u \to v)\big)$
* **Slack Calculation:** $\text{Slack}(v) = RT(v) - AT(v)$. Negative slack denotes a timing violation.

### 3.5 Execution Interface & Decoupling
To ensure the system remains agnostic and adaptable to distributed infrastructure:
* The core engine (`libmsta` static library / header-only core) must be strictly separated from the execution interface.
* **v1.0** utilizes a Command Line Interface (CLI).
* The architecture must guarantee that **v1.1** can attach a network wrapper (e.g., an asynchronous TCP server or gRPC endpoint) without modifying a single line of the graph calculation engine.

### 3.6 Architectural Roadmap (Scope)
* **v1.0 (Current Scope):** Gate-Level DAG. The graph models logic gates as single vertices with intrinsic delay per gate type.
* **v2.0 (Future Design):** Bipartite Pin-to-Pin Timing Graph. The graph models input and output pins independently, supporting asymmetric timing arcs per input (input-to-output timing arcs).

## 4. Non-Functional Constraints
* **Language Standard:** C++20 (`-std=c++20`), zero compiler extensions.
* **Safety & Reliability:** Strict compiler warning flags (`-Wall -Wextra -Wpedantic -Werror`), zero memory leaks verified via AddressSanitizer and UndefinedBehaviorSanitizer.
* **Memory Management:** RAII exclusively. Raw memory allocations (`malloc`/`free`/naked `new`) are forbidden.

## 5. Validation, Testing & Performance Metrics
* **Golden Model Testing:** Functional validation against the public ISCAS85 suite (c432, c880, c1355, c1908, c3540, c7552). The critical path and worst slack results must match the analytical reference baseline.
* **Stress Testing:** Generation of synthetic combinatorial circuits exceeding $10^6$ logic gates.
    * **Throughput:** Execution time for Topological Sort + Forward/Backward passes must be $< 150\,\text{ms}$ for $10^6$ nodes.
    * **Cache-Miss Ratio:** Profiling via `perf stat -e L1-dcache-load-misses,LLC-load-misses` (or Apple Instruments) to quantitatively prove the superiority of the DOD (SoA/CSR) approach over classical Object-Oriented representations.