#============================================================================
#
# Program: Bender
#
# Copyright (c) Kitware Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0.txt
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
#============================================================================

cmake_minimum_required(VERSION 2.8.10)

#-----------------------------------------------------------------------------
# Must be called before "project"
include(CMake/BenderOSXVariables.cmake)

#-----------------------------------------------------------------------------
#
set(kit Bender)
project(${kit})

#-----------------------------------------------------------------------------
# See http://cmake.org/cmake/help/cmake-2-8-docs.html#section_Policies for details
#-----------------------------------------------------------------------------
if(POLICY CMP0016)
  cmake_policy(SET CMP0016 NEW)
endif()
if(POLICY CMP0017)
  cmake_policy(SET CMP0017 OLD)
endif()

#-----------------------------------------------------------------------------
# Update CMake module path
# Note: FindXXX.cmake script specific to utility should be copied into Utilities/CMake
#
set(${kit}_CMAKE_DIR "${${kit}_SOURCE_DIR}/CMake")
set(CMAKE_MODULE_PATH  ${${kit}_CMAKE_DIR} ${CMAKE_MODULE_PATH})

#-----------------------------------------------------------------------------
# Library mode: SHARED (default) or STATIC
if(NOT DEFINED BUILD_SHARED_LIBS)
  option(BUILD_SHARED_LIBS "Build Bender with shared libraries." ON)
  mark_as_advanced(BUILD_SHARED_LIBS)
endif()
set(${kit}_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})

set(${kit}_INSTALL_BIN_DIR "bin")
set(${kit}_INSTALL_LIB_DIR "bin")
set(${kit}_INSTALL_INCLUDE_DIR "include")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${${kit}_BINARY_DIR}/${${kit}_INSTALL_BIN_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${${kit}_BINARY_DIR}/${${kit}_INSTALL_LIB_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${${kit}_BINARY_DIR}/${${kit}_INSTALL_LIB_DIR})

#-----------------------------------------------------------------------------
# Set a default build type if none was specified
#
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Debug' as none was specified.")
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

#-----------------------------------------------------------------------------
# Superbuild Option - Enabled by default
#
option(${kit}_SUPERBUILD "Build ${kit} and the projects it depends on via SuperBuild.cmake." ON)
mark_as_advanced(${kit}_SUPERBUILD)

#-----------------------------------------------------------------------------
# Prerequisites
#-----------------------------------------------------------------------------
find_package(Git REQUIRED)

#-----------------------------------------------------------------------------
# Project version number.
#
set(${kit}_VERSION_MAJOR 0)
set(${kit}_VERSION_MINOR 1)
set(${kit}_VERSION_PATCH 0)
# Releases define a tweak level
#set(${kit}_VERSION_TWEAK 0)
#set(${kit}_VERSION_RC 1)

include(BenderExtractRepositoryInfo)
bender_extract_repository_info(VAR_PREFIX ${kit}) # Used to configure ${kit}VersionConfigure.h.in

if (${kit}_WC_LAST_CHANGED_DATE)
  string(REGEX REPLACE ".*([0-9][0-9][0-9][0-9]\\-[0-9][0-9]\\-[0-9][0-9]).*" "\\1"
    ${kit}_BUILDDATE "${${kit}_WC_LAST_CHANGED_DATE}")
  set(${kit}_VERSION_UID "${(${kit}_BUILDDATE}")
else()
  set(${kit}_VERSION_UID "${${kit}_WC_REVISION}")
endif()

set(${kit}_VERSION "${${kit}_VERSION_MAJOR}.${${kit}_VERSION_MINOR}")
set(${kit}_VERSION_FULL "${${kit}_VERSION}.${${kit}_VERSION_PATCH}")
if(${kit}_VERSION_RC)
  set(${kit}_VERSION_FULL "${${kit}_VERSION_FULL}-rc${${kit}_VERSION_RC}")
endif()
if(NOT DEFINED ${kit}_VERSION_TWEAK)
  set(${kit}_VERSION_FULL "${${kit}_VERSION_FULL}-${${kit}_VERSION_UID}")
endif()

#-----------------------------------------------------------------------------
# Slicer configuration
#-----------------------------------------------------------------------------
add_subdirectory(Modules)

#-----------------------------------------------------------------------------
# Superbuild script
#

if(${kit}_SUPERBUILD)
  set(${kit}_SUPERBUILD_DIR ${${kit}_CMAKE_DIR}/Superbuild)
  add_subdirectory(${${kit}_SUPERBUILD_DIR})
  return()
endif()

# --------------------------------------------------------------------------
# Configure and install headers
# --------------------------------------------------------------------------
configure_file(
  ${${kit}_CMAKE_DIR}/benderVersionConfigure.h.in
  ${CMAKE_CURRENT_BINARY_DIR}/benderVersionConfigure.h
  )
set_property(GLOBAL APPEND PROPERTY ${kit}_INCLUDE_DIRS
  ${CMAKE_CURRENT_BINARY_DIR}
  )

# --------------------------------------------------------------------------
# Sub projects
# --------------------------------------------------------------------------
include(BenderExportLibrary)

enable_testing()
add_subdirectory(Libs)

#
# The following variables have been set by each sub directory project.
# They are used to configure ${kit}Config.cmake.in.
get_property(${kit}_LIBRARIES GLOBAL PROPERTY ${kit}_LIBRARIES)
list(REMOVE_DUPLICATES ${kit}_LIBRARIES)

get_property(${kit}_INCLUDE_DIRS GLOBAL PROPERTY ${kit}_INCLUDE_DIRS)
list(REMOVE_DUPLICATES ${kit}_INCLUDE_DIRS)

get_property(${kit}_LIBRARY_DIRS GLOBAL PROPERTY ${kit}_LIBRARY_DIRS)
list(REMOVE_DUPLICATES ${kit}_LIBRARY_DIRS)

# --------------------------------------------------------------------------
# Install project
# --------------------------------------------------------------------------

configure_file(
  ${${kit}_CMAKE_DIR}/${kit}Config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${kit}Config.cmake
  )

