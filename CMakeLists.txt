cmake_minimum_required(VERSION 3.12)
include(CMakeDependentOption)
set(APP_BUILD_OPTIONS)
set(APP_LINK_LIB)
set(CLI_LINK_LIB)

# Zig makes cross-compiling easier and ignores glibc version issue
# target can be: x86_64-macos-none, x86_64-macos.11.0-none, x86_64-windows-gnu, mipsel-linux-musl, x86_64-linux-gnu.2.17, ...
# Using `zig targets | jq .libc` to get the list of targets
option(ZIG_TARGET "Setting a target and using `zig cc/c++` to compile" OFF)

# example: -DZIG_NIGHTLY="0.13.0-dev.46+3648d7df1"
# You can find the latest version by: `curl -sS https://ziglang.org/download/index.json | jq .master.version`
option(ZIG_NIGHTLY "Download nightly build instead of release version if zig is not in the system path" OFF)

# Is not recommended to use pcap++ provided by package manager, because we need immediate_mode
option(USE_SYSTEM_PCAPPLUSPLUS "Find pcapp++ from system path" OFF)
cmake_dependent_option(USE_SYSTEM_PCAP "Find pcap from system path" ON "NOT USE_SYSTEM_PCAPPLUSPLUS" ON)

option(BUILD_CLI "build command line tool" ON)
option(BUILD_TEST "build test" OFF)

if (ZIG_TARGET)
    include(${CMAKE_SOURCE_DIR}/cmake/zig.cmake)
    set(CMAKE_TOOLCHAIN_FILE "${CMAKE_SOURCE_DIR}/cmake/zig.toolchain.cmake")
    add_compile_options(--target=${ZIG_TARGET})
    message(STATUS "Custom compile options: ${ZIG_COMPILE_OPTION}")
    if (ZIG_COMPILE_OPTION)
        add_compile_options(${ZIG_COMPILE_OPTION})
        add_link_options(${ZIG_COMPILE_OPTION})
    endif ()
endif ()

if (CMAKE_BUILD_TYPE STREQUAL Debug)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address,undefined")
    list(APPEND APP_BUILD_OPTIONS -DDEBUG)
endif ()

project(pppwn)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS ON)
if (MINGW)
    set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static" CACHE STRING "" FORCE)
    add_compile_definitions(-D_Post_invalid_=) # Fix incompatibility with old npcap
    add_compile_options("-Wno-macro-redefined" "-Wno-implicit-function-declaration")
    list(APPEND CLI_LINK_LIB winmm)
endif ()

if (USE_SYSTEM_PCAPPLUSPLUS)
    find_package(PcapPlusPlus REQUIRED)
    list(APPEND APP_LINK_LIB PcapPlusPlus::Pcap++)
else ()
    set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)
    set(BUILD_SHARED_LIBS OFF)
    if(NOT USE_SYSTEM_PCAP)
        if (MINGW)
            # Fix a pcap CMakeLists bug
            set(BUILD_SHARED_LIBS ON)
            # Fix zig cross-compiling issue
            set(CMAKE_USE_PTHREADS_INIT ON)
        endif ()
        include(FetchContent)
        set(DISABLE_DBUS ON)
        set(DISABLE_RDMA ON)
        set(BUILD_WITH_LIBNL OFF)
        FetchContent_Declare(pcap GIT_REPOSITORY https://github.com/the-tcpdump-group/libpcap.git
                GIT_TAG libpcap-1.10.4)
        FetchContent_MakeAvailable(pcap)
        file(COPY ${pcap_SOURCE_DIR}/pcap DESTINATION ${CMAKE_BINARY_DIR}/vendor/include)
        include_directories("${CMAKE_BINARY_DIR}/vendor/include")
        set(PCAP_INCLUDE_DIR "${CMAKE_BINARY_DIR}/vendor/include/pcap")
        set(PCAP_LIBRARY pcap_static)
        # Add some variables to let PcapPlusPlus know the libpcap path
        add_library(PCAP::PCAP INTERFACE IMPORTED)
        target_link_libraries(PCAP::PCAP INTERFACE pcap_static)
        target_include_directories(PCAP::PCAP INTERFACE "${PCAP_INCLUDE_DIR}")
        target_compile_definitions(PCAP::PCAP INTERFACE "_BSD_SOURCE" "_DEFAULT_SOURCE")
        set(PCAP_NEEDS_THREADS ON)
        set(HAVE_PCAP_IMMEDIATE_MODE ON)
        set(HAVE_PCAP_DIRECTION ON)
    endif ()
    set(BUILD_SHARED_LIBS OFF)
    set(PCAPPP_ENABLE_PCAP_SET_DIRECTION ON)
    set(PCAPPP_ENABLE_PCAP_IMMEDIATE_MODE ON)
    include(FetchContent)
    FetchContent_Declare(PcapPlusPlus GIT_REPOSITORY https://github.com/seladb/PcapPlusPlus.git
            GIT_TAG v23.09)
    FetchContent_MakeAvailable(PcapPlusPlus)
    list(APPEND APP_LINK_LIB Pcap++)
endif ()

add_library(${PROJECT_NAME}_static STATIC src/exploit.cpp src/packet.cpp)
set_target_properties(${PROJECT_NAME}_static PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME}_static PUBLIC ${APP_LINK_LIB})
target_compile_options(${PROJECT_NAME}_static PUBLIC ${APP_BUILD_OPTIONS})
target_include_directories(${PROJECT_NAME}_static PUBLIC include)

if (BUILD_CLI)
    add_executable(${PROJECT_NAME} src/main.cpp)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_static ${CLI_LINK_LIB})
endif ()

if (BUILD_TEST)
    add_subdirectory(tests)
endif ()