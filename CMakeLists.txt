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

# Add uSockets subdirectory
add_subdirectory(uSockets)

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
