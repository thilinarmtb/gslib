cmake_minimum_required(VERSION 3.10)

# Project name
project(gslib LANGUAGES C Fortran)

# Options similar to Makefile variables
option(GSLIB_USE_MPI "Enable MPI" ON)
option(GSLIB_UNDERSCORE "Add underscore function names" ON)
option(GSLIB_USE_GLOBAL_LONG_LONG "Use long long" ON)
option(GSLIB_USE_USR_EXIT "Use custom error handling" OFF)
option(GSLIB_USE_NBC "Non-blocking communication" OFF)
set(GSLIB_PREFIX "gslib_")
set(GSLIB_FPREFIX "fgslib_")

option(BLAS "Enable BLAS" OFF)
option(MKL "Use MKL (requires BLAS enabled)" OFF)
option(SHARED "Build shared library" ON)

# Define compilation flags
set(CMAKE_C_FLAGS_RELEASE "-O2")
set(CMAKE_Fortran_FLAGS_RELEASE "-O2")

# Define the compile definitions
if(GSLIB_USE_MPI)
  find_package(MPI REQUIRED)
  add_definitions(-DGSLIB_USE_MPI)
  set(CMAKE_C_COMPILER ${MPI_C_COMPILER})
  set(CMAKE_Fortran_COMPILER ${MPI_Fortran_COMPILER})
endif()

if(GSLIB_UNDERSCORE)
  add_definitions(-DGSLIB_UNDERSCORE)
endif()

add_definitions(-DGSLIB_PREFIX=${GSLIB_PREFIX})
add_definitions(-DGSLIB_FPREFIX=${GSLIB_FPREFIX})
if (GSLIB_USE_GLOBAL_LONG_LONG)
  add_definitions(-DGSLIB_USE_GLOBAL_LONG_LONG)
endif()

if(GSLIB_USE_USR_EXIT)
  add_definitions(-DGSLIB_USE_USR_EXIT)
endif()

if(GSLIB_USE_NBC)
  add_definitions(-DGSLIB_USE_NBC)
endif()

if(NOT BLAS)
  add_definitions(-DGSLIB_USE_NAIVE_BLAS)
elseif(BLAS AND MKL)
  add_definitions(-DGSLIB_USE_CBLAS -DGSLIB_USE_MKL)
endif()

# Source directories
set(SRCROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(TESTDIR ${SRCROOT}/tests)
set(FTESTDIR ${TESTDIR}/fortran)
set(SRCDIR ${SRCROOT}/src)
set(LIBDIR ${SRCROOT}/lib)

# Source files
set(GS
    ${SRCDIR}/gs.c
    ${SRCDIR}/sort.c
    ${SRCDIR}/sarray_transfer.c
    ${SRCDIR}/sarray_sort.c
    ${SRCDIR}/gs_local.c
    ${SRCDIR}/fail.c
    ${SRCDIR}/crystal.c
    ${SRCDIR}/comm.c
    ${SRCDIR}/tensor.c)

set(FWRAPPER
    ${SRCDIR}/fcrystal.c
    ${SRCDIR}/findpts.c)

set(INTP
    ${SRCDIR}/findpts_local.c
    ${SRCDIR}/obbox.c
    ${SRCDIR}/poly.c
    ${SRCDIR}/lob_bnd.c
    ${SRCDIR}/findpts_el_3.c
    ${SRCDIR}/findpts_el_2.c)

if(SHARED)
  add_library(gs SHARED ${GS} ${FWRAPPER} ${INTP})
else()
  add_library(gs STATIC ${GS} ${FWRAPPER} ${INTP})
endif()
target_include_directories(gs
  PUBLIC
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${SRCDIR}>
)

# Configure library export options
set(EXPORT_TARGETS_FILE "gsTargets.cmake") # Exported targets filename

# Make it exportable by creating an export name
install(TARGETS gs
  EXPORT gsTargets
  ARCHIVE DESTINATION lib
)

# Install headers and CMake configuration
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/gsConfigVersion.cmake"
  VERSION 1.0.9
  COMPATIBILITY AnyNewerVersion
)
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/gsConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/gsConfig.cmake"
  @ONLY
)
install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/gsConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/gsConfigVersion.cmake"
  DESTINATION lib/cmake/gs
)

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/config.h.in"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/config.h"
  @ONLY
)

file(GLOB HEADER_FILES "${SRCDIR}/*.h")
install(FILES ${HEADER_FILES} DESTINATION include)
install(EXPORT gsTargets
  FILE gsTargets.cmake
  NAMESPACE gs::
  DESTINATION lib/cmake/gs
)
