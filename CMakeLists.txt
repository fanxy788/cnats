cmake_minimum_required(VERSION 2.8)

project(cnats)
include(CTest)

# Uncomment to have the build process verbose
#set(CMAKE_VERBOSE_MAKEFILE TRUE)

# Uncomment to have the executable moved to 'build' instead of their respective 'build/xxx' directories
#set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})

option(NATS_UPDATE_VERSION "Update the version file" OFF)
option(NATS_UPDATE_DOC "Update the doc template file" OFF)
option(NATS_COVERAGE "Code coverage" OFF)
option(NATS_BUILD_WITH_TLS "Build with TLS support" ON)
option(NATS_BUILD_WITH_TLS_CLIENT_METHOD "Use TLS_client_method()" OFF)
option(NATS_BUILD_EXAMPLES "Build examples" ON)
option(NATS_BUILD_LIBUV_EXAMPLE "Build libuv examples" OFF)
option(NATS_BUILD_LIBEVENT_EXAMPLE "Build libevent examples" OFF)
option(NATS_BUILD_STATIC_EXAMPLES "Statically link examples" OFF)
option(NATS_BUILD_STREAMING "Build NATS Streaming" ON)
option(NATS_BUILD_NO_PREFIX_CONNSTS "No prefix for connection status enum" OFF)
option(NATS_COMPILER_HARDENING "Compiler hardening flags" OFF)

if(NATS_BUILD_WITH_TLS)
  find_package(OpenSSL REQUIRED)
endif(NATS_BUILD_WITH_TLS)

set(LIBUV_DIR "/usr/local/" CACHE PATH "Libuv install directory")
set(LIBEVENT_DIR "/usr/local/" CACHE PATH "Libevent install directory")

set(NATS_DOC_PROJECT_NAME "NATS C Client")
if(NATS_BUILD_STREAMING)
  set(PROTOBUF_DIR "${PROJECT_SOURCE_DIR}/pbuf")
  add_definitions(-DNATS_HAS_STREAMING)
  set(NATS_DOC_INCLUDE_STREAMING "NATS_HAS_STREAMING")
  set(NATS_DOC_PROJECT_NAME "NATS C Client with Streaming support")
endif(NATS_BUILD_STREAMING)

if(NATS_BUILD_NO_PREFIX_CONNSTS)
  add_definitions(-DNATS_CONN_STATUS_NO_PREFIX)
endif(NATS_BUILD_NO_PREFIX_CONNSTS)

# Platform specific settings
if(UNIX)
  #---------------------------------------------------------------------------
  # Define NATS cache variables that override the CMAKE and MEMCHECK variables
  #---------------------------------------------------------------------------
  include(GNUInstallDirs)

  set(NATS_LIBDIR ${CMAKE_INSTALL_LIBDIR})
  set(NATS_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR})

  set(NATS_BUILD_TYPE Release CACHE STRING "Build type: Release, Debug, RelWithDebInfo, MinRelSize")
  set(CMAKE_BUILD_TYPE ${NATS_BUILD_TYPE} CACHE INTERNAL "")

  set(NATS_BUILD_ARCH "64" CACHE STRING "32 for 32bits builds")

  set(NATS_MEMCHECK_CMD "/usr/bin/valgrind" CACHE FILE "Memcheck tool")
  set(MEMORYCHECK_COMMAND ${NATS_MEMCHECK_CMD} CACHE INTERNAL "")

  set(NATS_MEMCHECK_CMD_OPTS "--leak-check=full --track-fds=yes --show-reachable=yes --num-callers=50" CACHE STRING "Memcheck options")
  set(MEMORYCHECK_COMMAND_OPTIONS ${NATS_MEMCHECK_CMD_OPTS} CACHE INTERNAL "")

  set(NATS_COMMON_C_FLAGS "-std=c99 -pedantic")
  
  # Some compiler hardening flags.
  if(NATS_COMPILER_HARDENING)
     set(NATS_COMMON_C_FLAGS "${NATS_COMMON_C_FLAGS} -fPIE -fstack-protector-all -D_FORTIFY_SOURCE=2 -O1")
     # Works only with GCC.
     if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
       set(NATS_COMMON_C_FLAGS "${NATS_COMMON_C_FLAGS} -Wl,-z,relro,-z,now -pie")
     endif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  endif(NATS_COMPILER_HARDENING)

  if(NATS_BUILD_TYPE MATCHES "Debug")
    set(NATS_COMMON_C_FLAGS "${NATS_COMMON_C_FLAGS} -ggdb")    
  endif(NATS_BUILD_TYPE MATCHES "Debug")
  
  set(NATS_WARNINGS "-Wall -W -Wno-unused-variable -Wno-unused-parameter -Wno-unused-function -Wstrict-prototypes -Wwrite-strings")
  set(NATS_PLATFORM_INCLUDE "unix")
  
  if(APPLE)
    set(CMAKE_MACOSX_RPATH OFF)
    set(NATS_OS "DARWIN")
    if(NATS_BUILD_STREAMING)
      set(NATS_PROTOBUF_LIB "${PROTOBUF_DIR}/lib/darwin/libprotobuf-c.a")
    endif(NATS_BUILD_STREAMING)
  else(APPLE)
    set(NATS_OS "LINUX")
    set(NATS_USE_PTHREAD "-pthread")
    set(NATS_EXTRA_LIB "rt")
    if(NATS_COVERAGE)
      set(NATS_CODE_COVERAGE "--coverage")
    endif()
    if(NATS_BUILD_STREAMING)
      set(NATS_PROTOBUF_LIB "${PROTOBUF_DIR}/lib/linux/libprotobuf-c.so")
    endif(NATS_BUILD_STREAMING)
  endif(APPLE)
  if (${NATS_BUILD_ARCH} MATCHES "32")
    if(NOT APPLE)
      message("-----------------------------------------------------------")
      message("If build fails, you probably need to install libc6-dev-i386")
      message("apt-get install libc6-dev-i386")
      message("-----------------------------------------------------------")
    endif(NOT APPLE)
      
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
    set(CMAKE_C_LINKER_FLAGS "${CMAKE_C_LINKER_FLAGS} -m32")
  endif(${NATS_BUILD_ARCH} MATCHES "32")
  
