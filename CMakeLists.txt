cmake_minimum_required(VERSION 3.10)

# ---------------------------------------------------------
# Compiler selection (must be before project())
# ---------------------------------------------------------
option(USE_CLANG "Use clang/clang++ compiler" OFF)
option(USE_GCC "Use gcc/g++ compiler" OFF)

if(USE_CLANG AND USE_GCC)
    message(FATAL_ERROR "Cannot specify both USE_CLANG and USE_GCC.")
elseif(USE_CLANG)
    message(STATUS "Forcing clang compiler")
    set(CMAKE_C_COMPILER clang)
    set(CMAKE_CXX_COMPILER clang++)
elseif(USE_GCC)
    message(STATUS "Forcing GCC compiler")
    set(CMAKE_C_COMPILER gcc)
    set(CMAKE_CXX_COMPILER g++)
else()
    message(STATUS "Using default system compiler")
endif()

project(xcplib VERSION 0.9.2 LANGUAGES C CXX)

# ---------------------------------------------------------
# OPTIONS
# ---------------------------------------------------------
option(BUILD_TEST "Build test targets" ON)
option(BUILD_EXAMPLES "Build example programs" ON)

# ---------------------------------------------------------
# Packages
# ---------------------------------------------------------
find_package(Threads REQUIRED)

# ---------------------------------------------------------
# Platform settings
# ---------------------------------------------------------
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(STATUS "64-bit platform")
else()
    message(STATUS "32-bit platform")
endif()

if(WIN32)
    set(WINDOWS TRUE)
elseif(APPLE)
    set(MACOS TRUE)
elseif(UNIX)
    set(LINUX TRUE)
endif()

# ---------------------------------------------------------
# Standards
# ---------------------------------------------------------
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

if(WIN32)
    set(CMAKE_CXX_STANDARD 20)
else()
    set(CMAKE_CXX_STANDARD 17)
endif()

set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ---------------------------------------------------------
# Warnings
# ---------------------------------------------------------
if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    set(COMMON_WARNING_FLAGS -pedantic -Wall)
elseif(MSVC)
    set(COMMON_WARNING_FLAGS /W3)
endif()

# ---------------------------------------------------------
# Compiler flags
# ---------------------------------------------------------
if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O1 -g -fno-omit-frame-pointer -fno-optimize-sibling-calls -DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O1 -g -fno-omit-frame-pointer -fno-optimize-sibling-calls -DNDEBUG")

    set(CMAKE_C_FLAGS_DEBUG "-O0 -g")
    set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g")
elseif(MSVC)
    set(CMAKE_C_FLAGS_RELEASE "/O2 /Zi /DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELEASE "/O2 /Zi /DNDEBUG")

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "/O2 /Zi /Oy- /DNDEBUG")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "/O2 /Zi /Oy- /DNDEBUG")
endif()

# ---------------------------------------------------------
# Sources
# ---------------------------------------------------------
set(xcplib_SOURCES
    src/xcpAppl.c
    src/xcpLite.c
    src/xcpEthServer.c
    src/xcpEthTl.c
    src/xcpQueue32.c
    src/xcpQueue64.c
    src/a2l.c
    src/persistence.c
    src/platform.c
)

set(PROJECT_ROOT ${CMAKE_CURRENT_LIST_DIR})
message(STATUS "PROJECT_ROOT: ${PROJECT_ROOT}")

# ---------------------------------------------------------
# Library
# ---------------------------------------------------------
add_library(xcplib ${xcplib_SOURCES})
target_include_directories(xcplib PUBLIC "${PROJECT_ROOT}/inc" "${PROJECT_ROOT}/src")

if(COMMON_WARNING_FLAGS)
    target_compile_options(xcplib PRIVATE ${COMMON_WARNING_FLAGS})
endif()

