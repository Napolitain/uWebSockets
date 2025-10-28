cmake_minimum_required(VERSION 3.15)
project(uWebSockets VERSION 20.0.0 LANGUAGES C CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Set C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Options matching Makefile flags
option(WITH_OPENSSL "Enable OpenSSL 1.1+ support" OFF)
option(WITH_WOLFSSL "Enable WolfSSL 4.2.0 support" OFF)
option(WITH_BORINGSSL "Enable BoringSSL support" OFF)
option(WITH_ZLIB "Enable zlib support (permessage-deflate)" ON)
option(WITH_LIBDEFLATE "Enable libdeflate as fast path to zlib" OFF)
option(WITH_LIBUV "Enable libuv event-loop" OFF)
option(WITH_ASIO "Enable ASIO event-loop" OFF)
option(WITH_GCD "Enable GCD event-loop" OFF)
option(WITH_IO_URING "Enable io_uring event-loop" OFF)
option(WITH_QUIC "Enable experimental Http3 support" OFF)
option(WITH_PROXY "Enable PROXY Protocol v2 support" OFF)
option(WITH_LTO "Enable Link Time Optimization" ON)
option(WITH_ASAN "Enable AddressSanitizer" OFF)
option(BUILD_EXAMPLES "Build example programs" ON)

# Install directories
include(GNUInstallDirs)
set(UWEBSOCKETS_INSTALL_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/uWebSockets")

# Build uSockets library inline (since it's a submodule without CMakeLists.txt)
set(USOCKETS_SOURCES
    uSockets/src/bsd.c
    uSockets/src/context.c
    uSockets/src/loop.c
    uSockets/src/socket.c
    uSockets/src/udp.c
    uSockets/src/eventing/epoll_kqueue.c
    uSockets/src/eventing/gcd.c
    uSockets/src/eventing/libuv.c
    uSockets/src/crypto/openssl.c
    uSockets/src/quic.c
    uSockets/src/io_uring/io_context.c
    uSockets/src/io_uring/io_loop.c
    uSockets/src/io_uring/io_socket.c
)

# Add C++ sources if using OpenSSL or BoringSSL
if(WITH_OPENSSL OR WITH_BORINGSSL)
    list(APPEND USOCKETS_SOURCES uSockets/src/crypto/sni_tree.cpp)
endif()

# Add ASIO C++ source if using ASIO
if(WITH_ASIO)
    list(APPEND USOCKETS_SOURCES uSockets/src/eventing/asio.cpp)
endif()

# Create uSockets static library
add_library(uSockets STATIC ${USOCKETS_SOURCES})
add_library(uSockets::uSockets ALIAS uSockets)

# Set include directories for uSockets
target_include_directories(uSockets
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/uSockets/src>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/uSockets>
)

# Compiler flags for uSockets
target_compile_options(uSockets PRIVATE -O3)

# Add LTO if enabled and not MSVC
if(WITH_LTO AND NOT MSVC)
    target_compile_options(uSockets PRIVATE -flto)
endif()

# Configure uSockets based on options
if(WITH_BORINGSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_OPENSSL)
    target_include_directories(uSockets PUBLIC uSockets/boringssl/include)
    target_compile_options(uSockets PUBLIC -pthread)
    # Note: BoringSSL libraries need to be built separately with "make boringssl"
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/uSockets/boringssl/build/ssl/libssl.a")
        target_link_libraries(uSockets PUBLIC
            ${CMAKE_CURRENT_SOURCE_DIR}/uSockets/boringssl/build/ssl/libssl.a
            ${CMAKE_CURRENT_SOURCE_DIR}/uSockets/boringssl/build/crypto/libcrypto.a
            pthread
        )
    endif()
elseif(WITH_OPENSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_OPENSSL)
    target_link_libraries(uSockets PUBLIC ssl crypto)
elseif(WITH_WOLFSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_WOLFSSL)
    target_include_directories(uSockets PUBLIC /usr/local/include)
    target_link_libraries(uSockets PUBLIC wolfssl)
else()
    target_compile_definitions(uSockets PUBLIC LIBUS_NO_SSL)
endif()

# Event loop configuration for uSockets
if(WITH_IO_URING)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_IO_URING)
    # Link liburing if available
    if(EXISTS "/usr/lib/liburing.a")
        target_link_libraries(uSockets PUBLIC /usr/lib/liburing.a)
    endif()
endif()

if(WITH_LIBUV)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_LIBUV)
    target_link_libraries(uSockets PUBLIC uv)
endif()

if(WITH_ASIO)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_ASIO)
    target_compile_options(uSockets PUBLIC -pthread)
    target_link_libraries(uSockets PUBLIC pthread)
endif()

if(WITH_GCD)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_GCD)
    if(APPLE)
        target_link_libraries(uSockets PUBLIC "-framework CoreFoundation")
    endif()
endif()

# QUIC configuration for uSockets
if(WITH_QUIC)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_QUIC)
    target_include_directories(uSockets PUBLIC uSockets/lsquic/include)
    target_compile_options(uSockets PUBLIC -pthread)
    target_link_libraries(uSockets PUBLIC pthread z m)
    # Link lsquic if available
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/uSockets/lsquic/src/liblsquic/liblsquic.a")
        target_link_libraries(uSockets PUBLIC
            ${CMAKE_CURRENT_SOURCE_DIR}/uSockets/lsquic/src/liblsquic/liblsquic.a)
    endif()