elseif(WIN32)
  set(NATS_LIBDIR "lib")
  set(NATS_INCLUDE_DIR "include")
  set(NATS_EXTRA_LIB "Ws2_32")
  set(NATS_OS "_WIN32")
  set(NATS_PLATFORM_INCLUDE "win")
  if(NATS_BUILD_STREAMING)
    set(NATS_PROTOBUF_LIB "${PROTOBUF_DIR}/lib/win/protobuf-c.lib")
  endif(NATS_BUILD_STREAMING)
endif(UNIX)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${NATS_CODE_COVERAGE} ${NATS_COMMON_C_FLAGS} ${NATS_USE_PTHREAD} ${NATS_WARNINGS}")

if(UNIX)
  if(APPLE)
  else(APPLE)
    add_definitions(-D_GNU_SOURCE)
  endif(APPLE)
endif(UNIX)
add_definitions(-D${NATS_OS})
add_definitions(-D_REENTRANT)
if(NATS_BUILD_WITH_TLS)
  add_definitions(-DNATS_HAS_TLS)
  if(NATS_BUILD_WITH_TLS_CLIENT_METHOD)
    add_definitions(-DNATS_USE_TLS_CLIENT_METHOD)
  endif(NATS_BUILD_WITH_TLS_CLIENT_METHOD)
endif(NATS_BUILD_WITH_TLS)


#------------
# Versionning and Doc

set(NATS_VERSION_MAJOR  1)
set(NATS_VERSION_MINOR  8)
set(NATS_VERSION_PATCH  0)
set(NATS_VERSION_SUFFIX "")

set(NATS_VERSION_REQUIRED_NUMBER 0x010100)

if(NATS_UPDATE_VERSION OR NATS_UPDATE_DOC)
configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
	${CMAKE_CURRENT_SOURCE_DIR}/src/version.h
	@ONLY)

configure_file(
	${CMAKE_SOURCE_DIR}/doc/DoxyFile.NATS.Client.in
	${CMAKE_SOURCE_DIR}/doc/DoxyFile.NATS.Client
	@ONLY)
endif(NATS_UPDATE_VERSION OR NATS_UPDATE_DOC)
#------------

#----------------------------
# Add the project directories

add_subdirectory(src)
add_subdirectory(examples)
add_subdirectory(examples/getstarted)
if(NATS_BUILD_STREAMING)
  add_subdirectory(examples/stan)
endif()
add_subdirectory(test)
add_subdirectory(test/dylib)
#----------------------------

