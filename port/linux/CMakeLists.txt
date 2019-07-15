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

set(MBEDTLS_DIR "${CMAKE_SOURCE_DIR}/../../deps/mbedtls")
set(GTEST_DIR "${CMAKE_SOURCE_DIR}/../../deps/gtest")

# Gather source files
file(GLOB SRC_COMMON
  ${CMAKE_SOURCE_DIR}/../../util/*.c
  ${CMAKE_SOURCE_DIR}/../../api/c-timestamp/timestamp_format.c
  ${CMAKE_SOURCE_DIR}/../../api/c-timestamp/timestamp_valid.c
  ${CMAKE_SOURCE_DIR}/../../api/c-timestamp/timestamp_parse.c
  ${CMAKE_SOURCE_DIR}/../../deps/tinycbor/src/cborencoder.c
  ${CMAKE_SOURCE_DIR}/../../deps/tinycbor/src/cborencoder_close_container_checked.c
  ${CMAKE_SOURCE_DIR}/../../deps/tinycbor/src/cborparser.c
)

file(GLOB SRC_SERVER
  ${CMAKE_SOURCE_DIR}/../../messaging/coap/*.c
  ${CMAKE_SOURCE_DIR}/../../api/*.c
  ${CMAKE_SOURCE_DIR}/../../port/linux/*.c
)
set(SRC_CLIENT ${SRC_SERVER})

if(SECURE)
  file(GLOB SRC_DTLS
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/[a-l]*.c
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/md*.c
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/[n-x]*.c
  )
  list(REMOVE_ITEM SRC_DTLS
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/certs.c
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/x509_crl.c
    ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/x509_csr.c
  )
  list(APPEND SRC_COMMON ${SRC_DTLS})

  file(GLOB SRC_SECURE
    ${CMAKE_SOURCE_DIR}/../../security/oc_[a-n]*.c
    ${CMAKE_SOURCE_DIR}/../../security/oc_[p-z]*.c
  )
  list(APPEND SRC_SERVER ${SRC_SECURE})
  list(APPEND SRC_CLIENT ${SRC_SECURE})
  if(DYNAMIC)
	file(GLOB SRC_OBT
      ${CMAKE_SOURCE_DIR}/../../security/oc_obt*.c
    )
    list(APPEND SRC_CLIENT ${SRC_OBT})
  else()
    list(APPEND SRC_COMMON
      ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/library/memory_buffer_alloc.c
    )
  endif()
endif()

if(CLOUD)
  file(GLOB SRC_CLOUD ${CMAKE_SOURCE_DIR}/../../api/cloud/*.c)
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
  set(CLOUD_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/../../api/cloud)
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

# Libraries
add_library(client-static STATIC $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj>)
set_target_properties(client-static PROPERTIES OUTPUT_NAME iotivity-constrained-client)

add_library(client-shared SHARED $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj>)
target_link_libraries(client-shared PUBLIC Threads::Threads)
set_target_properties(client-shared PROPERTIES OUTPUT_NAME iotivity-constrained-client)
set_property(TARGET client-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})


add_library(server-static STATIC $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:server-obj>)
set_target_properties(server-static PROPERTIES OUTPUT_NAME iotivity-constrained-server)

add_library(server-shared SHARED $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:server-obj>)
target_link_libraries(server-shared PUBLIC Threads::Threads)
set_target_properties(server-shared PROPERTIES OUTPUT_NAME iotivity-constrained-server)
set_property(TARGET server-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})


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

# Sample applications
if(SAMPLES)
if(SECURE AND DYNAMIC)
add_executable(onboarding-tool ${CMAKE_SOURCE_DIR}/../../onboarding_tool/obtmain.c)
target_compile_definitions(onboarding-tool PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(onboarding-tool PUBLIC client-static Threads::Threads m rt)
endif()

add_executable(server ${CMAKE_SOURCE_DIR}/../../apps/server_linux.c)
target_compile_definitions(server PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server PUBLIC server-static Threads::Threads m rt)

add_executable(client ${CMAKE_SOURCE_DIR}/../../apps/client_linux.c)
target_compile_definitions(client PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client PUBLIC client-static Threads::Threads m rt)
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
  add_subdirectory("${CMAKE_SOURCE_DIR}/../../deps/gtest" "gtest")
  macro(package_add_test TESTNAME)
    add_executable(${TESTNAME} ${ARGN})
    target_compile_options(${TESTNAME} PRIVATE ${TEST_COMPILE_OPTIONS})
    target_compile_features(${TESTNAME} PRIVATE cxx_nullptr)
    target_compile_definitions(${TESTNAME} PRIVATE ${PUBLIC_COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")
    target_include_directories(${TESTNAME} SYSTEM PRIVATE "${CMAKE_SOURCE_DIR}/../../deps/gtest/include")
    target_include_directories(${TESTNAME} PRIVATE ../../messaging/coap)
    if(SECURE)
      target_include_directories(${TESTNAME} PRIVATE "${CMAKE_SOURCE_DIR}/../../security")
    endif()
    if(CLOUD)
      target_include_directories(${TESTNAME} PRIVATE "${CMAKE_SOURCE_DIR}/../../api/cloud")
    endif()
    target_link_libraries(${TESTNAME} PRIVATE ${TEST_LINK_LIBS})
    add_test(NAME ${TESTNAME} COMMAND ${TESTNAME})
    set_target_properties(${TESTNAME} PROPERTIES FOLDER unittests)
  endmacro()

  file(GLOB SRC_APITEST ${CMAKE_SOURCE_DIR}/../../api/unittest/*.cpp)
  package_add_test(apitest ${SRC_APITEST})

  file(GLOB SRC_PLATFORMTEST ${CMAKE_SOURCE_DIR}/../../port/unittest/*.cpp)
  package_add_test(platformtest ${SRC_PLATFORMTEST})

  file(GLOB SRC_MESSAGINGTEST ${CMAKE_SOURCE_DIR}/../../messaging/coap/unittest/*.cpp)
  package_add_test(messagingtest ${SRC_MESSAGINGTEST})

  if(SECURE)
    file(GLOB SRC_SECURITYTEST ${CMAKE_SOURCE_DIR}/../../security/unittest/*.cpp)
    package_add_test(securitytest ${SRC_SECURITYTEST})
  endif()

  if(CLOUD)
    file(GLOB SRC_CLOUDTEST ${CMAKE_SOURCE_DIR}/../../api/cloud/unittest/*.cpp)
    package_add_test(cloudtest ${SRC_CLOUDTEST})
  endif()
endif()

# Header files
file(GLOB HEADERS ${CMAKE_SOURCE_DIR}/../../include/*.h)
list(APPEND HEADERS ${CMAKE_SOURCE_DIR}/oc_config.h)

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
  iotivity-constrained-client-server.pc
)
foreach(pkg-config-file IN LISTS PKG_CONFIG_FILES)
configure_file(
  "${pkg-config-file}.in"
  ${pkg-config-file}
  @ONLY
)
endforeach()
install(FILES ${PKG_CONFIG_FILES}
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)
