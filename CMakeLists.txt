##
##  Copyright 2016-2017 libfpta authors: please see AUTHORS file.
##
##  This file is part of libfpta, aka "Fast Positive Tables".
##
##  libfpta is free software: you can redistribute it and/or modify
##  it under the terms of the GNU General Public License as published by
##  the Free Software Foundation, either version 3 of the License, or
##  (at your option) any later version.
##
##  libfpta is distributed in the hope that it will be useful,
##  but WITHOUT ANY WARRANTY; without even the implied warranty of
##  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
##  GNU General Public License for more details.
##
##  You should have received a copy of the GNU General Public License
##  along with libfpta.  If not, see <http://www.gnu.org/licenses/>.
##

##
##  libfpta = { Fast Positive Tables, aka Позитивные Таблицы }
##
##  Ultra fast, compact, embeddable storage engine for (semi)structured data:
##  multiprocessing with zero-overhead, full ACID semantics with MVCC,
##  variety of indexes, saturation, sequences and much more.
##  Please see README.md at https://github.com/leo-yuriev/libfpta
##
##  The Future will Positive. Всё будет хорошо.
##
##  "Позитивные таблицы" предназначены для построения высокоскоростных
##  локальных хранилищ структурированных данных, с целевой производительностью
##  до 1.000.000 запросов в секунду на каждое ядро процессора.
##

cmake_minimum_required(VERSION 2.8.2)
if(NOT CMAKE_VERSION VERSION_LESS 3.1.0)
  cmake_policy(SET CMP0054 NEW)
endif()

project(libfpta C CXX)

if(NOT "$ENV{TEAMCITY_PROCESS_FLOW_ID}" STREQUAL "")
  set(CI TEAMCITY)
  message(STATUS "TeamCity CI")
elseif(NOT "$ENV{TRAVIS}" STREQUAL "")
  set(CI TRAVIS)
  message(STATUS "Travis CI")
elseif(NOT "$ENV{CIRCLECI}" STREQUAL "")
  set(CI CIRCLE)
  message(STATUS "Circle CI")
elseif(NOT "$ENV{APPVEYOR}" STREQUAL "")
  set(CI APPVEYOR)
  message(STATUS "AppVeyor CI")
elseif(NOT "$ENV{CI}" STREQUAL "")
  set(CI "$ENV{CI}")
  message(STATUS "Other CI (${CI})")
else()
  message(STATUS "Assume No any CI environment")
  unset(CI)
endif()

include(CheckLibraryExists)
include(CheckIncludeFiles)
include(CheckCCompilerFlag)
include(CheckSymbolExists)
include(CheckCSourceRuns)
include(CheckCXXSourceRuns)
include(CheckCSourceCompiles)
include(CheckCXXSourceCompiles)
include(TestBigEndian)
include(CheckFunctionExists)
include(FindPackageMessage)
include(CheckStructHasMember)
include(CMakeDependentOption)

if(NOT CMAKE_VERSION VERSION_LESS 3.9)
  cmake_policy(SET CMP0069 NEW)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT CMAKE_INTERPROCEDURAL_OPTIMIZATION_AVAILABLE)
else()
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_AVAILABLE FALSE)
endif()

#
# Set default build type to Release. This is to ease a User's life.
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
    "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
    FORCE)
endif()
string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UPPERCASE)

option(BUILD_SHARED_LIBS "Build shared libraries (DLLs)." ON)

# Define GNU standard installation directories
include(GNUInstallDirs)

include(cmake.fpta/utils.cmake)
# the order is significant: we need to know os and compiler to configure libs
include(cmake.fpta/arch.cmake)
include(cmake.fpta/os.cmake)
include(cmake.fpta/profile.cmake)
include(cmake.fpta/compiler.cmake)
# include(cmake.fpta/atomic.cmake)

if(CMAKE_C_COMPILER_ID STREQUAL "MSVC" AND MSVC_VERSION LESS 1900)
  message(SEND_ERROR "MSVC compiler ${MSVC_VERSION} is too old for building FPTA."
    "At least 'Microsoft Visual Studio 2015' is required.")
endif()

if(CMAKE_CONFIGURATION_TYPES OR NOT CMAKE_BUILD_TYPE_UPPERCASE STREQUAL "DEBUG")
  set(INTERPROCEDURAL_OPTIMIZATION_DEFAULT ON)
else()
  set(INTERPROCEDURAL_OPTIMIZATION_DEFAULT OFF)
endif()

if(CMAKE_INTERPROCEDURAL_OPTIMIZATION_AVAILABLE
    OR GCC_LTO_AVAILABLE OR MSVC_LTO_AVAILABLE OR CLANG_LTO_AVAILABLE)
  option(INTERPROCEDURAL_OPTIMIZATION "Enable interprocedural/LTO optimization." ${INTERPROCEDURAL_OPTIMIZATION_DEFAULT})
