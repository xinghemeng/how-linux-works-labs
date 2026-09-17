add_executable(sched_cpp main.cpp)

set_target_properties(sched_cpp PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
)

target_compile_options(sched_cpp PRIVATE -Wall -Wextra)
