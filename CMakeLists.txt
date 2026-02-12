cmake_minimum_required(VERSION 3.20)
project(GRey LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(GREY_BUILD_EXAMPLES "Build examples" ON)
option(GREY_BUILD_UTEST    "Build unit tests" ON)

# ============================================================
# Auto header-only libraries from source/<name>/
# - Each direct subfolder source/<name>/ becomes target <name> (lowercased)
# - Include root is source/ so you can: #include <poisson/poisson.hpp>
# ============================================================

set(GREY_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/source")

if(NOT EXISTS "${GREY_SOURCE_DIR}")
  message(FATAL_ERROR "Missing required directory: source/")
endif()

file(GLOB GREY_SOURCE_CHILDREN LIST_DIRECTORIES true
     "${GREY_SOURCE_DIR}/*")

set(GREY_LIBS "")

foreach(child IN LISTS GREY_SOURCE_CHILDREN)
  if(IS_DIRECTORY "${child}")
    get_filename_component(lib_name "${child}" NAME)

    # sanitize -> valid CMake target name
    string(REGEX REPLACE "[^A-Za-z0-9_]" "_" lib_target "${lib_name}")
    # normalize to lowercase to avoid Windows case traps
    string(TOLOWER "${lib_target}" lib_target)

    if(lib_target STREQUAL "")
      message(FATAL_ERROR "Invalid source subdirectory name: '${lib_name}'")
    endif()

    add_library(${lib_target} INTERFACE)
    target_include_directories(${lib_target} INTERFACE "${GREY_SOURCE_DIR}")
    target_compile_features(${lib_target} INTERFACE cxx_std_20)

    list(APPEND GREY_LIBS ${lib_target})
  endif()
endforeach()

# ============================================================
# Function for examples
# In examples/<example>/CMakeLists.txt put ONLY:
#   add_example_with_libs(lib1 lib2 ...)
#
# - Builds examples/<example>/main.cpp
# - Target name: ex_<example>
# - Links requested libraries (names are case-insensitive; normalized to lowercase)
# ============================================================

function(add_example_with_libs)
  get_filename_component(_ex_dir "${CMAKE_CURRENT_SOURCE_DIR}" NAME)
  set(_target "ex_${_ex_dir}")

  set(_main "${CMAKE_CURRENT_SOURCE_DIR}/main.cpp")
  if(NOT EXISTS "${_main}")
    message(FATAL_ERROR "Example '${_ex_dir}' missing main.cpp at: ${_main}")
  endif()

  add_executable(${_target} "${_main}")

  set(_libs_to_link "")
  foreach(_lib IN LISTS ARGN)
    string(TOLOWER "${_lib}" _lib_lc)
    list(FIND GREY_LIBS "${_lib_lc}" _idx)
    if(_idx EQUAL -1)
      message(FATAL_ERROR
        "Example '${_ex_dir}' requests library '${_lib}', but no such library exists.\n"
        "Expected to find folder: source/${_lib}/ (library names are derived from source/* subfolders)"
      )
    endif()
    list(APPEND _libs_to_link "${_lib_lc}")
  endforeach()

  target_link_libraries(${_target} PRIVATE ${_libs_to_link})
endfunction()

# ============================================================
# Examples: auto-add every examples/<name>/ that has CMakeLists.txt
# ============================================================

if(GREY_BUILD_EXAMPLES)
  set(GREY_EXAMPLES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/examples")

  if(EXISTS "${GREY_EXAMPLES_DIR}")
    file(GLOB GREY_EXAMPLE_DIRS LIST_DIRECTORIES true
         "${GREY_EXAMPLES_DIR}/*")

    foreach(dir IN LISTS GREY_EXAMPLE_DIRS)
      if(IS_DIRECTORY "${dir}" AND EXISTS "${dir}/CMakeLists.txt")
        get_filename_component(_name "${dir}" NAME)
        add_subdirectory("examples/${_name}")
      endif()
    endforeach()
  endif()
endif()

# ============================================================
# Unit tests:
# - Requires: utests/main.cpp
# - Also compiles any utests/src/*.cpp (optional)
# - Builds executable: grey_utests
# - Custom target: utest (runs grey_utests)
# ============================================================

if(GREY_BUILD_UTEST)
  set(UTESTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/utests")
  set(UTESTS_MAIN "${UTESTS_DIR}/main.cpp")
  set(UTESTS_SRC_DIR "${UTESTS_DIR}/src")

  if(NOT EXISTS "${UTESTS_MAIN}")
    message(FATAL_ERROR
      "GREY_BUILD_UTEST is ON but missing unit test runner: utests/main.cpp\n"
      "Create it, even if minimal (int main(){return 0;})."
    )
  endif()

  file(GLOB_RECURSE CONFIGURE_DEPENDS UTESTS_CPP
       "${UTESTS_SRC_DIR}/*.cpp")

  add_executable(grey_utests
    "${UTESTS_MAIN}"
    ${UTESTS_CPP}
  )

  target_include_directories(grey_utests PRIVATE
    "${UTESTS_DIR}"
    "${UTESTS_SRC_DIR}"
  )

  # link all auto libraries to unit tests (simple + robust)
  target_link_libraries(grey_utests PRIVATE ${GREY_LIBS})

  add_custom_target(utest
    COMMAND grey_utests
    DEPENDS grey_utests
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Running unit tests"
  )
endif()
