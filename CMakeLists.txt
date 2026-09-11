cmake_minimum_required(VERSION 3.20)

project(mSTAandCC
        VERSION 0.1.0
        DESCRIPTION "Mini Static Timing Analysis and Circuit Compiler"
        LANGUAGES CXX
)

# Enforce strict modern C++20 standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Export compile commands for tooling (clangd / linters)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Configuration option for Sanitizers (AddressSanitizer + UndefinedBehaviorSanitizer)
option(ENABLE_SANITIZERS "Enable ASan and UBSan" ON)

# Core Circuit Engine Library Target

add_library(msta_core STATIC
        src/circuit.cpp
        src/netlist_parser.cpp
        src/timing_engine.cpp
)

target_include_directories(msta_core
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
        PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Compiler Warnings and Strict Dialect Enforcement
target_compile_options(msta_core PRIVATE
        -Wall
        -Wextra
        -Wpedantic
        -Wconversion
        -Wshadow
        -Wnon-virtual-dtor
        -Werror
)

# Sanitizers configuration
if(ENABLE_SANITIZERS)
    target_compile_options(msta_core PUBLIC -fsanitize=address,undefined -fno-omit-frame-pointer)
    target_link_options(msta_core PUBLIC -fsanitize=address,undefined)
endif()

# Main Executable CLI Target

add_executable(mSTAandCC
        main.cpp
)

target_link_libraries(mSTAandCC PRIVATE msta_core)

# Testing Target

enable_testing()

add_executable(test_graph
        tests/test_graph.cpp
)

target_link_libraries(test_graph PRIVATE msta_core)

# Register the unit test with CTest
add_test(NAME CoreGraphTests COMMAND test_graph)