# Copyright (c) Microsoft Corporation
#
# All rights reserved.
#
# MIT License
#
# Permission is hereby granted, free of charge, to any person obtaining a copy of
# this software and associated documentation files (the "Software"), to deal in
# the Software without restriction, including without limitation the rights to
# use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
# of the Software, and to permit persons to whom the Software is furnished to do
# so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED *AS IS*, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.0)

set(LZ4_TOP_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

# Parse version information
file(STRINGS "${LZ4_TOP_SOURCE_DIR}/lib/lz4.h" LZ4_VERSION_MAJOR REGEX "^#define LZ4_VERSION_MAJOR +([0-9]+) +.*$")
string(REGEX REPLACE "^#define LZ4_VERSION_MAJOR +([0-9]+) +.*$" "\\1" LZ4_VERSION_MAJOR "${LZ4_VERSION_MAJOR}")
file(STRINGS "${LZ4_TOP_SOURCE_DIR}/lib/lz4.h" LZ4_VERSION_MINOR REGEX "^#define LZ4_VERSION_MINOR +([0-9]+) +.*$")
string(REGEX REPLACE "^#define LZ4_VERSION_MINOR +([0-9]+) +.*$" "\\1" LZ4_VERSION_MINOR "${LZ4_VERSION_MINOR}")
file(STRINGS "${LZ4_TOP_SOURCE_DIR}/lib/lz4.h" LZ4_VERSION_RELEASE REGEX "^#define LZ4_VERSION_RELEASE +([0-9]+) +.*$")
string(REGEX REPLACE "^#define LZ4_VERSION_RELEASE +([0-9]+) +.*$" "\\1" LZ4_VERSION_RELEASE "${LZ4_VERSION_RELEASE}")
set(LZ4_VERSION_STRING "${LZ4_VERSION_MAJOR}.${LZ4_VERSION_MINOR}.${LZ4_VERSION_RELEASE}")
mark_as_advanced(LZ4_VERSION_STRING LZ4_VERSION_MAJOR LZ4_VERSION_MINOR LZ4_VERSION_RELEASE)

project(lz4 VERSION ${LZ4_VERSION_STRING} LANGUAGES C)

option(LZ4_EXPORT_PACKAGE "Export the cmake package" OFF)
option(LZ4_EXPORT_PKGCONFIG "Generate and export the pkg configuration" OFF)

include(GNUInstallDirs)
set(INSTALL_BIN_DIR      ${CMAKE_INSTALL_BINDIR}             CACHE PATH "Path where exe and dll will be installed")
set(INSTALL_LIB_DIR      ${CMAKE_INSTALL_LIBDIR}             CACHE PATH "Path where lib will be installed")
set(INSTALL_INCLUDE_DIR  ${CMAKE_INSTALL_INCLUDEDIR}         CACHE PATH "Path where headers will be installed")
set(INSTALL_CMAKE_DIR    "${CMAKE_INSTALL_DATAROOTDIR}/lz4"  CACHE PATH "Path where cmake configs will be installed")

set(LZ4_LIB_SOURCE_DIR "${LZ4_TOP_SOURCE_DIR}/lib")

set(LZ4_HEADERS
    "${LZ4_LIB_SOURCE_DIR}/lz4.h"
    "${LZ4_LIB_SOURCE_DIR}/lz4hc.h"
    "${LZ4_LIB_SOURCE_DIR}/lz4frame.h")

set(LZ4_SOURCES
    "${LZ4_LIB_SOURCE_DIR}/lz4.c"
    "${LZ4_LIB_SOURCE_DIR}/lz4hc.c"
    "${LZ4_LIB_SOURCE_DIR}/lz4frame.c"
    "${LZ4_LIB_SOURCE_DIR}/xxhash.c")

add_library(lz4  ${LZ4_HEADERS} ${LZ4_SOURCES})

target_include_directories(lz4 PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/lib> $<INSTALL_INTERFACE:include>)
set_target_properties(lz4 PROPERTIES
    PUBLIC_HEADER ${LZ4_HEADERS}
    OUTPUT_NAME lz4
    SOVERSION "${LZ4_VERSION_MAJOR}"
    VERSION "${LZ4_VERSION_STRING}")

target_compile_definitions(lz4 PUBLIC XXH_NAMESPACE=LZ4_)

if(MSVC AND BUILD_SHARED_LIBS)
    target_compile_definitions(lz4 PRIVATE LZ4_DLL_EXPORT=1)
endif()


include(CMakePackageConfigHelpers)
write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/lz4ConfigVersion.cmake"
        VERSION ${LZ4_VERSION_STRING}
        COMPATIBILITY SameMajorVersion)

install(TARGETS lz4
    EXPORT lz4Config
    RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
    PUBLIC_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}"
    PRIVATE_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}"
    COMPONENT dev
)

file(GLOB lz4h "${CMAKE_CURRENT_LIST_DIR}/lib/*.h")
list(REMOVE_ITEM lz4h "${CMAKE_CURRENT_LIST_DIR}/lib/xxhash.h")
install(FILES ${lz4h} DESTINATION "${INSTALL_INCLUDE_DIR}")

install(EXPORT lz4Config
  FILE lz4-config.cmake
  NAMESPACE lz4::
  DESTINATION "${INSTALL_CMAKE_DIR}"
)

install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/lz4ConfigVersion.cmake
        DESTINATION ${INSTALL_CMAKE_DIR})

if(LZ4_EXPORT_PACKAGE)
    # Export the package for use from the build-tree (this registers the build-tree with a global CMake-registry)
    export(PACKAGE lz4)
endif()

if(LZ4_EXPORT_PKGCONFIG)
    # Create pkgconfig module file
    file(STRINGS "lib/lz4.h" LZ4_H REGEX "^#define LZ4_VERSION_(MAJOR|MINOR|RELEASE) ")
    string(REGEX REPLACE "^.*MAJOR +([0-9]+).*MINOR +([0-9]+).*RELEASE +([0-9]+).*$" "\\1.\\2.\\3" VERSION "${LZ4_H}")
    set(INCLUDEDIR [[${prefix}/include]])
    set(LIBDIR [[${prefix}/lib]])
    configure_file(lib/liblz4.pc.in lib/liblz4.pc)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/lib/liblz4.pc" DESTINATION "lib/pkgconfig")
endif()
