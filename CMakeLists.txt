# ReXGlue SDK - Xbox 360 Recompilation Toolkit
# Modern C++23 cross-platform SDK for recompiling Xbox 360 executables
cmake_minimum_required(VERSION 3.25)

project(ReXGlue
    VERSION 0.1.2
    LANGUAGES C CXX
    DESCRIPTION "Xbox 360 Recompilation SDK"
)

#==========================================================
# Build Options
#==========================================================
option(REXGLUE_BUILD_TESTS       "Build PPC instruction and unit tests" OFF)
option(REXGLUE_ENABLE_SANITIZERS "Enable UndefinedBehaviorSanitizer" OFF)

# Graphics backend selection
# D3D12: Windows only (x64 and ARM64), ON by default on Windows.
# Vulkan: Cross-platform, ON by default on non-Windows platforms.
if(WIN32)
    option(REXGLUE_USE_D3D12  "Enable D3D12 graphics backend" ON)
    option(REXGLUE_USE_VULKAN "Enable Vulkan graphics backend" OFF)
else()
    set(REXGLUE_USE_D3D12 OFF)
    option(REXGLUE_USE_VULKAN "Enable Vulkan graphics backend" ON)
endif()

if(NOT REXGLUE_USE_D3D12 AND NOT REXGLUE_USE_VULKAN)
    message(FATAL_ERROR "At least one graphics backend must be enabled (REXGLUE_USE_D3D12 or REXGLUE_USE_VULKAN)")
endif()

# SDK install support
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Enforce Clang18 compiler (necesario para ARM64 también)
if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    message(FATAL_ERROR "ReXGlue requires Clang compiler. Please set CMAKE_CXX_COMPILER to clang++")
endif()

if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "18.0")
    message(FATAL_ERROR "ReXGlue requires Clang 18 or newer (found ${CMAKE_CXX_COMPILER_VERSION})")
endif()

# if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
#     message(FATAL_ERROR "ReXGlue only supports x64 architecture")
# endif()


# Set C++23 standard
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_DEBUG_POSTFIX d)
set(CMAKE_RELWITHDEBINFO_POSTFIX rd)

# Global compile options 
add_compile_options(
    -fno-strict-aliasing
    -ffp-model=strict
    $<$<CONFIG:DEBUG>:-g>
    $<$<CONFIG:DEBUG>:-O0>
    $<$<CONFIG:RELEASE>:-O3>
    $<$<CONFIG:RELEASE>:-DNDEBUG>
    -fno-char8_t
)

# Platform x86_64 or arm64 support
if(WIN32)
    # Detect windows arch
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|ARM64)$")
        set(REX_PLATFORM "win-arm64")
    else()
        set(REX_PLATFORM "win-amd64")
    endif()
    add_compile_definitions(REX_PLATFORM_WINDOWS=1)
elseif(UNIX AND NOT APPLE)
    # detected Linux/Unix
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|ARM64)$")
        set(REX_PLATFORM "linux-arm64")
        add_compile_definitions(REX_PLATFORM_LINUX=1 REX_ARCH_ARM64=1)
    else()
        set(REX_PLATFORM "linux-amd64")
        add_compile_definitions(REX_PLATFORM_LINUX=1)
    endif()
    

    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    
    # mcmodel=large only for x86_64 CPUs
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64)$")
        add_compile_options(-mcmodel=large)
    endif()
    
    # ARM64 CPUs support
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|ARM64)$")
        add_compile_options(-march=armv8-a)
    endif()
else()
    message(FATAL_ERROR "Unsupported platform. ReXGlue supports Windows and Linux only.")
endif()

# Output directories (multi-config appends /<Config>/ automatically)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/out/${REX_PLATFORM}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/out/${REX_PLATFORM}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/out/${REX_PLATFORM}")

# Sanitizer support para debugging (UBSan es portable, ASan tiene limitaciones)
if(REXGLUE_ENABLE_SANITIZERS)
    message(STATUS "Enabling UndefinedBehaviorSanitizer (UBSan only - ASan conflicts with custom memory mapping)")
    add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=undefined)
endif()

if(REXGLUE_BUILD_TESTS)
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/thirdparty/catch2/extras")
endif()

# Third-party dependencies (vendored sources only)
add_subdirectory(thirdparty)

# Enable warnings after thirdparty
add_compile_options(
    -Wall 
    -Wextra
)

include_directories(${CMAKE_SOURCE_DIR}/include)

add_subdirectory(src/core)
add_subdirectory(src/filesystem)
add_subdirectory(src/ui)
add_subdirectory(src/input)
add_subdirectory(src/audio)
add_subdirectory(src/graphics)
add_subdirectory(src/kernel)
add_subdirectory(src/runtime)
add_subdirectory(src/codegen)    
add_subdirectory(src/rexglue)

# Add tests subdirectory if tests are enabled
if(REXGLUE_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

# Print configuration summary
message(STATUS "===========================================")
message(STATUS "ReXGlue Configuration Summary:")
message(STATUS "  Platform: ${REX_PLATFORM}")
message(STATUS "  Architecture: ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "  Compiler: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "  Configurations: ${CMAKE_CONFIGURATION_TYPES}")
message(STATUS "  Output Dir: ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
message(STATUS "  C++ Standard: C++${CMAKE_CXX_STANDARD}")
message(STATUS "  Graphics: D3D12=${REXGLUE_USE_D3D12} Vulkan=${REXGLUE_USE_VULKAN}")
message(STATUS "===========================================")

# SDK install/export rules
include(cmake/rexglue_install.cmake)
