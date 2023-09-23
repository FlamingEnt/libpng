# CMakeLists.txt

# Copyright (c) 2018-2023 Cosmin Truta
# Copyright (c) 2007,2009-2018 Glenn Randers-Pehrson
# Written by Christian Ehrlicher, 2007
# Revised by Roger Lowman, 2009-2010
# Revised by Clifford Yapp, 2011-2012,2017
# Revised by Claudio Bley, 2013
# Revised by Roger Leigh, 2016
# Revised by Andreas Franek, 2016
# Revised by Sam Serrels, 2017
# Revised by Vadim Barkov, 2017
# Revised by Vicky Pfau, 2018
# Revised by Cameron Cawley, 2018,2021
# Revised by Kyle Bentley, 2018
# Revised by David Callu, 2020
# Revised by Steve Robinson, 2020
# Revised by Simon Hausmann, 2020
# Revised by Alex Gaynor, 2020
# Revised by Owen Rudge, 2020
# Revised by Gleb Mazovetskiy, 2021
# Revised by Christopher Sean Morrison, 2022
# Revised by B. Scott Michel, 2022
# Revised by Jeremy Maitin-Shepard, 2022
# Revised by Martin Storsjo, 2022
# Revised by Jon Creighton, 2023
# Revised by Gunther Nikl, 2023

# This code is released under the libpng license.
# For conditions of distribution and use, see the disclaimer
# and license in png.h

cmake_minimum_required(VERSION 3.1)
cmake_policy(VERSION 3.1)

project(libpng C ASM)
enable_testing()

include(CMakeParseArguments)
include(CheckCSourceCompiles)
include(GNUInstallDirs)

set(PNGLIB_MAJOR 1)
set(PNGLIB_MINOR 6)
set(PNGLIB_REVISION 41)
#set(PNGLIB_SUBREVISION 0)
set(PNGLIB_SUBREVISION "git")
set(PNGLIB_VERSION ${PNGLIB_MAJOR}.${PNGLIB_MINOR}.${PNGLIB_REVISION})
set(PNGLIB_ABI_VERSION ${PNGLIB_MAJOR}${PNGLIB_MINOR})
set(PNGLIB_SHARED_VERSION ${PNGLIB_ABI_VERSION}.${PNGLIB_REVISION}.${PNGLIB_SUBREVISION})

# Allow the users to specify an application-specific API prefix for libpng
# vendoring purposes. A standard libpng build should have no such prefix.
set(PNG_PREFIX ""
    CACHE STRING "Prefix to prepend to the API function names")

# Allow the users to override the postfix appended to debug library file names.
# Previously, we used to set CMAKE_DEBUG_POSTFIX globally. That variable should
# not be cached, however, because doing so would affect all projects processed
# after libpng, in unexpected and undesirable ways.
set(PNG_DEBUG_POSTFIX "d"
    CACHE STRING "Postfix to append to library file names under the Debug configuration")

# Allow the users to import their own extra configuration settings.
set(DFA_XTRA ""
    CACHE FILEPATH "File containing extra configuration settings")

# Allow the users to configure the following build options.
option(PNG_SHARED "Build libpng as a shared lib" OFF)
option(PNG_STATIC "Build libpng as a static lib" ON)
option(PNG_FRAMEWORK "Build libpng as a Mac OS X framework" OFF)
option(PNG_EXECUTABLES "Build libpng executables" OFF)
option(PNG_TESTS "Build libpng tests" OFF)
option(PNG_DEBUG "Enable debug output" OFF)
option(PNG_HARDWARE_OPTIMIZATIONS "Enable hardware optimizations" ON)

# Location of zlib and library name
set(ZLIB_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/../zlib)
set(ZLIB_LIBRARIES zlibstatic)

if(UNIX AND NOT APPLE AND NOT BEOS AND NOT HAIKU AND NOT EMSCRIPTEN)
  find_library(M_LIBRARY m)
  if(NOT M_LIBRARY)
    set(M_LIBRARY "")
  endif()
