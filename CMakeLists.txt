cmake_minimum_required(VERSION 3.10)

# Project name
project(gslib LANGUAGES C Fortran VERSION 1.0.9)

# Options similar to Makefile variables
option(GSLIB_USE_MPI "Enable MPI" ON)
option(GSLIB_UNDERSCORE "Add underscore function names" ON)
option(GSLIB_USE_GLOBAL_LONG_LONG "Use long long" ON)
option(GSLIB_USE_USR_EXIT "Use custom error handling" OFF)
option(GSLIB_USE_NBC "Non-blocking communication" OFF)
option(BLAS "Enable BLAS" OFF)
option(MKL "Use MKL (requires BLAS enabled)" OFF)
option(SHARED "Build shared library" ON)

set(GSLIB_PREFIX "gslib_")
set(GSLIB_FPREFIX "fgslib_")
set(GSLIB_USE_NAIVE_BLAS ON)
set(GSLIB_USE_CBLAS OFF)
set(GSLIB_USE_MKL OFF)

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

if(BLAS)
  set(GSLIB_USE_NAIVE_BLAS OFF)
  set(GSLIB_USE_CBLAS ON)
  if(MKL)
    set(GSLIB_USE_MKL ON)
  endif()
endif()

if (GSLIB_USE_NAIVE_BLAS)
  add_definitions(-DGSLIB_USE_NAIVE_BLAS)
endif()
if (GSLIB_USE_CBLAS)
  add_definitions(-DGSLIB_USE_CBLAS)
endif()
if (GSLIB_USE_MKL)
  add_definitions(-DGSLIB_USE_MKL)
endif()

# Source directories
set(SRCROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(BLDROOT ${CMAKE_CURRENT_BINARY_DIR})
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
add_library(gs::gs ALIAS gs)

# Install gslib
install(TARGETS gs
  EXPORT gsTargets
  ARCHIVE DESTINATION lib
)
configure_file(
  "${SRCROOT}/config.h.in"
  "${SRCROOT}/src/config.h"
  @ONLY
)
file(GLOB HEADER_FILES "${SRCDIR}/*.h")
install(FILES ${HEADER_FILES} DESTINATION include)

# Export gs targets.
set(EXPORT_TARGETS_FILE "gsTargets.cmake")
install(EXPORT gsTargets
  FILE gsTargets.cmake
  NAMESPACE gs::
  DESTINATION lib/cmake/gs
)
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${BLDROOT}/gsConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)
configure_file(
  "${SRCROOT}/gsConfig.cmake.in"
  "${BLDROOT}/gsConfig.cmake"
  @ONLY
)
install(FILES
  "${BLDROOT}/gsConfig.cmake"
  "${BLDROOT}/gsConfigVersion.cmake"
  DESTINATION lib/cmake/gs
)