endif()

option(PROVIDE_VERSIONINFO "Provide library's version information." ON)

if(INTERPROCEDURAL_OPTIMIZATION)
  if(GCC_LTO_AVAILABLE)
    set(LTO_ENABLED TRUE)
    set(CMAKE_AR ${CMAKE_GCC_AR} CACHE PATH "Path to ar program with LTO-plugin" FORCE)
    set(CMAKE_NM ${CMAKE_GCC_NM} CACHE PATH "Path to nm program with LTO-plugin" FORCE)
    set(CMAKE_RANLIB ${CMAKE_GCC_RANLIB} CACHE PATH "Path to ranlib program with LTO-plugin" FORCE)
    message(STATUS "FPTA indulge Link-Time Optimization by GCC")
  elseif(CLANG_LTO_AVAILABLE)
    set(LTO_ENABLED TRUE)
    set(CMAKE_AR ${CMAKE_CLANG_AR} CACHE PATH "Path to ar program with LTO-plugin" FORCE)
    set(CMAKE_NM ${CMAKE_CLANG_NM} CACHE PATH "Path to nm program with LTO-plugin" FORCE)
    set(CMAKE_RANLIB ${CMAKE_CLANG_RANLIB} CACHE PATH "Path to ranlib program with LTO-plugin" FORCE)
    message(STATUS "FPTA indulge Link-Time Optimization by CLANG")
  elseif(MSVC_LTO_AVAILABLE)
    set(LTO_ENABLED TRUE)
    message(STATUS "FPTA indulge Link-Time Optimization by MSVC")
  elseif(CMAKE_INTERPROCEDURAL_OPTIMIZATION_AVAILABLE)
    message(STATUS "FPTA indulge Interprocedural Optimization by CMake")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    set(LTO_ENABLED TRUE)
  else()
    message(WARNING "Unable to engage interprocedural/LTO optimization.")
  endif()
else()
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)
  set(LTO_ENABLED FALSE)
endif()

find_program(VALGRIND valgrind)
if(VALGRIND)
  # LY: cmake is ugly and nasty.
  #      - therefore memcheck-options should be defined before including ctest;
  #      - otherwise ctest may ignore it.
  set(MEMORYCHECK_SUPPRESSIONS_FILE
    "${PROJECT_SOURCE_DIR}/test/valgrind_suppress.txt"
    CACHE FILEPATH "Suppressions file for Valgrind" FORCE)
  set(MEMORYCHECK_COMMAND_OPTIONS
    "--trace-children=yes --leak-check=full --track-origins=yes --error-exitcode=42 --error-markers=@ --errors-for-leak-kinds=definite --fair-sched=yes --suppressions=${MEMORYCHECK_SUPPRESSIONS_FILE}"
    CACHE STRING "Valgrind options" FORCE)
  set(VALGRIND_COMMAND_OPTIONS "${MEMORYCHECK_COMMAND_OPTIONS}" CACHE STRING "Valgrind options" FORCE)
endif()
include(cmake.fpta/testing.cmake)

find_program(ECHO echo)
find_program(CAT cat)
find_program(GIT git)
find_program(LD ld)

check_function_exists(memmem HAVE_MEMMEM)
check_function_exists(memrchr HAVE_MEMRCHR)
check_function_exists(uuidgen HAVE_UUIDGEN)
CHECK_INCLUDE_FILES(unistd.h HAVE_UNISTD_H)
CHECK_INCLUDE_FILES(sys/uio.h HAVE_SYS_UIO_H)
CHECK_INCLUDE_FILES(sys/stat.h HAVE_SYS_STAT_H)
CHECK_STRUCT_HAS_MEMBER("struct timeval" tv_usec sys/select.h HAVE_TIMEVAL_TV_USEC)
CHECK_STRUCT_HAS_MEMBER("struct timespec" tv_nsec time.h HAVE_TIMESPEC_TV_NSEC)

CHECK_FUNCTION_EXISTS(pow NOT_NEED_LIBM)
if(NOT_NEED_LIBM)
  set(LIB_MATH "")
else()
  set(CMAKE_REQUIRED_LIBRARIES m)
  CHECK_FUNCTION_EXISTS(pow HAVE_LIBM)
  if(HAVE_LIBM)
    set(LIB_MATH m)
  else()
    message(FATAL_ERROR "No libm found for math support")
  endif()
endif()
CHECK_SYMBOL_EXISTS(nan math.h HAVE_nan)
CHECK_SYMBOL_EXISTS(nanf math.h HAVE_nanf)
CHECK_INCLUDE_FILES(ieee754.h HAVE_IEEE754_H)
CHECK_INCLUDE_FILES(endian.h HAVE_ENDIAN_H)
CHECK_INCLUDE_FILES(byteswap.h HAVE_BYTESWAP_H)
CHECK_INCLUDE_FILES(sys/param.h HAVE_SYS_PARAM_H)
CHECK_INCLUDE_FILES(netinet/in.h HAVE_NETINET_IN_H)
CHECK_INCLUDE_FILES(resolv.h HAVE_RESOLV_H)

