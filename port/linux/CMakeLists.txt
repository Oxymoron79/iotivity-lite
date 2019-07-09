cmake_minimum_required(VERSION 3.2)

project(iotivity-constrained VERSION 1.0.0 LANGUAGES C)

option(DYNAMIC "Build with dynamic resource creation." ON)
option(SECURE "Build with security layer." ON)
option(PKI "Build with private key infrastructure." ON)
option(CLOUD "Build with cloud support." ON)
option(IPV4 "Build with IPv4 support." ON)
option(TCP "Build with TCP support." ON)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

find_package(Threads REQUIRED)
include(GNUInstallDirs)

set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
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
set(COMPILER_DEFS "")
if(SECURE)
  list(APPEND COMPILER_DEFS PRIVATE "__OC_RANDOM")
  list(APPEND COMPILER_DEFS PRIVATE "OC_SECURITY")
endif()
if(DYNAMIC)
  list(APPEND COMPILER_DEFS PRIVATE "OC_DYNAMIC_ALLOCATION")
endif()
if(PKI)
  list(APPEND COMPILER_DEFS PRIVATE "OC_PKI")
endif()
if(CLOUD)
  list(APPEND COMPILER_DEFS "OC_CLOUD")
  set(IPV4 ON)
  set(TCP ON)
endif()
if(IPV4)
  list(APPEND COMPILER_DEFS "OC_IPV4")
endif()
if(TCP)
  list(APPEND COMPILER_DEFS "OC_TCP")
endif()

# Include directories
set(COMMON_INCLUDE_DIRS
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/..
  ${CMAKE_SOURCE_DIR}/../..
  ${CMAKE_SOURCE_DIR}/../../include
)
if(SECURE)
  list(APPEND COMMON_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/../../deps/mbedtls/include)
endif()
if(CLOUD)
  set(CLOUD_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/../../api/cloud)
endif()

# Object libraries
add_library(common-obj OBJECT ${SRC_COMMON})
target_include_directories(common-obj PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(common-obj PRIVATE ${COMPILER_DEFS})

add_library(client-obj OBJECT ${SRC_CLIENT})
target_include_directories(client-obj PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(client-obj PRIVATE ${COMPILER_DEFS} "OC_CLIENT")

add_library(server-obj OBJECT ${SRC_SERVER})
target_include_directories(server-obj PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(server-obj PRIVATE ${COMPILER_DEFS} "OC_SERVER")

add_library(client-server-obj OBJECT ${SRC_CLIENT})
target_include_directories(client-server-obj PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(client-server-obj PRIVATE ${COMPILER_DEFS} "OC_CLIENT" "OC_SERVER")

if(CLOUD)
  add_library(cloud-obj OBJECT ${SRC_CLOUD})
  target_include_directories(cloud-obj PRIVATE ${COMMON_INCLUDE_DIRS})
  target_compile_definitions(cloud-obj PRIVATE ${COMPILER_DEFS} "OC_CLOUD")
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


add_library(client-server-static STATIC $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj> $<TARGET_OBJECTS:cloud-obj>)
set_target_properties(client-server-static PROPERTIES OUTPUT_NAME iotivity-constrained-client-server)

add_library(client-server-shared SHARED $<TARGET_OBJECTS:common-obj> $<TARGET_OBJECTS:client-obj> $<TARGET_OBJECTS:cloud-obj>)
target_link_libraries(client-server-shared PUBLIC Threads::Threads)
set_target_properties(client-server-shared PROPERTIES OUTPUT_NAME iotivity-constrained-client-server)
set_property(TARGET client-server-shared PROPERTY SOVERSION ${PROJECT_VERSION_MAJOR})

add_executable(onboarding-tool ${CMAKE_SOURCE_DIR}/../../onboarding_tool/obtmain.c)
target_include_directories(onboarding-tool PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(onboarding-tool PRIVATE ${COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(onboarding-tool PUBLIC client-static Threads::Threads m rt)

add_executable(server ${CMAKE_SOURCE_DIR}/../../apps/server_linux.c)
target_include_directories(server PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(server PRIVATE ${COMPILER_DEFS} "OC_SERVER")
target_link_libraries(server PUBLIC server-static Threads::Threads m rt)

add_executable(client ${CMAKE_SOURCE_DIR}/../../apps/client_linux.c)
target_include_directories(client PRIVATE ${COMMON_INCLUDE_DIRS})
target_compile_definitions(client PRIVATE ${COMPILER_DEFS} "OC_CLIENT")
target_link_libraries(client PUBLIC client-static Threads::Threads m rt)

# Header files
file(GLOB HEADERS ${CMAKE_SOURCE_DIR}/../../include/*.h)
list(APPEND HEADERS ${CMAKE_SOURCE_DIR}/oc_config.h)

### Generate pkg-config files
configure_file(
  "iotivity-constrained-client.pc.in"
  "${GENERATED_DIR}/iotivity-constrained-client.pc"
  @ONLY
)
install(FILES ${GENERATED_DIR}/iotivity-constrained-client.pc
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)

configure_file(
  "iotivity-constrained-server.pc.in"
  "${GENERATED_DIR}/iotivity-constrained-server.pc"
  @ONLY
)
install(FILES ${GENERATED_DIR}/iotivity-constrained-server.pc
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)

configure_file(
  "iotivity-constrained-client-server.pc.in"
  "${GENERATED_DIR}/iotivity-constrained-client-server.pc"
  @ONLY
)
install(FILES ${GENERATED_DIR}/iotivity-constrained-client-server.pc
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
)
