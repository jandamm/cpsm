cmake_minimum_required(VERSION 2.8.12)

project(cpsm)
option(PY3 "Build for python3 instead of python2." OFF)
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
include_directories(${PROJECT_SOURCE_DIR}/src)
set(CMAKE_BUILD_TYPE Release)
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU|Clang|Intel")
  add_compile_options(-std=c++0x)
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  # Versions of MSVC that support C++11 at all support it by default, so hope
  # for the best...
else()
  message(FATAL_ERROR "Unknown compiler - don't know how to turn on C++11.")
endif()
set(CMAKE_MACOSX_RPATH 1)

find_package(Threads REQUIRED)

set(Boost_USE_MULTITHREADED ON)
find_package(Boost REQUIRED COMPONENTS program_options)
include_directories(${Boost_INCLUDE_DIRS})

if(PY3)
    set(Python_ADDITIONAL_VERSIONS 3.8 3.7 3.6 3.5 3.4 3.3)
    find_package(PythonInterp 3 REQUIRED)
else()
    set(Python_ADDITIONAL_VERSIONS 2.7 2.6)
    find_package(PythonInterp REQUIRED)
endif()
find_package(PythonConfig REQUIRED)

find_package(ICU)
if(ICU_FOUND)
  include_directories(${ICU_INCLUDE_DIRS})
  add_definitions(-DCPSM_CONFIG_ICU=1)
endif()

add_library(cpsm_core src/str_util.cc)
if(ICU_FOUND)
  target_link_libraries(cpsm_core ${ICU_LIBRARIES})
endif()
target_link_libraries(cpsm_core ${CMAKE_THREAD_LIBS_INIT})
set_target_properties(cpsm_core PROPERTIES COMPILE_FLAGS "-fPIC")

add_library(cpsm_py SHARED src/ctrlp_util.cc src/python_extension.cc)
target_link_libraries(cpsm_py cpsm_core)
set_target_properties(cpsm_py PROPERTIES COMPILE_FLAGS ${PYTHON_COMPILE_FLAGS})
set_target_properties(cpsm_py PROPERTIES LINK_FLAGS ${PYTHON_LINK_FLAGS})
set_target_properties(cpsm_py PROPERTIES PREFIX "")
if(APPLE)
  set_target_properties(cpsm_py PROPERTIES SUFFIX ".so")
endif()
install(TARGETS cpsm_py DESTINATION ${PROJECT_SOURCE_DIR}/autoload)
install(TARGETS cpsm_py DESTINATION ${PROJECT_SOURCE_DIR}/bin)

add_executable(cpsm_cli src/cpsm_cli_main.cc)
target_link_libraries(cpsm_cli cpsm_core ${Boost_PROGRAM_OPTIONS_LIBRARIES})
install(TARGETS cpsm_cli DESTINATION ${PROJECT_SOURCE_DIR}/bin)

enable_testing()

add_executable(matcher_test src/matcher_test.cc)
target_link_libraries(matcher_test cpsm_core)
add_test(matcher_test matcher_test)
