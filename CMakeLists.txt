cmake_minimum_required(VERSION 3.16)
project(lib_gateway_tcp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# True when this repository is the top-level project, false when it is
# pulled into another project with add_subdirectory() (a vendored copy).
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  set(TCP_GATEWAY_IS_TOP_LEVEL ON)
else()
  set(TCP_GATEWAY_IS_TOP_LEVEL OFF)
endif()

# Install rules are on for a standalone build and off for a vendored copy,
# so a parent project's "make install" does not install this library.
option(TCP_GATEWAY_INSTALL "Generate install rules" ${TCP_GATEWAY_IS_TOP_LEVEL})

# Shared by default when built on its own; static by default when vendored.
# Set BUILD_SHARED_LIBS before add_subdirectory() to override either default.
if(NOT DEFINED BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS ${TCP_GATEWAY_IS_TOP_LEVEL})
endif()

include(GNUInstallDirs)

# --- Dependencies ---
find_package(Boost REQUIRED COMPONENTS system)
find_package(Protobuf REQUIRED)

# Own variable name, so a parent project's B64_LIBRARIES is never touched.
# (find_library(... REQUIRED) needs CMake 3.18, so check by hand.)
find_library(TCP_GATEWAY_B64_LIBRARY b64)
if(NOT TCP_GATEWAY_B64_LIBRARY)
  message(FATAL_ERROR "libb64 not found. Install it with: sudo apt install libb64-dev")
endif()

# --- Library target ---
# Type (SHARED or STATIC) follows BUILD_SHARED_LIBS, set above.
add_library(tcp_gateway
    src/tcp_client.cpp
    src/tcp_server.cpp
    src/tcp_session.cpp
)
add_library(tcp_gateway::tcp_gateway ALIAS tcp_gateway)

# Headers already live under include/lib_gateway_tcp/, so the build-tree and
# installed include paths are the same form for consumers.
target_include_directories(tcp_gateway
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE
        ${Protobuf_INCLUDE_DIRS}
)

target_link_libraries(tcp_gateway
    PUBLIC
        Boost::system
        ${Protobuf_LIBRARIES}
        ${TCP_GATEWAY_B64_LIBRARY}
)

# PIC so a static build can still be linked into a shared object.
set_target_properties(tcp_gateway PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

# --- Install rules (standalone build only, by default) ---
if(TCP_GATEWAY_INSTALL)
  include(CMakePackageConfigHelpers)

  install(TARGETS tcp_gateway
      EXPORT tcp_gatewayTargets
      LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )

  install(DIRECTORY include/lib_gateway_tcp
      DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )

  # CMake package export so find_package() works
  install(EXPORT tcp_gatewayTargets
      FILE tcp_gatewayTargets.cmake
      NAMESPACE tcp_gateway::
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tcp_gateway
  )

  configure_package_config_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake/tcp_gatewayConfig.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/tcp_gatewayConfig.cmake
      INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tcp_gateway
  )

  write_basic_package_version_file(
      ${CMAKE_CURRENT_BINARY_DIR}/tcp_gatewayConfigVersion.cmake
      VERSION ${PROJECT_VERSION}
      COMPATIBILITY SameMajorVersion
  )

  install(FILES
      ${CMAKE_CURRENT_BINARY_DIR}/tcp_gatewayConfig.cmake
      ${CMAKE_CURRENT_BINARY_DIR}/tcp_gatewayConfigVersion.cmake
      DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tcp_gateway
  )
endif()
