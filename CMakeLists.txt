#
# Copyright 2020 Autodesk
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
#
cmake_minimum_required(VERSION 3.13...3.17)

project(maya-usd)

#------------------------------------------------------------------------------
# options
#------------------------------------------------------------------------------
option(BUILD_MAYAUSD_LIBRARY "Build Core USD libraries." ON)
option(BUILD_ADSK_PLUGIN "Build Autodesk USD plugin." ON)
option(BUILD_PXR_PLUGIN "Build the Pixar USD plugin and libraries." ON)
option(BUILD_AL_PLUGIN "Build the Animal Logic USD plugin and libraries." ON)
# Default value made dependent on maya + usd version - see below
#   Convert back to a normal option if we no longer support maya-2019, or
#   no longer support USD-20.11
#option(BUILD_HDMAYA "Build the Maya-To-Hydra plugin and scene delegate." ON)
option(BUILD_RFM_TRANSLATORS "Build translators for RenderMan for Maya shaders." ON)
option(BUILD_TESTS "Build tests." ON)
option(BUILD_STRICT_MODE "Enforce all warnings as errors." ON)
option(BUILD_SHARED_LIBS "Build libraries as shared or static." ON)
option(BUILD_WITH_PYTHON_3 "Build with python 3." OFF)
set(BUILD_WITH_PYTHON_3_VERSION 3.7 CACHE STRING "The version of Python 3 to build with")
option(CMAKE_WANT_UFE_BUILD "Enable building with UFE (if found)." ON)
option(CMAKE_WANT_MATERIALX_BUILD "Enable building with MaterialX (experimental)." OFF)

set(PXR_OVERRIDE_PLUGINPATH_NAME PXR_PLUGINPATH_NAME
    CACHE STRING "Name of env var USD searches to find plugins")

#------------------------------------------------------------------------------
# internal flags to control build
#------------------------------------------------------------------------------
# MAYAUSD_TO_USD_RELATIVE_PATH : Set this variable to any relative path from install
#                                folder to USD location. If defined will set relative
#                                rpaths for USD libraries.

#------------------------------------------------------------------------------
# global options
#------------------------------------------------------------------------------
# Avoid noisy install messages
set(CMAKE_INSTALL_MESSAGE "NEVER")

set(CMAKE_OSX_ARCHITECTURES "x86_64")

set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules
)

# Use RUNPATH instead of RPATH for all shared libs, executables and modules on Linux
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux") # IS_LINUX not yet defined
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--enable-new-dtags")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--enable-new-dtags")
    set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--enable-new-dtags")
endif()

#------------------------------------------------------------------------------
# modules and definitions
#------------------------------------------------------------------------------
include(cmake/utils.cmake)
find_package(Maya 2018 REQUIRED)

if (BUILD_MAYAUSD_LIBRARY)
    include(cmake/mayausd_version.info)
    set(MAYAUSD_VERSION "${MAYAUSD_MAJOR_VERSION}.${MAYAUSD_MINOR_VERSION}.${MAYAUSD_PATCH_LEVEL}")
endif()

if (DEFINED PYTHON_INCLUDE_DIR AND DEFINED PYTHON_LIBRARIES AND DEFINED Python_EXECUTABLE)
    SET(PYTHON_INCLUDE_DIRS "${PYTHON_INCLUDE_DIR}")
    SET(PYTHONLIBS_FOUND TRUE)
    # Use the Python module to find the python lib.
    if(BUILD_WITH_PYTHON_3)
        find_package(Python ${BUILD_WITH_PYTHON_3_VERSION} EXACT REQUIRED COMPONENTS Interpreter)
    else()
        find_package(Python 2.7 EXACT REQUIRED COMPONENTS Interpreter)
    endif()
    if(NOT Python_Interpreter_FOUND)
        set(PYTHONLIBS_FOUND FALSE)
    endif()
endif()
if (NOT PYTHONLIBS_FOUND)
    include(cmake/python.cmake)
endif()
message(STATUS "Build MayaUSD with Python3 = " ${BUILD_WITH_PYTHON_3})
message(STATUS "   PYTHON_INCLUDE_DIRS = ${PYTHON_INCLUDE_DIRS}")
message(STATUS "   PYTHON_LIBRARIES    = ${PYTHON_LIBRARIES}")
message(STATUS "   Python_EXECUTABLE   = ${Python_EXECUTABLE}")

include(cmake/jinja.cmake)

find_package(USD 0.20.05 REQUIRED)
if (CMAKE_WANT_MATERIALX_BUILD)
    # Requires at least USD 21.08 for hdMtlx module and USD must have been built with MaterialX:
    if(NOT TARGET hdMtlx)
        set(CMAKE_WANT_MATERIALX_BUILD OFF)
        message(WARNING "Disabling MaterialX VP2 rendering: it is not supported by this USD package.")
    endif()
endif()
include(cmake/usd.cmake)

if( MAYA_APP_VERSION LESS 2020 AND PXR_VERSION GREATER 2005 AND PXR_VERSION LESS 2102 )
    # Default to OFF if maya==2019 AND USD==(20.08||20.11)
    #   Due to a problem with glew - if you force on with these versions, you
    #   will probably need to LD_PRELOAD your glew lib.
    #   See:
    #      https://github.com/Autodesk/maya-usd/discussions/1017
    option(BUILD_HDMAYA "Build the Maya-To-Hydra plugin and scene delegate." OFF)
else()
    option(BUILD_HDMAYA "Build the Maya-To-Hydra plugin and scene delegate." ON)