else()
  # libm is not needed and/or not available.
  set(M_LIBRARY "")
endif()

# CMake currently sets CMAKE_SYSTEM_PROCESSOR to one of x86_64 or arm64 on macOS,
# based upon the OS architecture, not the target architecture. As such, we need
# to check CMAKE_OSX_ARCHITECTURES to identify which hardware-specific flags to
# enable. Note that this will fail if you attempt to build a universal binary in
# a single CMake invocation.
if (APPLE AND CMAKE_OSX_ARCHITECTURES)
  set(TARGET_ARCH ${CMAKE_OSX_ARCHITECTURES})
else()
  set(TARGET_ARCH ${CMAKE_SYSTEM_PROCESSOR})
endif()

if(PNG_HARDWARE_OPTIMIZATIONS)

# Set definitions and sources for ARM.
if(TARGET_ARCH MATCHES "^arm" OR
   TARGET_ARCH MATCHES "^aarch64")
  if(TARGET_ARCH MATCHES "^arm64" OR
     TARGET_ARCH MATCHES "^aarch64")
    set(PNG_ARM_NEON_POSSIBLE_VALUES on off)
    set(PNG_ARM_NEON "on"
        CACHE STRING "Enable ARM NEON optimizations: on|off; on is default")
  else()
    set(PNG_ARM_NEON_POSSIBLE_VALUES check on off)
    set(PNG_ARM_NEON "off"
        CACHE STRING "Enable ARM NEON optimizations: check|on|off; off is default")
  endif()
  set_property(CACHE PNG_ARM_NEON
               PROPERTY STRINGS ${PNG_ARM_NEON_POSSIBLE_VALUES})
  list(FIND PNG_ARM_NEON_POSSIBLE_VALUES ${PNG_ARM_NEON} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_ARM_NEON must be one of [${PNG_ARM_NEON_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_ARM_NEON} STREQUAL "off")
    set(libpng_arm_sources
        arm/arm_init.c
        arm/filter_neon.S
        arm/filter_neon_intrinsics.c
        arm/palette_neon_intrinsics.c)
    if(${PNG_ARM_NEON} STREQUAL "on")
      add_definitions(-DPNG_ARM_NEON_OPT=2)
    elseif(${PNG_ARM_NEON} STREQUAL "check")
      add_definitions(-DPNG_ARM_NEON_CHECK_SUPPORTED)
    endif()
  else()
    add_definitions(-DPNG_ARM_NEON_OPT=0)
  endif()
endif()

# Set definitions and sources for PowerPC.
if(TARGET_ARCH MATCHES "^powerpc*" OR
   TARGET_ARCH MATCHES "^ppc64*")
  set(PNG_POWERPC_VSX_POSSIBLE_VALUES on off)
  set(PNG_POWERPC_VSX "on"
      CACHE STRING "Enable POWERPC VSX optimizations: on|off; on is default")
  set_property(CACHE PNG_POWERPC_VSX
               PROPERTY STRINGS ${PNG_POWERPC_VSX_POSSIBLE_VALUES})
  list(FIND PNG_POWERPC_VSX_POSSIBLE_VALUES ${PNG_POWERPC_VSX} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_POWERPC_VSX must be one of [${PNG_POWERPC_VSX_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_POWERPC_VSX} STREQUAL "off")
    set(libpng_powerpc_sources
        powerpc/powerpc_init.c
        powerpc/filter_vsx_intrinsics.c)
    if(${PNG_POWERPC_VSX} STREQUAL "on")
      add_definitions(-DPNG_POWERPC_VSX_OPT=2)
    endif()
  else()
    add_definitions(-DPNG_POWERPC_VSX_OPT=0)
  endif()
endif()

# Set definitions and sources for Intel.
if(TARGET_ARCH MATCHES "^i?86" OR
   TARGET_ARCH MATCHES "^x86_64*")
  set(PNG_INTEL_SSE_POSSIBLE_VALUES on off)
  set(PNG_INTEL_SSE "on"
      CACHE STRING "Enable INTEL_SSE optimizations: on|off; on is default")
  set_property(CACHE PNG_INTEL_SSE
               PROPERTY STRINGS ${PNG_INTEL_SSE_POSSIBLE_VALUES})
  list(FIND PNG_INTEL_SSE_POSSIBLE_VALUES ${PNG_INTEL_SSE} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_INTEL_SSE must be one of [${PNG_INTEL_SSE_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_INTEL_SSE} STREQUAL "off")
    set(libpng_intel_sources
        intel/intel_init.c
        intel/filter_sse2_intrinsics.c)
    if(${PNG_INTEL_SSE} STREQUAL "on")
      add_definitions(-DPNG_INTEL_SSE_OPT=1)
    endif()
  else()
    add_definitions(-DPNG_INTEL_SSE_OPT=0)
  endif()
endif()

# Set definitions and sources for MIPS.
if(TARGET_ARCH MATCHES "mipsel*" OR
   TARGET_ARCH MATCHES "mips64el*")
  set(PNG_MIPS_MSA_POSSIBLE_VALUES on off)
  set(PNG_MIPS_MSA "on"
      CACHE STRING "Enable MIPS_MSA optimizations: on|off; on is default")
  set_property(CACHE PNG_MIPS_MSA
               PROPERTY STRINGS ${PNG_MIPS_MSA_POSSIBLE_VALUES})
  list(FIND PNG_MIPS_MSA_POSSIBLE_VALUES ${PNG_MIPS_MSA} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_MIPS_MSA must be one of [${PNG_MIPS_MSA_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_MIPS_MSA} STREQUAL "off")
    set(libpng_mips_sources
        mips/mips_init.c
        mips/filter_msa_intrinsics.c)
    if(${PNG_MIPS_MSA} STREQUAL "on")
      add_definitions(-DPNG_MIPS_MSA_OPT=2)
    endif()
  else()
    add_definitions(-DPNG_MIPS_MSA_OPT=0)
  endif()
endif()

else(PNG_HARDWARE_OPTIMIZATIONS)

# Set definitions and sources for ARM.
if(TARGET_ARCH MATCHES "^arm" OR
   TARGET_ARCH MATCHES "^aarch64")
  add_definitions(-DPNG_ARM_NEON_OPT=0)
endif()

# Set definitions and sources for PowerPC.
if(TARGET_ARCH MATCHES "^powerpc*" OR
   TARGET_ARCH MATCHES "^ppc64*")
  add_definitions(-DPNG_POWERPC_VSX_OPT=0)
endif()

# Set definitions and sources for Intel.
if(TARGET_ARCH MATCHES "^i?86" OR
   TARGET_ARCH MATCHES "^x86_64*")
  add_definitions(-DPNG_INTEL_SSE_OPT=0)
endif()

# Set definitions and sources for MIPS.
if(TARGET_ARCH MATCHES "mipsel*" OR
   TARGET_ARCH MATCHES "mips64el*")
  add_definitions(-DPNG_MIPS_MSA_OPT=0)
endif()

endif(PNG_HARDWARE_OPTIMIZATIONS)

option(ld-version-script "Enable linker version script" ON)
if(ld-version-script AND NOT ANDROID AND NOT APPLE)
  # Check if LD supports linker scripts.
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map" "
VERS_1 { global: sym1; local: *; };
VERS_2 { global: sym2; main; } VERS_1;
")
  set(_SAVED_CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})
  if(NOT CMAKE_HOST_SYSTEM_NAME MATCHES "^SunOS")
    # Avoid using CMAKE_SHARED_LIBRARY_C_FLAGS in version script checks on
    # Solaris, because of an incompatibility with the Solaris link editor.
    list(APPEND CMAKE_REQUIRED_FLAGS ${CMAKE_SHARED_LIBRARY_C_FLAGS})
  endif()
  list(APPEND CMAKE_REQUIRED_FLAGS "-Wl,--version-script='${CMAKE_CURRENT_BINARY_DIR}/conftest.map'")
  check_c_source_compiles("
void sym1(void) {}
void sym2(void) {}
int main(void) { return 0; }
" HAVE_LD_VERSION_SCRIPT)
  if(NOT HAVE_LD_VERSION_SCRIPT)
    set(CMAKE_REQUIRED_FLAGS ${_SAVED_CMAKE_REQUIRED_FLAGS})
    if(NOT CMAKE_HOST_SYSTEM_NAME MATCHES "^SunOS")
      # Again, avoid using CMAKE_SHARED_LIBRARY_C_FLAGS in version script
      # checks on Solaris.
      list(APPEND CMAKE_REQUIRED_FLAGS ${CMAKE_SHARED_LIBRARY_C_FLAGS})
    endif()
    list(APPEND CMAKE_REQUIRED_FLAGS "-Wl,-M -Wl,${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
    check_c_source_compiles("
void sym1(void) {}
void sym2(void) {}
int main(void) { return 0; }
" HAVE_SOLARIS_LD_VERSION_SCRIPT)
  endif()
  set(CMAKE_REQUIRED_FLAGS ${_SAVED_CMAKE_REQUIRED_FLAGS})
  file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
endif()

include_directories(${CMAKE_CURRENT_BINARY_DIR})

# List the source code files.
set(libpng_public_hdrs
    png.h
    pngconf.h
    pnglibconf.h
)
set(libpng_private_hdrs
    pngpriv.h
    pngdebug.h
    pnginfo.h
    pngstruct.h
)
set(libpng_sources
    ${libpng_public_hdrs}
    ${libpng_private_hdrs}
    png.c
    pngerror.c
    pngget.c
    pngmem.c
    pngpread.c
    pngread.c
    pngrio.c
    pngrtran.c
    pngrutil.c
    pngset.c
    pngtrans.c
    pngwio.c
    pngwrite.c
    pngwtran.c
    pngwutil.c
    ${libpng_arm_sources}
    ${libpng_intel_sources}
    ${libpng_mips_sources}
    ${libpng_powerpc_sources}
)
set(pngtest_sources
    pngtest.c
)
set(pngvalid_sources
    contrib/libtests/pngvalid.c
)
set(pngstest_sources
    contrib/libtests/pngstest.c
)
set(pngunknown_sources
    contrib/libtests/pngunknown.c
)
set(pngimage_sources
    contrib/libtests/pngimage.c
)
set(pngfix_sources
    contrib/tools/pngfix.c
)
set(png_fix_itxt_sources
    contrib/tools/png-fix-itxt.c
)

if(MSVC OR (WIN32 AND (CMAKE_C_COMPILER_ID MATCHES ".*Clang")))
  add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif()

if(PNG_DEBUG)
  add_definitions(-DPNG_DEBUG)
endif()

# Now build our targets.
include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${ZLIB_INCLUDE_DIRS})

# Initialize the list of libpng library targets.
set(PNG_LIBRARY_TARGETS "")

# Initialize the libpng library file names.
if(UNIX
   OR (WIN32 AND NOT "${CMAKE_SHARED_LIBRARY_PREFIX}" STREQUAL "")
   OR (WIN32 AND NOT "${CMAKE_STATIC_LIBRARY_PREFIX}" STREQUAL ""))
  # We are on a Unix or Unix-like toolchain like the GNU toolchain on Windows.
  # Library file names are expected to have an implicit prefix such as "lib".
  # Let CMake prepend and append its usual prefixes and suffixes by default.
  set(PNG_SHARED_OUTPUT_NAME "png${PNGLIB_ABI_VERSION}")
  set(PNG_STATIC_OUTPUT_NAME "png${PNGLIB_ABI_VERSION}")
else()
  # We are, most likely, on a Windows toolchain like MSVC, Clang on Windows,
  # Borland/Embarcadero, etc. We need to specify the "libpng" name explicitly.
  # We also need to use a custom suffix, in order to distinguish between the
  # shared import library name and the static library name.
  set(PNG_SHARED_OUTPUT_NAME "libpng${PNGLIB_ABI_VERSION}")
  set(PNG_STATIC_OUTPUT_NAME "libpng${PNGLIB_ABI_VERSION}_static")
endif()

if(PNG_SHARED)
  add_library(png_shared SHARED ${libpng_sources})
  add_dependencies(png_shared png_genfiles)
  list(APPEND PNG_LIBRARY_TARGETS png_shared)
  set_target_properties(png_shared PROPERTIES
                        OUTPUT_NAME "${PNG_SHARED_OUTPUT_NAME}"
                        DEBUG_POSTFIX "${PNG_DEBUG_POSTFIX}"
                        VERSION "${PNGLIB_SHARED_VERSION}"
                        SOVERSION "${PNGLIB_ABI_VERSION}")
  if(UNIX AND AWK)
    if(HAVE_LD_VERSION_SCRIPT)
      set_target_properties(png_shared PROPERTIES
                            LINK_FLAGS "-Wl,--version-script='${CMAKE_CURRENT_BINARY_DIR}/libpng.vers'")
    elseif(HAVE_SOLARIS_LD_VERSION_SCRIPT)
      set_target_properties(png_shared PROPERTIES
                            LINK_FLAGS "-Wl,-M -Wl,'${CMAKE_CURRENT_BINARY_DIR}/libpng.vers'")
    endif()
  endif()
  if(WIN32)
    set_target_properties(png_shared PROPERTIES DEFINE_SYMBOL PNG_BUILD_DLL)
  endif()
  target_link_libraries(png_shared ${ZLIB_LIBRARIES} ${M_LIBRARY})
endif()

if(PNG_STATIC)
  add_library(png_static STATIC ${libpng_sources})
  add_dependencies(png_static png_genfiles)
  list(APPEND PNG_LIBRARY_TARGETS png_static)
  set_target_properties(png_static PROPERTIES
                        OUTPUT_NAME "${PNG_STATIC_OUTPUT_NAME}"
                        DEBUG_POSTFIX "${PNG_DEBUG_POSTFIX}")
  target_link_libraries(png_static ${ZLIB_LIBRARIES} ${M_LIBRARY})
endif()

if(PNG_FRAMEWORK)
  add_library(png_framework SHARED ${libpng_sources})
  add_dependencies(png_framework png_genfiles)
  list(APPEND PNG_LIBRARY_TARGETS png_framework)
  set_target_properties(png_framework PROPERTIES
                        FRAMEWORK TRUE
                        FRAMEWORK_VERSION "${PNGLIB_VERSION}"
                        MACOSX_FRAMEWORK_SHORT_VERSION_STRING "${PNGLIB_MAJOR}.${PNGLIB_MINOR}"
                        MACOSX_FRAMEWORK_BUNDLE_VERSION "${PNGLIB_VERSION}"
                        MACOSX_FRAMEWORK_IDENTIFIER "org.libpng.libpng"
                        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
                        PUBLIC_HEADER "${libpng_public_hdrs}"
                        OUTPUT_NAME "png")
  target_link_libraries(png_framework ${ZLIB_LIBRARIES} ${M_LIBRARY})
endif()

if(NOT PNG_LIBRARY_TARGETS)
  message(SEND_ERROR "No library variant selected to build. "
                     "Please enable at least one of the following options: "
                     "PNG_SHARED, PNG_STATIC, PNG_FRAMEWORK")
endif()

# Create source generation scripts.
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/genchk.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/genchk.cmake
               @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/genout.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/genout.cmake
               @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/gensrc.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/gensrc.cmake
               @ONLY)

# TODO: Create MSVC import lib for MinGW-compiled shared lib.
# pexports libpng.dll > libpng.def
# lib /def:libpng.def /machine:x86
