cmake_minimum_required(VERSION 3.14)
project(sqlparser LANGUAGES CXX)

# === Options ===
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(RELAXED_BUILD "Build with most compiler warnings deactivated" OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# === Compile Mode ===
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(OPT_FLAG "-g")
    message(STATUS "Building in debug mode")
else()
    set(OPT_FLAG "-O3")
    message(STATUS "Building in release mode (use -DCMAKE_BUILD_TYPE=Debug for debug mode)")
endif()

# === Directories ===
set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(PARSER_DIR ${SRC_DIR}/parser)
set(BIN_DIR ${CMAKE_CURRENT_BINARY_DIR}/bin)

# === Parser Sources ===
set(PARSER_CPP
    ${PARSER_DIR}/bison_parser.cpp
    ${PARSER_DIR}/flex_lexer.cpp
)
set(PARSER_H
    ${PARSER_DIR}/bison_parser.h
    ${PARSER_DIR}/flex_lexer.h
)

# === Library Sources ===
file(GLOB_RECURSE LIB_CPP
    "${SRC_DIR}/*.cpp"
)
list(REMOVE_ITEM LIB_CPP ${PARSER_CPP})

file(GLOB_RECURSE LIB_H
    "${SRC_DIR}/*.h"
)
list(REMOVE_ITEM LIB_H ${PARSER_H})

set(SQLPARSER_SOURCES ${LIB_CPP} ${PARSER_CPP})

# === Compiler Flags ===
set(LIB_CFLAGS ${OPT_FLAG})
if(NOT RELAXED_BUILD)
    set(LIB_CFLAGS "${LIB_CFLAGS} -Wall -Werror")
endif()
if(BUILD_SHARED_LIBS)
    set(LIB_CFLAGS "${LIB_CFLAGS} -fPIC")
endif()

# === Library Target ===
add_library(sqlparser ${SQLPARSER_SOURCES})
target_include_directories(sqlparser PUBLIC ${SRC_DIR})
target_compile_options(sqlparser PRIVATE -Wall -Werror)

# Suppress warnings for generated parser files
# These warnings are common in auto-generated code and can be safely suppressed
set_source_files_properties(
    ${PARSER_DIR}/flex_lexer.cpp
    PROPERTIES COMPILE_FLAGS "-Wno-sign-compare -Wno-unneeded-internal-declaration -Wno-register"
)

# For bison_parser.cpp, suppress warnings that are common in generated parser code
# These warnings don't indicate actual problems in the generated code
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set_source_files_properties(
        ${PARSER_DIR}/bison_parser.cpp
        PROPERTIES COMPILE_FLAGS "-Wno-free-nonheap-object -Wno-unused-but-set-variable"
    )
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set_source_files_properties(
        ${PARSER_DIR}/bison_parser.cpp
        PROPERTIES COMPILE_FLAGS "-Wno-free-nonheap-object"
    )
else()
    # For other compilers, use minimal suppression
    set_source_files_properties(
        ${PARSER_DIR}/bison_parser.cpp
        PROPERTIES COMPILE_FLAGS "-Wno-free-nonheap-object"
    )
endif()

# === Install Rules ===
install(TARGETS sqlparser
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
)
install(DIRECTORY ${SRC_DIR}/ DESTINATION include/hsql FILES_MATCHING PATTERN "*.h")

# === Benchmark Target ===
find_package(benchmark QUIET)
if(benchmark_FOUND)
    file(GLOB_RECURSE BM_CPP "benchmark/*.cpp")
    add_executable(sqlparser_benchmark ${BM_CPP})
    target_include_directories(sqlparser_benchmark PRIVATE ${SRC_DIR})
    target_link_libraries(sqlparser_benchmark PRIVATE sqlparser benchmark::benchmark pthread stdc++fs)
    set_target_properties(sqlparser_benchmark PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${BIN_DIR})
endif()

# === Test Target ===
file(GLOB_RECURSE TEST_CPP "test/*.cpp")
if(TEST_CPP)
    add_executable(sqlparser_tests ${TEST_CPP})
    target_include_directories(sqlparser_tests PRIVATE ${SRC_DIR} test)
    target_link_libraries(sqlparser_tests PRIVATE sqlparser stdc++)
    set_target_properties(sqlparser_tests PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${BIN_DIR})
    add_custom_target(run_tests
        COMMAND bash ${CMAKE_CURRENT_SOURCE_DIR}/test/test.sh
        DEPENDS sqlparser_tests
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
endif()

# === Example Target ===
file(GLOB_RECURSE EXAMPLE_CPP "example/*.cpp")
if(EXAMPLE_CPP)
    add_executable(sqlparser_example ${EXAMPLE_CPP})
    target_include_directories(sqlparser_example PRIVATE ${SRC_DIR})
    target_link_libraries(sqlparser_example PRIVATE sqlparser)
    set_target_properties(sqlparser_example PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${BIN_DIR})
endif()

# === Clean Targets ===
add_custom_target(cleanparser
    COMMAND ${CMAKE_COMMAND} --build . --target clean
    WORKING_DIRECTORY ${PARSER_DIR}
)

# === Format Target (requires astyle) ===
find_program(ASTYLE_EXECUTABLE astyle)
if(ASTYLE_EXECUTABLE)
    file(GLOB_RECURSE ALL_SRC "${SRC_DIR}/*.cpp" "${SRC_DIR}/*.h" "test/*.cpp" "test/*.h" "example/*.cpp" "example/*.h")
    add_custom_target(format
        COMMAND ${ASTYLE_EXECUTABLE} --options=astyle.options ${ALL_SRC}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
endif()