cmake_minimum_required(VERSION 3.8)
project(dbow2_vendor)

include(ExternalProject)

find_package(ament_cmake REQUIRED)
find_package(OpenCV REQUIRED)

set(DBOW2_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vendor/DBoW2)
if(NOT EXISTS ${DBOW2_SOURCE_DIR}/CMakeLists.txt)
  message(FATAL_ERROR "Missing vendored DBoW2 source at ${DBOW2_SOURCE_DIR}.")
endif()

# Stage third-party install in the package build tree, then install into this
# package's final install prefix during `colcon build` install step.
set(DBOW2_STAGING_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/dbow2_install)

ExternalProject_Add(dbow2_src
  SOURCE_DIR ${DBOW2_SOURCE_DIR}
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/dbow2
  INSTALL_DIR ${DBOW2_STAGING_PREFIX}
  CMAKE_ARGS
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
    -DOpenCV_DIR=${OpenCV_DIR}
    -DDBOW2_ENABLE_PEDANTIC_WARNINGS=OFF
    -DBUILD_Demo=OFF
    -DBUILD_DBoW2=ON
  BUILD_BYPRODUCTS
    ${DBOW2_STAGING_PREFIX}/lib/libDBoW2.so
)

# Bring staged headers/libs/cmake config into this package install prefix.
install(
  DIRECTORY ${DBOW2_STAGING_PREFIX}/include/
  DESTINATION include
)

install(
  DIRECTORY ${DBOW2_STAGING_PREFIX}/lib/
  DESTINATION lib
  FILES_MATCHING
    PATTERN "libDBoW2*.so*"
    PATTERN "libDBoW2*.a"
    PATTERN "libDBoW2*.dylib"
)

install(
  DIRECTORY ${DBOW2_STAGING_PREFIX}/lib/cmake/DBoW2/
  DESTINATION lib/cmake/DBoW2
  OPTIONAL
)

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/dbow2_vendor-extras.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/dbow2_vendor-extras.cmake
  @ONLY
)

ament_export_include_directories(include)
ament_export_dependencies(OpenCV)

ament_package(
  CONFIG_EXTRAS ${CMAKE_CURRENT_BINARY_DIR}/dbow2_vendor-extras.cmake
)
