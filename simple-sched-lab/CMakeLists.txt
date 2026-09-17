cmake_minimum_required(VERSION 3.16)

# C is enabled here so a C-only configure does not require a C++ toolchain.
# sched_cpp enables CXX from its own option below.
project(simple-sched-lab VERSION 0.1.0 LANGUAGES C)

option(SIMPLE_SCHED_BUILD_C 
        "Build the C implementation (target sched_c)" ON)
option(SIMPLE_SCHED_BUILD_CPP
       "Build the C++17 implementation (target sched_cpp)" ON)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT SIMPLE_SCHED_BUILD_C AND NOT SIMPLE_SCHED_BUILD_CPP)
    message(FATAL_ERROR
        "At least one of SIMPLE_SCHED_BUILD_C or SIMPLE_SCHED_BUILD_CPP must be ON")
endif()

if(SIMPLE_SCHED_BUILD_C)
    add_subdirectory(sched_c)
endif()

if(SIMPLE_SCHED_BUILD_CPP)
    enable_language(CXX)
    add_subdirectory(sched_cpp)
endif()
