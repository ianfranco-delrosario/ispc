## Copyright 2020 Intel Corporation
## SPDX-License-Identifier: BSD-3-Clause

cmake_minimum_required(VERSION 3.1)

## Global setup ##

set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

set(CMAKE_C_STANDARD   11)
set(CMAKE_CXX_STANDARD 11)

set(CMAKE_C_STANDARD_REQUIRED   ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(GNUInstallDirs)

## Establish project ##########################################################

# TODO: use ISPC version here
project(ispcrt VERSION 1.13.0 LANGUAGES C CXX)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

## Add library and executable targets #########################################

option(ISPCRT_BUILD_CPU "Enable CPU support in ispcrt" ON)
option(ISPCRT_BUILD_GPU "Enable Level0 GPU support in ispcrt" ON)
option(ISPCRT_BUILD_TASKING "Enable CPU tasking targets in ispcrt" ON)

if (NOT ISPCRT_BUILD_CPU AND NOT ISPCRT_BUILD_GPU)
  message(FATAL_ERROR "You must enable either CPU or GPU support!")
endif()

if (ISPCRT_BUILD_GPU)
  find_package(level_zero REQUIRED)
endif()

if (ISPCRT_BUILD_TASKING)
  # TODO: support multiple tasking system options
  find_package(OpenMP REQUIRED)
endif()

## Define a macro to build both static and shared versions of ispcrt

macro(build_ispcrt SHARED_OR_STATIC TARGET_NAME)
  ## Build library ##

  add_library(${TARGET_NAME} ${SHARED_OR_STATIC}
    $<$<BOOL:${ISPCRT_BUILD_TASKING}>:ispc_tasking.c>

    ispcrt.cpp
    $<$<BOOL:${ISPCRT_BUILD_CPU}>:detail/cpu/CPUDevice.cpp>
    $<$<BOOL:${ISPCRT_BUILD_GPU}>:detail/gpu/GPUDevice.cpp>
  )

  if (ISPCRT_BUILD_GPU)
    target_compile_definitions(${TARGET_NAME} PRIVATE ISPCRT_BUILD_GPU)
  endif()

  if (ISPCRT_BUILD_CPU)
    target_compile_definitions(${TARGET_NAME} PRIVATE ISPCRT_BUILD_CPU)
  endif()

  target_include_directories(${TARGET_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/ispcrt>
  )

  target_link_libraries(${TARGET_NAME}
  PUBLIC
    $<$<BOOL:${ISPCRT_BUILD_GPU}>:level_zero>
  PRIVATE
    ${CMAKE_DL_LIBS}
    $<$<BOOL:${ISPCRT_BUILD_TASKING}>:OpenMP::OpenMP_CXX>
    $<$<BOOL:${ISPCRT_BUILD_TASKING}>:OpenMP::OpenMP_C>
  )

  ## Install targets + exports ##

  set_target_properties(${TARGET_NAME}
      PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

  install(TARGETS ${TARGET_NAME}
    EXPORT ${PROJECT_NAME}_Exports
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
      NAMELINK_SKIP
    # on Windows put the dlls into bin
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    # ... and the import lib into the devel package
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )

  install(EXPORT ${PROJECT_NAME}_Exports
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
    NAMESPACE ${PROJECT_NAME}::
  )

  install(TARGETS ${TARGET_NAME}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
      NAMELINK_ONLY
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
endmacro()

build_ispcrt(SHARED ${PROJECT_NAME})
build_ispcrt(STATIC ${PROJECT_NAME}_static)

## Install headers ############################################################

install(FILES ispcrt.h ispcrt.hpp ispcrt.isph
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/ispcrt
)

## Configure CMake find_package() config files ################################

include(CMakePackageConfigHelpers)

configure_package_config_file(
  "${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in"
  "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
INSTALL_DESTINATION
  ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)

write_basic_package_version_file(
    "${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
  cmake/Findlevel_zero.cmake
  cmake/ispc.cmake
DESTINATION
  ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)
