cmake_minimum_required(VERSION 3.10)

# If project is not already defined, define it (for monorepo)
if (NOT CMAKE_PROJECT_NAME)
    project(uWebSockets)
endif()

# Compiler flags
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Set some default CXXFLAGS and LDFLAGS
set(CXXFLAGS "-march=native -O3 -Wpedantic -Wall -Wextra -Wsign-conversion -Wconversion")
set(LDFLAGS "")

# If not building with LTO, remove the flag
option(WITH_LTO "Enable Link-Time Optimization" ON)
if (WITH_LTO)
    set(CXXFLAGS "${CXXFLAGS} -flto")
endif()

# Optionally build with Zlib
option(WITH_ZLIB "Enable Zlib (permessage-deflate)" ON)
if (WITH_ZLIB)
    find_package(ZLIB REQUIRED)
    set(LDFLAGS "${LDFLAGS} ${ZLIB_LIBRARIES}")
else()
    add_definitions(-DUWS_NO_ZLIB)
endif()

# Optionally build with BoringSSL
option(WITH_BORINGSSL "Build with BoringSSL support" OFF)
option(WITH_OPENSSL "Build with OpenSSL support" OFF)
option(WITH_WOLFSSL "Build with WolfSSL support" OFF)

if (WITH_BORINGSSL)
    include_directories(uSockets/boringssl/include)
    set(LDFLAGS "${LDFLAGS} uSockets/boringssl/build/ssl/libssl.a uSockets/boringssl/build/crypto/libcrypto.a")
    add_definitions(-DLIBUS_USE_OPENSSL)
elseif (WITH_OPENSSL)
    find_package(OpenSSL REQUIRED)
    set(LDFLAGS "${LDFLAGS} ${OPENSSL_LIBRARIES}")
elseif (WITH_WOLFSSL)
    set(LDFLAGS "${LDFLAGS} -L/usr/local/lib -lwolfssl")
endif()

# Optionally build with QUIC (Http3)
option(WITH_QUIC "Build with Http3 QUIC support" OFF)
if (WITH_QUIC)
    add_definitions(-DLIBUS_USE_QUIC)
    set(LDFLAGS "${LDFLAGS} -pthread -lz -lm uSockets/lsquic/src/liblsquic/liblsquic.a")
endif()

# Optionally build with libuv or ASIO as event loop
option(WITH_LIBUV "Build with libuv event loop" OFF)
option(WITH_ASIO "Build with ASIO event loop" OFF)

if (WITH_LIBUV)
    find_package(libuv CONFIG REQUIRED)
    set(LDFLAGS "${LDFLAGS} ${LIBUV_LIBRARIES}")
elseif (WITH_ASIO)
    set(CXXFLAGS "${CXXFLAGS} -pthread")
    set(LDFLAGS "${LDFLAGS} -lpthread")
endif()

# Optionally build with AddressSanitizer (ASAN)
option(WITH_ASAN "Build with AddressSanitizer" OFF)
if (WITH_ASAN)
    set(CXXFLAGS "${CXXFLAGS} -fsanitize=address -g")
    set(LDFLAGS "${LDFLAGS} -lasan")
endif()

# Include paths
include_directories(src)
include_directories(uSockets/src)

# uSockets target (as it was in your Makefile)
add_subdirectory(uSockets)

# Add example executables
set(EXAMPLE_FILES "CachingApp" "HelloWorldThreaded" "Http3Server" "Broadcast" "HelloWorld" "Crc32" "ServerName"
        "EchoServer" "BroadcastingEchoServer" "UpgradeSync" "UpgradeAsync" "ParameterRoutes")

foreach(EXAMPLE_FILE IN LISTS EXAMPLE_FILES)
    string(STRIP "${LDFLAGS}" LDFLAGS_STRIPPED)  # Strip leading/trailing whitespace
    add_executable(${EXAMPLE_FILE} examples/${EXAMPLE_FILE}.cpp)
    target_link_libraries(${EXAMPLE_FILE} uSockets ${LDFLAGS_STRIPPED})
    target_compile_options(${EXAMPLE_FILE} PRIVATE ${CXXFLAGS})
endforeach()

# Install target (mimicking the 'install' section of your Makefile)
install(DIRECTORY src/ DESTINATION ${CMAKE_INSTALL_PREFIX}/include/uWebSockets)

# Clean target (CMake provides a clean target natively)
# If additional custom clean behavior is required, you can add commands here.

# Default target is all
add_custom_target(all_examples DEPENDS ${EXAMPLE_FILES})