if(WIN32)
    target_compile_definitions(xcplib PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

# ---------------------------------------------------------
# pthread linking (skip for QNX)
# ---------------------------------------------------------
if(CMAKE_SYSTEM_NAME MATCHES "QNX")
    message(STATUS "QNX detected -> skipping Threads::Threads (no -lpthread on QNX)")
else()
    target_link_libraries(xcplib PUBLIC Threads::Threads)
endif()

# ---------------------------------------------------------
# Atomic library
# ---------------------------------------------------------
include(CheckLibraryExists)
check_library_exists(atomic __atomic_store_4 "" HAVE_LIBATOMIC)
if(HAVE_LIBATOMIC)
    message(STATUS "Linking with libatomic")
    target_link_libraries(xcplib PUBLIC atomic)
endif()

# ---------------------------------------------------------
# Examples
# ---------------------------------------------------------
if(BUILD_EXAMPLES)

    # hello_xcp
    add_executable(hello_xcp examples/hello_xcp/src/main.c)
    target_include_directories(hello_xcp PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(hello_xcp PRIVATE xcplib)

    # no_a2l_demo
    add_executable(no_a2l_demo examples/no_a2l_demo/src/main.c)
    target_include_directories(no_a2l_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(no_a2l_demo PRIVATE xcplib)

    # hello_xcp_cpp
    add_executable(hello_xcp_cpp examples/hello_xcp_cpp/src/main.cpp)
    target_include_directories(hello_xcp_cpp PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(hello_xcp_cpp PRIVATE xcplib)

    # c_demo
    add_executable(c_demo examples/c_demo/src/main.c)
    target_include_directories(c_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(c_demo PRIVATE xcplib)

    # cpp_demo
    add_executable(cpp_demo
        examples/cpp_demo/src/main.cpp
        examples/cpp_demo/src/sig_gen.cpp
        examples/cpp_demo/src/lookup.cpp
    )
    target_include_directories(cpp_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(cpp_demo PRIVATE xcplib)

    # struct_demo
    add_executable(struct_demo examples/struct_demo/src/main.c)
    target_include_directories(struct_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(struct_demo PRIVATE xcplib)

    # multi_thread_demo
    add_executable(multi_thread_demo examples/multi_thread_demo/src/main.c)
    target_include_directories(multi_thread_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(multi_thread_demo PRIVATE xcplib)

    # bpf_demo
    add_executable(bpf_demo examples/bpf_demo/src/main.c)
    target_include_directories(bpf_demo PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(bpf_demo PRIVATE xcplib)

    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        find_library(LIBBPF_LIBRARY bpf)
        if(LIBBPF_LIBRARY)
            target_link_libraries(bpf_demo PRIVATE ${LIBBPF_LIBRARY})
        endif()
    endif()

    # Suffix for non-Windows
    if(NOT WIN32)
        set_target_properties(
            hello_xcp no_a2l_demo hello_xcp_cpp c_demo cpp_demo
            struct_demo multi_thread_demo bpf_demo
            PROPERTIES SUFFIX ".out"
        )
    endif()

endif() # BUILD_EXAMPLES

# ---------------------------------------------------------
# Tests
# ---------------------------------------------------------
if(BUILD_TEST)

    add_executable(type_detection_test_c
        test/type_detection_test/src/main.c
        test/type_detection_test/src/c_version_test.c
    )
    target_include_directories(type_detection_test_c PUBLIC "${PROJECT_ROOT}/inc")

    add_executable(type_detection_test_cpp
        test/type_detection_test/src/main.cpp
        test/type_detection_test/src/cpp_version_test.cpp
    )
    target_include_directories(type_detection_test_cpp PUBLIC "${PROJECT_ROOT}/inc")

    add_executable(a2l_test test/a2l_test/src/main.c)
    target_include_directories(a2l_test PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(a2l_test PRIVATE xcplib)

    add_executable(cal_test test/cal_test/src/main.cpp)
    target_include_directories(cal_test PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(cal_test PRIVATE xcplib)

    add_executable(daq_test test/daq_test/src/main.c)
    target_include_directories(daq_test PUBLIC "${PROJECT_ROOT}/inc")
    target_link_libraries(daq_test PRIVATE xcplib)

    if(NOT WIN32)
        set_target_properties(
            type_detection_test_c type_detection_test_cpp a2l_test
            cal_test daq_test
            PROPERTIES SUFFIX ".out"
        )
    endif()

endif() # BUILD_TEST
