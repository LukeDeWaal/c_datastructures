cmake_minimum_required(VERSION 3.10)
project(sc_signal C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

add_library(
        sc_signal SHARED
        sc_signal.c
        sc_signal.h)

target_include_directories(sc_signal PUBLIC ${CMAKE_CURRENT_LIST_DIR})

if (CMAKE_SYSTEM_NAME MATCHES Windows)
    target_link_libraries(sc_signal -lws2_32)
endif()

if (NOT CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -Wall -Wextra -pedantic -Werror")
endif ()


# --------------------------------------------------------------------------- #
# --------------------- Test Configuration Start ---------------------------- #
# --------------------------------------------------------------------------- #
if (SC_BUILD_TEST)

    include(CTest)
    include(CheckCCompilerFlag)

    if (SC_CLANG_TIDY)
        message(STATUS "Enabled CLANG_TIDY")

        set(CMAKE_C_CLANG_TIDY
                clang-tidy;
                -line-filter=[{"name":"${PROJECT_NAME}.h"},{"name":"${PROJECT_NAME}.c"}];
                -checks=clang-analyzer-*,misc-*,portability-*,bugprone-*,-bugprone-reserved-identifier*,-misc-include-cleaner*;
                -warnings-as-errors=clang-analyzer-*,misc-*,portability-*,bugprone-*;)
    endif ()

    enable_testing()

    add_executable(${PROJECT_NAME}_test signal_test.c sc_signal.c)

    target_compile_options(${PROJECT_NAME}_test PRIVATE -DSC_SIZE_MAX=1400000ul)

    if (CMAKE_SYSTEM_NAME MATCHES Windows)
        target_link_libraries(${PROJECT_NAME}_test -lws2_32)
    endif()

    check_c_source_compiles("
      #include <execinfo.h>
      #include <unistd.h>

      int main(int argc, char **argv) {
          void *array[10];
          size_t size = backtrace(array, 10);
          backtrace_symbols_fd(array, size, STDERR_FILENO);
          return 0;
  }" HAVE_BACKTRACE)

    FIND_LIBRARY(EXECINFO_LIBRARY NAMES execinfo)
    IF (EXECINFO_LIBRARY)
        SET(CMAKE_REQUIRED_LIBRARIES "${EXECINFO_LIBRARY}")
    ENDIF (EXECINFO_LIBRARY)

    if (${HAVE_BACKTRACE})
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DHAVE_BACKTRACE")
    endif ()

    if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND SC_USE_WRAP)
        if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
                "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")

            target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-omit-frame-pointer)
            target_compile_options(${PROJECT_NAME}_test PRIVATE -DSC_HAVE_WRAP -DSC_SIGNAL_TEST)
            target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-builtin)
            target_link_options(${PROJECT_NAME}_test PRIVATE -Wl,--wrap=signal)

        endif ()
    endif ()

    if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
            "${CMAKE_C_COMPILER_ID}" STREQUAL "AppleClang" OR
            "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")

        target_compile_options(${PROJECT_NAME}_test PRIVATE -fno-omit-frame-pointer)

        if (SANITIZER)
            target_compile_options(${PROJECT_NAME}_test PRIVATE -fsanitize=${SANITIZER})
            target_link_options(${PROJECT_NAME}_test PRIVATE -fsanitize=${SANITIZER})
        endif ()
    endif ()


    add_test(NAME ${PROJECT_NAME}_test COMMAND ${PROJECT_NAME}_test)

    SET(MEMORYCHECK_COMMAND_OPTIONS
            "-q --log-fd=2 --trace-children=yes --track-origins=yes       \
           --leak-check=full --show-leak-kinds=all  \
           --error-exitcode=255")

    add_custom_target(valgrind_${PROJECT_NAME} ${CMAKE_COMMAND}
            -E env CTEST_OUTPUT_ON_FAILURE=1
            ${CMAKE_CTEST_COMMAND} -C $<CONFIG>
            --overwrite MemoryCheckCommandOptions=${MEMORYCHECK_COMMAND_OPTIONS}
            --verbose -T memcheck WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

    add_custom_target(check_${PROJECT_NAME} ${CMAKE_COMMAND}
            -E env CTEST_OUTPUT_ON_FAILURE=1
            ${CMAKE_CTEST_COMMAND} -C $<CONFIG> --verbose
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

    # ------------------------- Code Coverage Start ----------------------------- #

    if (${CMAKE_BUILD_TYPE} MATCHES "Coverage")
        if ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
            target_compile_options(${PROJECT_NAME}_test PRIVATE --coverage)
            target_link_libraries(${PROJECT_NAME}_test gcov)
        else ()
            message(FATAL_ERROR "Only GCC is supported for coverage")
        endif ()
    endif ()

    add_custom_target(coverage_${PROJECT_NAME})
    add_custom_command(
            TARGET coverage_${PROJECT_NAME}
            POST_BUILD
            COMMAND lcov --capture --directory .
            --output-file coverage.info --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
            COMMAND lcov --remove coverage.info '/usr/*' '*example*' '*test*'
            --output-file coverage.info --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
            COMMAND lcov --list coverage.info --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
    )

    add_dependencies(coverage_${PROJECT_NAME} check_${PROJECT_NAME})

    # -------------------------- Code Coverage End ------------------------------ #

endif ()

# ----------------------- Test Configuration End ---------------------------- #

