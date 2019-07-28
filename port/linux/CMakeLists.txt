cmake_minimum_required(VERSION 3.2)

project(iotivity-constrained VERSION 1.0.0 LANGUAGES C)

option(DYNAMIC "Build with dynamic resource creation." ON)
option(SECURE "Build with security layer." ON)
option(PKI "Build with private key infrastructure." ON)
option(CLOUD "Build with cloud support." OFF)
option(IPV4 "Build with IPv4 support." OFF)
option(TCP "Build with TCP support." OFF)
option(SAMPLES "Build sample applications." ON)
option(UNIT_TESTS "Build the unittests" ON)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)
include(GNUInstallDirs)

if(SECURE)
  find_package(Git REQUIRED)
  find_package(Patch REQUIRED)
  set(MBEDTLS_DIR ${PROJECT_SOURCE_DIR}/../../deps/mbedtls)
  set(MBEDTLS_PATCH_FILE ${MBEDTLS_DIR}/patched.txt)
  if(EXISTS ${PROJECT_SOURCE_DIR}/../../.git)
    if(NOT EXISTS ${MBEDTLS_DIR}/.git)
      message(STATUS "Initialize deps/mbedtls submodule")
      execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init deps/mbedtls
                      WORKING_DIRECTORY  ${PROJECT_SOURCE_DIR}/../..
                      RESULT_VARIABLE GIT_SUBMOD_INIT)
      if(NOT GIT_SUBMOD_INIT EQUAL "0")
        message(FATAL_ERROR "git submodule update --init deps/mbedtls failed with: ${GIT_SUBMOD_INIT}")
      endif()
    endif()
    if(NOT EXISTS ${MBEDTLS_PATCH_FILE})
      message(STATUS "Clean deps/mbedtls submodule")
      execute_process(COMMAND ${GIT_EXECUTABLE} clean -fdx .
                      WORKING_DIRECTORY  ${MBEDTLS_DIR}
                      RESULT_VARIABLE GIT_SUBMOD_CLEAN)
      if(NOT GIT_SUBMOD_CLEAN EQUAL "0")
        message(FATAL_ERROR "git clean -fdx failed with: ${GIT_SUBMOD_CLEAN}")
      endif()
      message(STATUS "Hard reset deps/mbedtls submodule")
      execute_process(COMMAND ${GIT_EXECUTABLE} reset --hard
                      WORKING_DIRECTORY  ${MBEDTLS_DIR}
                      RESULT_VARIABLE GIT_SUBMOD_RESET)
      if(NOT GIT_SUBMOD_RESET EQUAL "0")
        message(FATAL_ERROR "git reset --hard failed with: ${GIT_SUBMOD_RESET}")
       endif()
      message(STATUS "Initialize deps/mbedtls submodule")
      execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init deps/mbedtls
                      WORKING_DIRECTORY  ${PROJECT_SOURCE_DIR}/../..
                      RESULT_VARIABLE GIT_SUBMOD_INIT)
      if(NOT GIT_SUBMOD_INIT EQUAL "0")
        message(FATAL_ERROR "git submodule update --init deps/mbedtls failed with: ${GIT_SUBMOD_INIT}")
      endif()
      message(STATUS "Patch deps/mbedtls submodule")
      file(GLOB MBEDTLS_PATCHES ${PROJECT_SOURCE_DIR}/../../patches/*.patch)
      file(APPEND ${MBEDTLS_PATCH_FILE} "Applied patches:\n")
      foreach(patch IN LISTS MBEDTLS_PATCHES)
        message(STATUS "Apply ${Patch_EXECUTABLE} ${patch}")
        execute_process(COMMAND ${Patch_EXECUTABLE} -r - -s -N -p1
                        INPUT_FILE ${patch}
                        WORKING_DIRECTORY  ${MBEDTLS_DIR}
                        RESULT_VARIABLE GIT_SUBMOD_PATCH)
        if(NOT GIT_SUBMOD_PATCH EQUAL "0")
          message(FATAL_ERROR "patch -r - -s -N -p1 < ${patch} failed with: ${GIT_SUBMOD_PATCH}")
        endif()
        file(APPEND ${MBEDTLS_PATCH_FILE} "${patch}\n")
      endforeach()
      set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES ${MBEDTLS_PATCH_FILE})
    endif()
  endif()
endif()

# Gather source files
file(GLOB SRC_COMMON
  ../../util/*.c
  ../../api/c-timestamp/timestamp_format.c
  ../../api/c-timestamp/timestamp_valid.c
  ../../api/c-timestamp/timestamp_parse.c
  ../../deps/tinycbor/src/cborencoder.c
  ../../deps/tinycbor/src/cborencoder_close_container_checked.c
  ../../deps/tinycbor/src/cborparser.c
)

file(GLOB SRC_SERVER
  ../../messaging/coap/*.c
  ../../api/*.c
  ../../port/linux/*.c
)
set(SRC_CLIENT ${SRC_SERVER})

if(SECURE)
  file(GLOB SRC_DTLS
    ../../deps/mbedtls/library/[a-l]*.c
    ../../deps/mbedtls/library/md*.c
    ../../deps/mbedtls/library/[n-x]*.c
  )
  list(REMOVE_ITEM SRC_DTLS
    ../../deps/mbedtls/library/certs.c
    ../../deps/mbedtls/library/x509_crl.c
    ../../deps/mbedtls/library/x509_csr.c
  )
  list(APPEND SRC_COMMON ${SRC_DTLS})

  file(GLOB SRC_SECURE
    ../../security/oc_[a-n]*.c
    ../../security/oc_[p-z]*.c
  )
  list(APPEND SRC_SERVER ${SRC_SECURE})
  list(APPEND SRC_CLIENT ${SRC_SECURE})
  if(DYNAMIC)
	file(GLOB SRC_OBT
      ../../security/oc_obt*.c
    )
    list(APPEND SRC_CLIENT ${SRC_OBT})
  else()
    list(APPEND SRC_COMMON
      ../../deps/mbedtls/library/memory_buffer_alloc.c
    )
  endif()
endif()

if(CLOUD)
  file(GLOB SRC_CLOUD ../../api/cloud/*.c)
endif()

# Compiler definitions
set(PRIVATE_COMPILER_DEFS "")
set(PUBLIC_COMPILER_DEFS "")

string(APPEND CMAKE_C_FLAGS_DEBUG " -O0 -DOC_DEBUG")
if(SECURE)
  list(APPEND PRIVATE_COMPILER_DEFS "__OC_RANDOM")
  list(APPEND PUBLIC_COMPILER_DEFS "OC_SECURITY")
endif()
if(DYNAMIC)
  list(APPEND PUBLIC_COMPILER_DEFS "OC_DYNAMIC_ALLOCATION")
endif()
if(PKI)
  list(APPEND PUBLIC_COMPILER_DEFS "OC_PKI")
endif()
if(CLOUD)
  list(APPEND PUBLIC_COMPILER_DEFS "OC_CLOUD")
  set(IPV4 ON)
  set(TCP ON)
endif()
if(IPV4)
  list(APPEND PUBLIC_COMPILER_DEFS "OC_IPV4")
endif()
if(TCP)
  list(APPEND PUBLIC_COMPILER_DEFS "OC_TCP")
endif()

# Include directories
include_directories(. .. ../.. ../../include)

if(SECURE)
  include_directories(../../deps/mbedtls/include)
endif()
if(CLOUD)
  set(CLOUD_INCLUDE_DIRS ../../api/cloud)
endif()

# Object libraries
add_library(common-obj OBJECT ${SRC_COMMON})
target_compile_definitions(common-obj PRIVATE ${PRIVATE_COMPILER_DEFS} ${PUBLIC_COMPILER_DEFS})

add_library(client-obj OBJECT ${SRC_CLIENT})
target_compile_definitions(client-obj PRIVATE ${PRIVATE_COMPILER_DEFS} ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")

add_library(server-obj OBJECT ${SRC_SERVER})
target_compile_definitions(server-obj PRIVATE ${PRIVATE_COMPILER_DEFS} ${PUBLIC_COMPILER_DEFS} "OC_SERVER")

add_library(client-server-obj OBJECT ${SRC_CLIENT})
target_compile_definitions(client-server-obj PRIVATE ${PRIVATE_COMPILER_DEFS} ${PUBLIC_COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")

if(CLOUD)
  add_library(cloud-obj OBJECT ${SRC_CLOUD})
  target_compile_definitions(cloud-obj PRIVATE ${PRIVATE_COMPILER_DEFS} ${PUBLIC_COMPILER_DEFS} "OC_CLOUD")
endif()

# Static and shared libraries
# Client
add_library(client-static STATIC $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj>)
set_target_properties(client-static PROPERTIES OUTPUT_NAME iotivity-constrained-client)

add_library(client-shared SHARED $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj>)
target_link_libraries(client-shared PUBLIC Threads::Threads)
set_target_properties(client-shared PROPERTIES OUTPUT_NAME iotivity-constrained-client)
set_property(TARGET client-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})

# Server
add_library(server-static STATIC $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:server-obj>)
set_target_properties(server-static PROPERTIES OUTPUT_NAME iotivity-constrained-server)

add_library(server-shared SHARED $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:server-obj>)
target_link_libraries(server-shared PUBLIC Threads::Threads)
set_target_properties(server-shared PROPERTIES OUTPUT_NAME iotivity-constrained-server)
set_property(TARGET server-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})

# Client-Server
set(client-server-lib-obj $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-server-obj>)
if(CLOUD)
  list(APPEND client-server-lib-obj $<TARGET_OBJECTS:cloud-obj>)
endif()
add_library(client-server-static STATIC ${client-server-lib-obj})
set_target_properties(client-server-static PROPERTIES OUTPUT_NAME iotivity-constrained-client-server)

add_library(client-server-shared SHARED ${client-server-lib-obj})
target_link_libraries(client-server-shared PUBLIC Threads::Threads)
set_target_properties(client-server-shared PROPERTIES OUTPUT_NAME iotivity-constrained-client-server)
set_property(TARGET client-server-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})

# Install libraries
install(TARGETS
  client-static client-shared
  server-static server-shared
  client-server-static client-server-shared)

# Install header files
install(DIRECTORY ../../include/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/iotivity-constrained
  FILES_MATCHING PATTERN "*.h")
install(FILES oc_config.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/iotivity-constrained/port)
install(DIRECTORY ../../util
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/iotivity-constrained
  FILES_MATCHING PATTERN "*.h")
install(DIRECTORY ../../messaging/coap/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/iotivity-constrained/messaging/coap
  FILES_MATCHING PATTERN "*.h"
  PATTERN "unittest" EXCLUDE)
install(DIRECTORY ../../deps/tinycbor/src/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/iotivity-constrained/deps/tinycbor/src
  FILES_MATCHING PATTERN "*.h")

# Generate pkg-config files
set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${prefix}/${CMAKE_INSTALL_LIBDIR}")
set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
set(version ${PROJECT_VERSION})
foreach(cflag IN LISTS PUBLIC_COMPILER_DEFS)
  string(APPEND extra_cflags "-D${cflag} ")
endforeach()

set(PKG_CONFIG_FILES
  iotivity-constrained-client.pc
  iotivity-constrained-server.pc
  iotivity-constrained-client-server.pc)
foreach(pkg-config-file IN LISTS PKG_CONFIG_FILES)
configure_file(
  "${pkg-config-file}.in"
  ${pkg-config-file}
  @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${pkg-config-file}
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endforeach()

# Sample applications
if(SAMPLES)
if(SECURE AND DYNAMIC)
add_executable(onboarding_tool ../../onboarding_tool/obtmain.c)
target_compile_definitions(onboarding_tool PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(onboarding_tool PUBLIC client-static Threads::Threads m rt)
install(TARGETS onboarding_tool DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)
endif()

add_executable(server ../../apps/server_linux.c)
target_compile_definitions(server PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server PUBLIC server-static Threads::Threads m rt)
install(TARGETS server DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(client ../../apps/client_linux.c)
target_compile_definitions(client PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client PUBLIC client-static Threads::Threads m rt)
install(TARGETS client DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(smart_lock ../../apps/smart_lock_linux.c)
target_compile_definitions(smart_lock PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(smart_lock PUBLIC client-static Threads::Threads m rt)
install(TARGETS smart_lock DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(temp_sensor ../../apps/temp_sensor_client_linux.c)
target_compile_definitions(temp_sensor PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(temp_sensor PUBLIC client-static Threads::Threads m rt)
install(TARGETS temp_sensor DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(simpleserver ../../apps/simpleserver.c)
target_compile_definitions(simpleserver PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(simpleserver PUBLIC server-static Threads::Threads m rt)
install(TARGETS simpleserver DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(simpleserver_pki ../../apps/simpleserver_pki.c)
target_compile_definitions(simpleserver_pki PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(simpleserver_pki PUBLIC server-static Threads::Threads m rt)
install(TARGETS simpleserver_pki DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(simpleclient ../../apps/simpleclient.c)
target_compile_definitions(simpleclient PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(simpleclient PUBLIC client-static Threads::Threads m rt)
install(TARGETS simpleclient DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(client_collections_linux ../../apps/client_collections_linux.c)
target_compile_definitions(client_collections_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client_collections_linux PUBLIC client-static Threads::Threads m rt)
install(TARGETS client_collections_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(server_collections_linux ../../apps/server_collections_linux.c)
target_compile_definitions(server_collections_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server_collections_linux PUBLIC server-static Threads::Threads m rt)
install(TARGETS server_collections_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(client_block_linux ../../apps/client_block_linux.c)
target_compile_definitions(client_block_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client_block_linux PUBLIC client-static Threads::Threads m rt)
install(TARGETS client_block_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(server_block_linux ../../apps/server_block_linux.c)
target_compile_definitions(server_block_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server_block_linux PUBLIC server-static Threads::Threads m rt)
install(TARGETS server_block_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(smart_home_server_linux ../../apps/smart_home_server_linux.c)
target_compile_definitions(smart_home_server_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(smart_home_server_linux PUBLIC server-static Threads::Threads m rt)
install(TARGETS smart_home_server_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(multi_device_server_linux ../../apps/multi_device_server_linux.c)
target_compile_definitions(multi_device_server_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(multi_device_server_linux PUBLIC server-static Threads::Threads m rt)
install(TARGETS multi_device_server_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(multi_device_client_linux ../../apps/multi_device_client_linux.c)
target_compile_definitions(multi_device_client_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(multi_device_client_linux PUBLIC client-static Threads::Threads m rt)
install(TARGETS multi_device_client_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

if(CLOUD)
add_executable(cloud_linux ../../apps/cloud_linux.c)
target_compile_definitions(cloud_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")
target_include_directories(cloud_linux PRIVATE ${CLOUD_INCLUDE_DIRS})
target_link_libraries(cloud_linux PUBLIC client-server-static Threads::Threads m rt)
install(TARGETS cloud_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(cloud_certification_tests ../../apps/cloud_certification_tests.c)
target_compile_definitions(cloud_certification_tests PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")
target_include_directories(cloud_certification_tests PRIVATE ${CLOUD_INCLUDE_DIRS})
target_link_libraries(cloud_certification_tests PUBLIC client-server-static Threads::Threads m rt)
install(TARGETS cloud_certification_tests DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)
endif()

add_executable(server_multithread_linux ../../apps/server_multithread_linux.c)
target_compile_definitions(server_multithread_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server_multithread_linux PUBLIC server-static Threads::Threads m rt)
install(TARGETS server_multithread_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)

add_executable(client_multithread_linux ../../apps/client_multithread_linux.c)
target_compile_definitions(client_multithread_linux PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client_multithread_linux PUBLIC client-static Threads::Threads m rt)
install(TARGETS client_multithread_linux DESTINATION ${CMAKE_INSTALL_BINDIR}/iotivity-constrained)
endif() # SAMPLES

if(UNIT_TESTS)
  enable_testing()
  enable_language(CXX)
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
  set(CMAKE_CXX_EXTENSIONS OFF)
  list(APPEND TEST_COMPILE_OPTIONS "-Wall" "-Wextra" "-fpermissive")
  list(APPEND TEST_LINK_LIBS client-server-static gtest_main Threads::Threads)
  if(CMAKE_COMPILER_IS_GNUCXX)
    list(APPEND TEST_COMPILE_OPTIONS "-fprofile-arcs" "-ftest-coverage")
    list(APPEND TEST_LINK_LIBS gcov)
  endif()
  add_subdirectory(../../deps/gtest gtest)
  macro(package_add_test TESTNAME)
    add_executable(${TESTNAME} ${ARGN})
    target_compile_options(${TESTNAME} PRIVATE ${TEST_COMPILE_OPTIONS})
    target_compile_features(${TESTNAME} PRIVATE cxx_nullptr)
    target_compile_definitions(${TESTNAME} PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")
    target_include_directories(${TESTNAME} SYSTEM PRIVATE ../../deps/gtest/include)
    target_include_directories(${TESTNAME} PRIVATE ../../messaging/coap)
    if(SECURE)
      target_include_directories(${TESTNAME} PRIVATE ../../security)
    endif()
    if(CLOUD)
      target_include_directories(${TESTNAME} PRIVATE ../../api/cloud)
    endif()
    target_link_libraries(${TESTNAME} PRIVATE ${TEST_LINK_LIBS})
    add_test(NAME ${TESTNAME} COMMAND ${TESTNAME})
    set_target_properties(${TESTNAME} PROPERTIES FOLDER unittests)
  endmacro()

  file(GLOB SRC_APITEST ../../api/unittest/*.cpp)
  package_add_test(apitest ${SRC_APITEST})

  file(GLOB SRC_PLATFORMTEST ../../port/unittest/*.cpp)
  file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/storage_test)
  package_add_test(platformtest ${SRC_PLATFORMTEST})

  file(GLOB SRC_MESSAGINGTEST ../../messaging/coap/unittest/*.cpp)
  package_add_test(messagingtest ${SRC_MESSAGINGTEST})

  if(SECURE)
    file(GLOB SRC_SECURITYTEST ../../security/unittest/*.cpp)
    package_add_test(securitytest ${SRC_SECURITYTEST})
  endif()

  if(CLOUD)
    file(GLOB SRC_CLOUDTEST ../../api/cloud/unittest/*.cpp)
    package_add_test(cloudtest ${SRC_CLOUDTEST})
  endif()
endif()
