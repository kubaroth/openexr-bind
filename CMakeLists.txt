cmake_minimum_required(VERSION 3.0)

#project(openexr)

include(ExternalProject)

find_package (Python 2.7 REQUIRED COMPONENTS Development)
find_package(Boost REQUIRED COMPONENTS system python)

set(CMAKE_CXX_STANDARD 11)
set(OFFLINE 0 CACHE BOOL "Skips download and uses tarballs downloaded into the root of the project")
set(EXR_VERSION 2.2.0 CACHE STRING "Major/Minor/Patch version of OpenEXR dependencies")

## Separate versoin into components
string(REPLACE "." " " VERSION_LIST ${EXR_VERSION})
separate_arguments(VERSION_LIST)
list(GET VERSION_LIST 0 MAJOR)
list(GET VERSION_LIST 1 MINOR)
list(GET VERSION_LIST 2 PATCH)
set(DSO_VERSION "${MAJOR}_${MINOR}")
message(STATUS "EXR_VERSION: " ${MAJOR}.${MINOR}.${PATCH})
message(STATUS "DSO_VERSION: " ${DSO_VERSION})

if(OFFLINE EQUAL 0)
  set (TAR_LOCATION http://download.savannah.nongnu.org/releases/openexr/)
else()
  set (TAR_LOCATION ${CMAKE_CURRENT_SOURCE_DIR})
endif()
message (STATUS "using location: ${TAR_LOCATION}")

ExternalProject_Add(
  "ilmbase"
  PREFIX "external"
  #URL http://download.savannah.nongnu.org/releases/openexr/ilmbase-2.2.0.tar.gz
  # URL ${CMAKE_CURRENT_SOURCE_DIR}/ilmbase-2.2.0.tar.gz
  URL ${TAR_LOCATION}/ilmbase-${EXR_VERSION}.tar.gz
  DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/ilmbase_download
  SOURCE_DIR ${CMAKE_BINARY_DIR}/ilmbase_src
  INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install_dir
  BUILD_COMMAND ""
  CMAKE_ARGS
  "-DCMAKE_BUILD_TYPE=Release"
  "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/install_dir"
)

# Create symlinks so that PyIlmbase does not complain.
add_custom_command(TARGET ilmbase POST_BUILD
  COMMAND ln -fs libIlmThread-${DSO_VERSION}.so.12 libIlmThread.so
  COMMAND ln -fs libImath-${DSO_VERSION}.so libImath.so
  COMMAND ln -fs libIexMath-${DSO_VERSION}.so libIexMath.so
  COMMAND ln -fs libIex-${DSO_VERSION}.so libIex.so
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/install_dir/lib/)

ExternalProject_Add(
  "openexr"
  PREFIX "external"
  # URL http://download.savannah.nongnu.org/releases/openexr/openexr-2.2.0.tar.gz
  # URL ${CMAKE_CURRENT_SOURCE_DIR}/openexr-2.2.0.tar.gz
  URL ${TAR_LOCATION}/openexr-${EXR_VERSION}.tar.gz
  DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/openexr_download
  SOURCE_DIR ${CMAKE_BINARY_DIR}/openexr_src
  INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install_dir
  BUILD_COMMAND ""
  CMAKE_ARGS
  "-DCMAKE_BUILD_TYPE=Release"
  "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/install_dir"
  "-DILMBASE_PACKAGE_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/install_dir"
  )

ExternalProject_Add_StepDependencies(openexr build ilmbase)

ExternalProject_Add(
  "pyilmbase"
  PREFIX "external"
  # URL http://download.savannah.gnu.org/releases/openexr/pyilmbase-2.2.0.tar.gz
  # URL /u/kr/SRC/openexr-bind/pyilmbase-2.2.0.tar.gz
  URL ${TAR_LOCATION}/pyilmbase-${EXR_VERSION}.tar.gz
  DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/pyilmbase_download
  SOURCE_DIR ${CMAKE_BINARY_DIR}/pyilmbase_src
  INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install_dir
  CONFIGURE_COMMAND ${CMAKE_BINARY_DIR}/openexr_src/./configure
  --prefix=${CMAKE_CURRENT_SOURCE_DIR}/install_dir PKG_CONFIG_PATH=../../install_dir/lib/pkgconfig --disable-ilmbasetest
  BUILD_COMMAND "make"
  )

ExternalProject_Add_StepDependencies(pyilmbase build ilmbase openexr)

ExternalProject_Add(
  "pybind11"
  PREFIX "external"
  GIT_REPOSITORY https://github.com/pybind/pybind11.git
  DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/pybind_download
  SOURCE_DIR ${CMAKE_BINARY_DIR}/pybind_src
  INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install_dir
  BUILD_COMMAND ""
  CMAKE_ARGS
  "-DCMAKE_BUILD_TYPE=Release"
  "-DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/install_dir"
  "-DPYBIND11_TEST=0"
 )

add_library(example SHARED src/example.cpp)         # sources
SET_TARGET_PROPERTIES(example PROPERTIES PREFIX "") # remove lib from library prefix
set(CMAKE_CXX_FLAGS "-fno-strict-aliasing -fwrapv")
add_dependencies(example pybind11 pyilmbase)

target_include_directories(example PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/install_dir/include/ ${Python_INCLUDE_DIR})