endif()

# Sanitizer support for uSockets
if(WITH_ASAN)
    target_compile_options(uSockets PUBLIC -fsanitize=address -g)
    target_link_options(uSockets PUBLIC -fsanitize=address)
endif()

# Create uWebSockets interface library (header-only)
add_library(uWebSockets INTERFACE)
add_library(uWebSockets::uWebSockets ALIAS uWebSockets)

# Set include directories for uWebSockets
target_include_directories(uWebSockets INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<INSTALL_INTERFACE:${UWEBSOCKETS_INSTALL_INCLUDEDIR}>
)

# Link uSockets
target_link_libraries(uWebSockets INTERFACE uSockets::uSockets)

# Configure compile definitions based on options
if(WITH_PROXY)
    target_compile_definitions(uWebSockets INTERFACE UWS_WITH_PROXY)
endif()

if(NOT WITH_ZLIB)
    target_compile_definitions(uWebSockets INTERFACE UWS_NO_ZLIB)
endif()

if(WITH_LIBDEFLATE)
    target_compile_definitions(uWebSockets INTERFACE UWS_USE_LIBDEFLATE)
    target_include_directories(uWebSockets INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libdeflate>
    )
endif()

# Build examples if requested
if(BUILD_EXAMPLES)
    # List of example files
    set(EXAMPLE_FILES
        SecureGzipFileServer
        Precompress
        EchoBody
        HelloWorldThreaded
        Http3Server
        Broadcast
        HelloWorld
        Crc32
        ServerName
        EchoServer
        BroadcastingEchoServer
        UpgradeSync
        UpgradeAsync
        ParameterRoutes
    )

    # Common compiler flags for examples
    set(EXAMPLE_CXX_FLAGS)
    list(APPEND EXAMPLE_CXX_FLAGS -march=native -O3 -Wpedantic -Wall -Wextra -Wsign-conversion -Wconversion)

    # Add LTO if enabled
    if(WITH_LTO AND NOT MSVC)
        list(APPEND EXAMPLE_CXX_FLAGS -flto=auto)
    endif()

    # Add sanitizers if enabled
    if(WITH_ASAN)
        list(APPEND EXAMPLE_CXX_FLAGS -fsanitize=address -g)
    endif()

    # Additional flags for specific backends
    if(WITH_ASIO)
        list(APPEND EXAMPLE_CXX_FLAGS -pthread)
    endif()

    # Create example executables
    foreach(EXAMPLE ${EXAMPLE_FILES})
        add_executable(${EXAMPLE} examples/${EXAMPLE}.cpp)
        target_link_libraries(${EXAMPLE} PRIVATE uWebSockets::uWebSockets)
        target_compile_options(${EXAMPLE} PRIVATE ${EXAMPLE_CXX_FLAGS})
        
        # Link additional libraries based on options
        if(WITH_ZLIB AND NOT WITH_LIBDEFLATE)
            target_link_libraries(${EXAMPLE} PRIVATE z)
        endif()
        
        if(WITH_LIBDEFLATE)
            # Assuming libdeflate is built and available
            target_link_libraries(${EXAMPLE} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/libdeflate/libdeflate.a)
        endif()
        
        if(WITH_LIBUV)
            target_link_libraries(${EXAMPLE} PRIVATE uv)
        endif()
        
        if(WITH_ASIO)
            target_link_libraries(${EXAMPLE} PRIVATE pthread)
        endif()
        
        if(WITH_ASAN)
            target_link_libraries(${EXAMPLE} PRIVATE asan)
        endif()
        
        if(WITH_QUIC)
            target_link_libraries(${EXAMPLE} PRIVATE pthread z m)
            # Add lsquic library if available
            if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/uSockets/lsquic/src/liblsquic/liblsquic.a")
                target_link_libraries(${EXAMPLE} PRIVATE 
                    ${CMAKE_CURRENT_SOURCE_DIR}/uSockets/lsquic/src/liblsquic/liblsquic.a)
            endif()
        endif()
    endforeach()
endif()

# Install targets
install(DIRECTORY src/
    DESTINATION ${UWEBSOCKETS_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h"
)

# Install uSockets library and headers
install(TARGETS uSockets
    EXPORT uSocketsTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY uSockets/src/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/uSockets
    FILES_MATCHING PATTERN "*.h"
)

install(EXPORT uSocketsTargets
    FILE uSocketsTargets.cmake
    NAMESPACE uSockets::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uSockets
)

# Export targets for use in other CMake projects
install(TARGETS uWebSockets
    EXPORT uWebSocketsTargets
    INCLUDES DESTINATION ${UWEBSOCKETS_INSTALL_INCLUDEDIR}
)

install(EXPORT uWebSocketsTargets
    FILE uWebSocketsTargets.cmake
    NAMESPACE uWebSockets::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uWebSockets
)

# Create and install package config file
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/uWebSocketsConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/uWebSocketsConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/uWebSocketsConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uWebSockets
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/uWebSocketsConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/uWebSocketsConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/uWebSockets
)