endif()

if(${MAYA_APP_VERSION} STRLESS "2019" AND CMAKE_WANT_UFE_BUILD)
    set(CMAKE_WANT_UFE_BUILD OFF)
    message(AUTHOR_WARNING "========================================================================================= \
                            UFE is not available in Maya${MAYA_MAJOR_VERSION}. Maya2019 and later are supported only. \
                            ========================================================================================= ")
endif()

if(CMAKE_WANT_UFE_BUILD)
    find_package(UFE QUIET)
    if(UFE_FOUND)
        message(STATUS "Building with UFE ${UFE_VERSION} features enabled.")
    else()
        message(STATUS "UFE not found. UFE features will be disabled.")
    endif()
endif()

if(CMAKE_WANT_MATERIALX_BUILD AND MAYA_LIGHTAPI_VERSION LESS 2)
    set(CMAKE_WANT_MATERIALX_BUILD OFF)
    message(WARNING "Disabling MaterialX VP2 rendering: it is not supported by this version of Maya.")
endif()

if(TARGET hdMtlx)
    # MaterialX was built into USD. We need to know where to find MaterialX targets.
    find_package(MaterialX REQUIRED)
    if(TARGET MaterialXCore)
        message(STATUS "Found MaterialX")
    endif()
endif()

if(DEFINED QT_LOCATION)
    if(NOT DEFINED QT_VERSION)
        if(${MAYA_APP_VERSION} STRLESS_EQUAL "2019")
            set(QT_VERSION "5.6")
        else()
            set(QT_VERSION "5.12")
        endif()
    endif()
    set(CMAKE_PREFIX_PATH "${QT_LOCATION}")
    find_package(Qt5 ${QT_VERSION} COMPONENTS Core Gui Widgets REQUIRED)
    if(Qt5_FOUND)
        message(STATUS "Building with Qt features enabled.")
    endif()
else()
    message(STATUS "QT_LOCATION not set. Building Qt features will be disabled.")
endif()

#------------------------------------------------------------------------------
# compiler configuration
#------------------------------------------------------------------------------
include(cmake/compiler_config.cmake)

#------------------------------------------------------------------------------
# gulark filesystem
#------------------------------------------------------------------------------
include(cmake/gulark.cmake)

#------------------------------------------------------------------------------
# test
#------------------------------------------------------------------------------
if (BUILD_TESTS)
    include(cmake/googletest.cmake)
    include(cmake/test.cmake)
    fetch_googletest()
    enable_testing()
    add_subdirectory(test)
    set(MAYAUSD_GTEST_PATH "PATH+:=lib/gtest")
endif()

#------------------------------------------------------------------------------
# lib
#------------------------------------------------------------------------------
if (BUILD_MAYAUSD_LIBRARY)
    add_subdirectory(lib)
endif()

#------------------------------------------------------------------------------
# plugins
#------------------------------------------------------------------------------
if (BUILD_PXR_PLUGIN)
    add_subdirectory(plugin/pxr)
endif()

if (BUILD_AL_PLUGIN)
    if(BUILD_USDMAYA_PXR_TRANSLATORS AND NOT BUILD_PXR_PLUGIN)
        message(FATAL_ERROR "Animal logic's pixar translator depends on pixar plugin!")
    endif()

    add_subdirectory(plugin/al)
endif()

if (BUILD_ADSK_PLUGIN)
    add_subdirectory(plugin/adsk)
endif()

#------------------------------------------------------------------------------
# install
#------------------------------------------------------------------------------
if (DEFINED MAYAUSD_TO_USD_RELATIVE_PATH)
    set(USD_INSTALL_LOCATION "${CMAKE_INSTALL_PREFIX}/${MAYAUSD_TO_USD_RELATIVE_PATH}")
else()
    set(USD_INSTALL_LOCATION ${PXR_USD_LOCATION})
endif()


#------------------------------------------------------------------------------
# Maya module files
#------------------------------------------------------------------------------
if (BUILD_ADSK_PLUGIN)
    if (IS_WINDOWS)
        configure_file("modules/mayaUSD_Win.mod.template" ${PROJECT_BINARY_DIR}/mayaUSD.mod)
    else()
        configure_file("modules/mayaUSD.mod.template" ${PROJECT_BINARY_DIR}/mayaUSD.mod)
    endif()
    install(FILES ${PROJECT_BINARY_DIR}/mayaUSD.mod DESTINATION ${CMAKE_INSTALL_PREFIX})
endif()

if (BUILD_PXR_PLUGIN)
    if (IS_WINDOWS)
        configure_file("modules/pxrUSD_Win.mod.template" ${PROJECT_BINARY_DIR}/pxrUSD.mod)
    else()
        configure_file("modules/pxrUSD.mod.template" ${PROJECT_BINARY_DIR}/pxrUSD.mod)
    endif()
    install(FILES ${PROJECT_BINARY_DIR}/pxrUSD.mod DESTINATION ${CMAKE_INSTALL_PREFIX})
endif()

if (BUILD_AL_PLUGIN)
    if (IS_WINDOWS)
        configure_file("modules/alUSD_Win.mod.template" ${PROJECT_BINARY_DIR}/alUSD.mod)
    else()
        configure_file("modules/alUSD.mod.template" ${PROJECT_BINARY_DIR}/alUSD.mod)
    endif()
    install(FILES ${PROJECT_BINARY_DIR}/alUSD.mod DESTINATION ${CMAKE_INSTALL_PREFIX})
endif()
