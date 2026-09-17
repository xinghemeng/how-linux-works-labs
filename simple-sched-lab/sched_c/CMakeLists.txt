add_executable(sched_c main.c)

set_target_properties(sched_c PROPERTIES
    C_STANDARD 11
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS ON
)

target_compile_options(sched_c PRIVATE -Wall -Wextra)
