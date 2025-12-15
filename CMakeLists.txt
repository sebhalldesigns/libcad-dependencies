cmake_minimum_required(VERSION 3.16)
project(libcad-dependencies)

# Enforce Release-only builds
if (NOT CMAKE_CONFIGURATION_TYPES) # single-config generators (Ninja, Makefiles)
    if (NOT CMAKE_BUILD_TYPE)
        set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
    elseif (NOT CMAKE_BUILD_TYPE STREQUAL "Release")
        message(FATAL_ERROR
            "This project only supports Release builds. "
            "Reconfigure with -DCMAKE_BUILD_TYPE=Release."
        )
    endif()
endif()

if (CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES Release CACHE STRING "" FORCE)
endif()

# Ensure Python is available and locate the gn python script directory
find_package(Python3 REQUIRED)

set(SKIA_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/skia")
set(SKIA_BIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/skia/bin")
set(SKIA_BUILD_DIR "${CMAKE_BINARY_DIR}/skia")
set(SKIA_DEPS_CHECK_FILE "${CMAKE_BINARY_DIR}/skia_deps_fetched")
set(SKIA_SPIRV_TOOLS_DIR "${SKIA_SRC_DIR}/third_party/externals/spirv-tools")
set(SKIA_NINJA_TARGET "skia")

if(WIN32 AND NOT EMSCRIPTEN)
    set(SKIA_STATIC_LIB "${SKIA_BUILD_DIR}/skia.lib" CACHE FILEPATH "Path to Skia library")
else()
    set(SKIA_STATIC_LIB "${SKIA_BUILD_DIR}/libskia.a" CACHE FILEPATH "Path to Skia library")
endif()

# Detect if Skia dependencies are already present. If not, run git-sync-deps to clone them all.
if (NOT EXISTS "${SKIA_SPIRV_TOOLS_DIR}")
    add_custom_command(
        OUTPUT ${SKIA_DEPS_CHECK_FILE}
        COMMAND ${Python3_EXECUTABLE} tools/git-sync-deps
        COMMAND ${CMAKE_COMMAND} -E touch ${SKIA_DEPS_CHECK_FILE}
        COMMENT "Fetching Skia third-party dependencies"
        WORKING_DIRECTORY ${SKIA_SRC_DIR}
        VERBATIM
    )
else()
    # Create a dummy file so CMake doesn't complain
    file(WRITE ${SKIA_DEPS_CHECK_FILE} "")
endif()

# Fetch GN and Ninja tools
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/gn_fetch
    COMMAND ${Python3_EXECUTABLE} ${SKIA_BIN_DIR}/fetch-gn
    DEPENDS ${SKIA_DEPS_CHECK_FILE} 
    COMMENT "Fetching GN for Skia"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/skia
    VERBATIM
)

# Generate skia code
if (EMSCRIPTEN)

    set(EMSDK_ROOT "${SKIA_SRC_DIR}/third_party/externals/emsdk")
    set(EMSCRIPTEN_ROOT "${EMSDK_ROOT}/upstream/emscripten")

    file(TO_CMAKE_PATH "${EMSCRIPTEN_ROOT}/emcc.py" EMCC_PY)
    file(TO_CMAKE_PATH "${EMSCRIPTEN_ROOT}/em++.py" EMCXX_PY)
    file(TO_CMAKE_PATH "${EMSCRIPTEN_ROOT}/emar.py" EMAR_PY)

    add_custom_command( 
        OUTPUT ${CMAKE_BINARY_DIR}/skia_gen
        COMMAND ${SKIA_BIN_DIR}/gn gen ${SKIA_BUILD_DIR} --args=is_official_build=true\ is_debug=false\ is_component_build=false\ skia_enable_tools=false\ skia_use_freetype=false\ skia_use_system_libpng=false\ skia_use_system_freetype2=false\ skia_use_system_zlib=false\ skia_use_system_libjpeg_turbo=false\ skia_use_system_harfbuzz=false\ skia_use_system_libwebp=false\ skia_use_system_expat=false\ skia_use_system_icu=false\ skia_use_webgl=true\ skia_use_gl=true\ target_os=\"emscripten\"\ target_cpu=\"wasm\"\ cc=\"${EMCC_PY}\"\ cxx=\"${EMCXX_PY}\"\ ar=\"${EMAR_PY}\"
        DEPENDS ${CMAKE_BINARY_DIR}/gn_fetch
        COMMENT "Generating Skia build files"
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/skia
        VERBATIM
    )
elseif (WIN32)
    add_custom_command( 
        OUTPUT ${CMAKE_BINARY_DIR}/skia_gen
        COMMAND ${SKIA_BIN_DIR}/gn gen ${SKIA_BUILD_DIR} --args=is_official_build=true\ is_debug=false\ is_component_build=false\ skia_enable_tools=false\ skia_use_system_libpng=false\ skia_use_system_zlib=false\ skia_use_system_libjpeg_turbo=false\ skia_use_system_harfbuzz=false\ skia_use_system_libwebp=false\ skia_use_system_expat=false\ skia_use_system_icu=false\ skia_use_gl=true\ extra_cflags=[\"/MT\"]\ extra_ldflags=[\"/MT\"]
        DEPENDS ${CMAKE_BINARY_DIR}/gn_fetch
        COMMENT "Generating Skia build files"
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/skia
        VERBATIM
    )
else()
    add_custom_command( 
        OUTPUT ${CMAKE_BINARY_DIR}/skia_gen
        COMMAND ${SKIA_BIN_DIR}/gn gen ${SKIA_BUILD_DIR} --args=is_official_build=true\ is_debug=false\ is_component_build=false\ skia_enable_tools=false\ skia_use_system_libpng=false\ skia_use_system_freetype2=false\ skia_use_system_zlib=false\ skia_use_system_libjpeg_turbo=false\ skia_use_system_harfbuzz=false\ skia_use_system_libwebp=false\ skia_use_system_expat=false\ skia_use_system_icu=false\ skia_use_egl=true\ skia_use_gl=true
        DEPENDS ${CMAKE_BINARY_DIR}/gn_fetch
        COMMENT "Generating Skia build files"
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/skia
        VERBATIM
    )
endif()

add_custom_command(
    OUTPUT ${SKIA_STATIC_LIB}
    COMMAND ninja -C ${SKIA_BUILD_DIR} skia
    DEPENDS ${CMAKE_BINARY_DIR}/skia_gen
    WORKING_DIRECTORY ${SKIA_BUILD_DIR}
    COMMENT "Building Skia"
    VERBATIM
)

add_custom_target(build_skia ALL
    DEPENDS ${SKIA_STATIC_LIB}
)

add_library(skia STATIC IMPORTED GLOBAL)

set_target_properties(skia PROPERTIES
    IMPORTED_LOCATION "${SKIA_STATIC_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES "${SKIA_SRC_DIR}/include"
)

add_dependencies(skia build_skia)

## GET PLATFORM/ARCH/BUILD_TYPE

if (EMSCRIPTEN)
    set(SKIA_PLATFORM "emscripten")
elseif (WIN32)
    set(SKIA_PLATFORM "windows")
elseif (APPLE)
    set(SKIA_PLATFORM "darwin")
elseif (UNIX)
    set(SKIA_PLATFORM "linux")
else()
    set(SKIA_PLATFORM "unknown")
endif()

if (EMSCRIPTEN)
    set(SKIA_ARCH "wasm")
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "AMD64|x86_64")
    set(SKIA_ARCH "amd64")
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "ARM64|aarch64")
    set(SKIA_ARCH "arm64")
else()
    set(SKIA_ARCH "${CMAKE_SYSTEM_PROCESSOR}")
endif()

set(CMAKE_BUILD_TYPE "release")

string(TOLOWER "${CMAKE_BUILD_TYPE}" SKIA_BUILD_TYPE)

## INSTALL & PACK

install(
    FILES ${SKIA_STATIC_LIB}
    DESTINATION lib
)

install(
    FILES ${SKIA_SRC_DIR}/LICENSE
    DESTINATION .
)

install(
    DIRECTORY ${SKIA_SRC_DIR}/include/
    DESTINATION include/skia
)

set(CPACK_GENERATOR "ZIP")

set(CPACK_PACKAGE_NAME "skia")

set(CPACK_PACKAGE_FILE_NAME
    "skia-${SKIA_PLATFORM}-${SKIA_ARCH}-${SKIA_BUILD_TYPE}"
)

# Put all packages in a shared output directory
set(CPACK_OUTPUT_FILE_PREFIX
    "${CMAKE_SOURCE_DIR}/output"
)

include(CPack)
