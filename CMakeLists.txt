#
# Copyright (c) 2018-2019 Kris Jusiak (kris at jusiak dot net)
#
# Distributed under the Boost Software License, Version 1.0.
# (See accompanying file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
#
cmake_minimum_required(VERSION 3.21...3.25)
project(
        te
        VERSION 1.0.0
        LANGUAGES CXX
)

if(NOT DEFINED CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD
          20
          CACHE STRING "Default value for CXX_STANDARD property of targets."
          )
  option(CMAKE_CXX_STANDARD_REQUIRED "Default value for CXX_STANDARD_REQUIRED property of targets." YES)
  option(CMAKE_CXX_EXTENSIONS "Default value for CXX_EXTENSIONS property of targets." NO)
endif()

option(BOOST_TE_ENABLE_MEMCHECK "Run the unit tests and examples under valgrind if it is found." OFF)
option(BOOST_TE_ENABLE_COVERAGE "Run coverage." OFF)
option(BOOST_TE_ENABLE_SANITIZERS "Run static analysis." OFF)
option(BOOST_TE_ENABLE_CLANG_FORMAT "Run Clang Format." OFF)
option(BOOST_TE_BUILD_EXAMPLES "Build the examples" OFF)
option(BOOST_TE_BUILD_TESTS "Build the tests" OFF)
option(BOOST_TE_USE_WARNINGS_AS_ERRORS "Build the tests" ${PROJECT_IS_TOP_LEVEL})
option(BOOST_TE_DISABLE_MODULE "Disable te module" OFF)
option(BOOST_TE_ALLOW_CPM_USE "Do not reach out across network for CPM" OFF)

if(NOT CMAKE_SKIP_INSTALL_RULES)
  if(BOOST_TE_ALLOW_CPM_USE)
    # ---- Add dependencies via CPM ----
    # see https://github.com/cpm-cmake/CPM.cmake for more info

    include(cmake/CPM.cmake)

    # PackageProject.cmake will be used to make our target installable
    CPMAddPackage(
            NAME PackageProject.cmake
            GITHUB_REPOSITORY TheLartians/PackageProject.cmake
            VERSION 1.10.0
    )
  else()
    include(cmake/PackageProject.cmake)
  endif()
endif()

add_library(te INTERFACE)
add_library(Boost::te ALIAS te)

if(NOT DEFINED INCLUDE_INSTALL_DIR)
  set(INCLUDE_INSTALL_DIR include/${PROJECT_NAME}-${PROJECT_VERSION}/include)
endif()
target_include_directories(te INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:${INCLUDE_INSTALL_DIR}>)
target_compile_features(te INTERFACE cxx_std_20)

if(BOOST_UT_USE_WARNINGS_AS_ERRORS)
  include(cmake/WarningsAsErrors.cmake)
endif()

if(BOOST_TE_ENABLE_CLANG_FORMAT)
  add_custom_target(style)
  add_custom_command(TARGET style COMMAND find ${CMAKE_CURRENT_LIST_DIR}/example
    ${CMAKE_CURRENT_LIST_DIR}/include ${CMAKE_CURRENT_LIST_DIR}/test -iname
    "*.hpp" -or -iname "*.cpp" | xargs clang-format -i)
endif()
add_compile_options(-Wall -Wextra -Werror -pedantic -pedantic-errors)
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  add_compile_options(-fconcepts)
endif()

if (BOOST_TE_ENABLE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0")
endif()

if(BOOST_TE_ENABLE_SANITIZERS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -g -fno-omit-frame-pointer -fsanitize=address -fsanitize=leak -fsanitize=undefined")
endif()

if(BOOST_TE_DISABLE_MODULE)
  target_compile_definitions(ut INTERFACE BOOST_TE_DISABLE_MODULE)
endif()

if(NOT CMAKE_SKIP_INSTALL_RULES)
  # Create target Boost::te and install target
  packageProject(
          NAME ${PROJECT_NAME}
          VERSION ${PROJECT_VERSION}
          NAMESPACE Boost
          BINARY_DIR ${PROJECT_BINARY_DIR}
          INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include
          INCLUDE_DESTINATION ${INCLUDE_INSTALL_DIR}
          # XXX variant: DISABLE_VERSION_SUFFIX YES
          COMPATIBILITY SameMajorVersion
          # Note: only if needed i.e. DEPENDENCIES "fmt 7.1.3; span"
  )
endif()

include_directories(include)

find_program(MEMORYCHECK_COMMAND valgrind)
if (ENABLE_MEMCHECK AND MEMORYCHECK_COMMAND)
  function(test name)
    string(REPLACE "/" "_" out ${name})
    add_executable(${out} ${CMAKE_CURRENT_LIST_DIR}/${name}.cpp)
    add_custom_command(TARGET ${out} COMMAND ${MEMORYCHECK_COMMAND} --leak-check=full --error-exitcode=1 ./${out})
  endfunction()
else()
  function(test name)
    string(REPLACE "/" "_" out ${name})
    add_executable(${out} ${CMAKE_CURRENT_LIST_DIR}/${name}.cpp)
    add_custom_command(TARGET ${out} COMMAND ./${out})
  endfunction()
endif()

# Note: now we can use the target Boost::te
include(cmake/AddCustomCommandOrTest.cmake)

if(BOOST_TE_BUILD_EXAMPLES)
  add_subdirectory(example)
endif()

if(BOOST_TE_BUILD_TESTS)
  enable_testing()

  add_subdirectory(test)
endif()