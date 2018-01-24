# MIT License
#
# Copyright (c) 2018 Joerg Hallmann
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
# https://github.com/joerghall/cbt-devtool-gcc
#
cmake_minimum_required(VERSION 3.5)

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
    include(${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
else ()
    message(FATAL_ERROR "Unable to find buildtools, please check if the submodule is initialized")
endif ()

set(default_ARTIFACT url=https://h1grid.com/artifactory/cbt/devtools/${PACKAGE}/${VERSION}/${PACKAGE}-${VERSION}-${PLATFORM}.{EXTENTION}
                    version=1
)

add_toolset(python2 2.7.14.43.rcd588a99 config=default )
add_toolset(ninja 1.8.2.11.rc3e3b700 config=default )
# add_toolset(pigz 1.8.2.7.r8fe061a4 config=default OVERRIDE_VERSION_WINDOWS=2.3.1)

include(python2)

set(CMAKE_CXX_STANDARD 14)

project(cbt-devtool-msvc)

enable_testing()

include(ProcessorCount)
ProcessorCount(N)
message(STATUS "Processor count ${N}")
if(N EQUAL 0)
    set(N 4)
endif()

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_BINARY_DIR}/packages)

# Standard locations
#
set(SOURCE_UPLOAD_DIR ${CMAKE_BINARY_DIR}/upload)
set(SOURCE_DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/download)
set(SOURCE_INSTALL_DIR ${CMAKE_BINARY_DIR}/install)
set(SOURCE_SRC_DIR ${CMAKE_BINARY_DIR}/src)
set(TARGET_INSTALL_DIR ${CMAKE_BINARY_DIR}/build)

# Generate version information for package
#
execute_process(COMMAND git rev-list HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    RESULT_VARIABLE VERSION_RESULT
    OUTPUT_VARIABLE VERSION_DATA
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

string(REPLACE "\n" ";" VERSION_DATA ${VERSION_DATA} )
list(LENGTH VERSION_DATA VERSION_REVISION)
list(GET VERSION_DATA 0 VERSION_HASH)
string(SUBSTRING ${VERSION_HASH} 0 8 VERSION_SHORTHASH)

## Package specifics
##
set(PACKAGE_VERSION 7.2.0)
set(PACKAGE_NAME msvc-cl)
set(PACKAGE_PLATFORM windows)

set(PACKAGE_VERSION_PACKAGE ${PACKAGE_VERSION}.${VERSION_REVISION}.r${VERSION_SHORTHASH})
set(PACKAGE_INSTALL_DIR ${SOURCE_SRC_DIR}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/${PACKAGE_PLATFORM})

add_custom_target(package_build
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_DOWNLOAD_DIR}
    COMMAND copy /R "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Enterprise\\VC\\Tools\\MSVC" ${PACKAGE_INSTALL_DIR}
    WORKING_DIRECTORY ${SOURCE_DOWNLOAD_DIR}/${PACKAGE_NAME}
)

add_custom_target(package_tar
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_UPLOAD_DIR}
    COMMAND ${CMAKE_COMMAND} -E tar czf ${SOURCE_UPLOAD_DIR}/${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-${PACKAGE_PLATFORM}.tgz ${PACKAGE_NAME}
    WORKING_DIRECTORY ${SOURCE_SRC_DIR}
    DEPENDS package_build
)

add_custom_target(publish_package
    COMMAND ${python2_BIN} ${upload_py_COMMAND} --file ${SOURCE_UPLOAD_DIR}/${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-${PACKAGE_PLATFORM}.tgz --url https://h1grid.com:443/artifactory/cbt/devtools/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-${PACKAGE_PLATFORM}.tgz
    DEPENDS package_tar
)