#
# Enable 'make tags' target.
find_program(CTAGS ctags)
if(CTAGS)
  add_custom_target(tags COMMAND ${CTAGS} -R -f tags
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  add_custom_target(ctags DEPENDS tags)
endif(CTAGS)

#
# Enable 'make reformat' target.
find_program(CLANG_FORMAT
  NAMES clang-format-6.0 clang-format-5.0 clang-format-4.0
  clang-format-3.9 clang-format-3.8 clang-format-3.7 clang-format)
if(CLANG_FORMAT AND UNIX)
  add_custom_target(reformat
    VERBATIM
    COMMAND
      git ls-files |
      grep -E \\.\(c|cxx|cc|cpp|h|hxx|hpp\)\(\\.in\)?\$ |
      xargs ${CLANG_FORMAT} -i --style=file
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif()

include(cmake.fpta/multilib.cmake)

if(NOT "${PROJECT_BINARY_DIR}" STREQUAL "${PROJECT_SOURCE_DIR}")
  add_custom_target(distclean)
  add_custom_command(TARGET distclean
    COMMAND ${CMAKE_COMMAND} -E remove_directory "${PROJECT_BINARY_DIR}"
    COMMENT "Removing the build directory and its content")
elseif(IS_DIRECTORY .git AND GIT)
  add_custom_target(distclean)
  add_custom_command(TARGET distclean
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMAND ${GIT} submodule foreach --recursive git clean -f -X -d
    COMMAND ${GIT} clean -f -X -d
    COMMENT "Removing all build files from the source directory")
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
setup_compile_flags()

#######################################################################

find_package(Threads REQUIRED)

option(FPTA_ALLOW_DOT4NAMES
  "Allow the dot character for table and column names." OFF)
option(FPTA_PROHIBIT_UPSERT_NAN
  "Disallow storing of NaN values in a floating-point columns" ON)
option(FPTA_PROHIBIT_UPSERT_DENIL
  "Disallow storing of 'Designated NIL' values in a numeric columns." ON)
option(FPTA_CLEAN_DENIL
  "Explicit checking for a values that reserved for 'Designated NIL' in the numeric columns and convert ones to the NIL." OFF)
option(FPTA_PROHIBIT_NEARBY4UNORDERED
  "Disallow non-exact cursors positioning for an unordered indexes." ON)
option(FPTA_PROHIBIT_LOSS_PRECISION
  "Disallow storing numeric values into a 32-bit floating-point columns with losing precision" OFF)
option(FPTA_ENABLE_RETURN_INTO_RANGE
  "Allow returning cursor into a data set, after it was been moved out to an invalid position." ON)
option(FPTA_ENABLE_ABORT_ON_PANIC
  "Force abort of application process in case of unrecoverable error" ON)

CMAKE_DEPENDENT_OPTION(T1HA_USE_FAST_ONESHOT_READ
  "Enable oneshot memory reading for little bit faster hashing" ON
  "NOT ENABLE_VALGRIND;NOT ENABLE_ASAN" OFF)

if(BUILD_SHARED_LIBS)
  set(LIBFPTA_STATIC FALSE)
else()
  set(LIBFPTA_STATIC TRUE)
endif()

if(PROVIDE_VERSIONINFO)
  set(HAVE_FPTA_VERSIONINFO TRUE)
else()
  set(HAVE_FPTA_VERSIONINFO FALSE)
endif()

set(HAVE_FPTU_VERSIONINFO FALSE)
set(LIBFPTU_STATIC TRUE)

set(FAST_POSITIVE_CONFIG_H "${CMAKE_CURRENT_BINARY_DIR}/fast_positive/config.h")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/fast_positive/config.h.in ${FAST_POSITIVE_CONFIG_H})

include_directories("${PROJECT_SOURCE_DIR}" "${CMAKE_CURRENT_BINARY_DIR}" "${PROJECT_SOURCE_DIR}/externals/libfptu" "${PROJECT_SOURCE_DIR}/externals")
add_subdirectory(externals)
add_subdirectory(src)
add_subdirectory(test)

if(HAVE_FPTA_VERSIONINFO)
  set(PACKAGE "libfpta")
  set(CPACK_PACKAGE_VERSION_MAJOR ${FPTA_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${FPTA_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${FPTA_VERSION_RELEASE})
  set(CPACK_PACKAGE_VERSION_COMMIT ${FPTA_VERSION_REVISION})
  set(PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}.${CPACK_PACKAGE_VERSION_COMMIT}")
  message(STATUS "libfpta package version is ${PACKAGE_VERSION}")
endif()
