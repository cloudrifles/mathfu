# Copyright 2014 Google Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
cmake_minimum_required(VERSION 2.8.12)

# Directory which contains the source for 3rd party libraries.
get_filename_component(
    third_party_root
    "${CMAKE_CURRENT_SOURCE_DIR}/../../../../external" REALPATH)
# Directory which contains source for FPL libraries.
get_filename_component(
    fpl_root "${CMAKE_CURRENT_SOURCE_DIR}/../../libs" REALPATH)

# If the dependencies directory exists, assume this is the root directory for
# all libraries required by this project.
set(dependencies_root "${CMAKE_SOURCE_DIR}/dependencies")
if(EXISTS "${dependencies_root}")
  set(third_party_root "${dependencies_root}")
  set(fpl_root "${dependencies_root}")
endif()

get_filename_component(
    vectorial_dir_default "${third_party_root}/vectorial" REALPATH)
get_filename_component(
    gtest_dir_default "${fpl_root}/googletest" REALPATH)

set(PROJECT_NAME MathFu)
project(MathFu)

# Set some variables describing file locations.
set(dependencies_vectorial_dir "${vectorial_dir_default}"
    CACHE PATH "Directory containing the Vectorial library.")
set(dependencies_gtest_dir "${gtest_dir_default}"
    CACHE PATH "Directory containing the googletest (gtest) library.")

# Options that control the build configuration.
# To configure MathFu flags per build target, see the
# mathfu_configure_flags() function.
option(mathfu_enable_simd "Use SIMD implementations when available." ON)
option(mathfu_build_benchmarks "Build MathFu benchmarks." ON)
option(mathfu_build_tests "Build MathFu unit tests." ON)

# Detect clang
if(CMAKE_CXX_COMPILER MATCHES ".*clang")
  set(CMAKE_COMPILER_IS_CLANGXX 1)
endif()

# Save the mathfu directory, store this in the cache so that it's globally
# accessible from mathfu_configure_flags().
set(mathfu_dir ${CMAKE_CURRENT_LIST_DIR} CACHE INTERNAL "mathfu directory")

# Set build options for ${target} that are required to build with MathFu.
# This takes the optional arguments "enable_simd force_padding".
#
# If enable_simd is specified it configures whether the project should be
# built with SIMD optimized functions.  If force_padding is specified and
# SIMD is enabled, some data structures are padded to work more efficiently
# with SIMD instructions.
#
# If enable_simd is not specified, the mathfu_enable_simd option is used.
# If force_padding isn't specified padding is enabled based upon the
# best general configuration for the target architecture.
function(mathfu_configure_flags target)
  set(enable_simd ${mathfu_enable_simd})

  # Add required includes to the target.
  target_include_directories(${target}
    PRIVATE ${mathfu_dir}/include ${dependencies_vectorial_dir}/include)

  # Parse optional arguments.
  set(additional_args ${ARGN})
  list(LENGTH additional_args num_additional_args)
  if(${num_additional_args} GREATER 0)
    list(GET additional_args 0 enable_simd)
  endif()
  if(${num_additional_args} GREATER 1)
    list(GET additional_args 1 force_padding)
  endif()

  # If the SIMD build option is enabled.
  if(enable_simd)
    # Enable SSE4.1 when building with GCC / Clang.
    # NOTE: It's also possible to build the library using using SSE2 with GCC /
    # Clang, change -msse4.1 to -msse2.
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR
        CMAKE_COMPILER_IS_CLANGXX)
      target_compile_options(${target} PRIVATE -msse4.1)
    endif()
    # Enable SSE2 by default when building with MSVC.
    if(MSVC)
      target_compile_options(${target} PRIVATE /arch:SSE2)
    endif()
    # Conditionally enable padding.
    if(DEFINED force_padding)
      if(force_padding)
        target_compile_definitions(${target} PRIVATE
          -DMATHFU_COMPILE_FORCE_PADDING=1)
      else()
        target_compile_definitions(${target} PRIVATE
          -DMATHFU_COMPILE_FORCE_PADDING=0)
      endif()
    endif()
  else()
    target_compile_definitions(${target} PRIVATE
      -DMATHFU_COMPILE_WITHOUT_SIMD_SUPPORT)
  endif()
endfunction()

# Modify CMAKE_C_FLAGS and CMAKE_CXX_FLAGS to enable a maximum reasonable
# warning level.
function(mathfu_enable_warnings target)
  get_target_property(target_compile_flags ${target} COMPILE_FLAGS)
  if(MSVC)
    target_compile_options(${target} PRIVATE /W4 /WX)
  elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR
      CMAKE_COMPILER_IS_CLANGXX)
    # Set the maximum warning level for gcc.
    target_compile_options(${target} PRIVATE -Wall -Wextra -Werror
      -Wno-long-long -Wno-variadic-macros)
  endif()
endfunction()

file(GLOB_RECURSE MATHFU_HEADERS ${CMAKE_SOURCE_DIR}/include/mathfu *.h)

if(mathfu_build_benchmarks)
  add_subdirectory(benchmarks)
endif()
if(mathfu_build_tests)
  add_subdirectory(unit_tests)
endif()
