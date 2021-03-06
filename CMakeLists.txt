# Copyright 2017-2018, Intel Corporation
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
#     * Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#
#     * Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in
#       the documentation and/or other materials provided with the
#       distribution.
#
#     * Neither the name of the copyright holder nor the names of its
#       contributors may be used to endorse or promote products derived
#       from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.5)
project(pmemkv)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake)
set(SOURCE_FILES src/pmemkv.cc src/pmemkv.h
    src/engines/blackhole.h src/engines/blackhole.cc
    src/engines/kvtree3.h src/engines/kvtree3.cc
    src/engines/btree.h src/engines/btree.cc
    src/engines/btree/persistent_b_tree.h src/engines/btree/pstring.h
    src/engines/vmap.h src/engines/vmap.cc
    src/engines/vcmap.h src/engines/vcmap.cc
    src/engines/caching.h src/engines/caching.cc
)
set(TEST_FILES tests/pmemkv_test.cc tests/mock_tx_alloc.cc
	tests/engines/blackhole_test.cc
	tests/engines/btree_test.cc
	tests/engines/kvtree_test.cc
	tests/engines/vmap_test.cc
	tests/engines/vcmap_test.cc
    	tests/engines/caching_test.cc
)
set(3RDPARTY ${PROJECT_SOURCE_DIR}/3rdparty)
set(GTEST_VERSION 1.7.0)

find_package(PkgConfig QUIET)
include(ExternalProject)
include(FindThreads)
include(FindMemkind)
include(TBBGet)

if(NOT MEMKIND_FOUND)
	message(FATAL_ERROR "Memkind library not found")
endif()

tbb_get(TBB_ROOT tbb_root CONFIG_DIR TBB_DIR)
find_package(TBB REQUIRED COMPONENTS tbb)

set(GTEST_URL ${CMAKE_SOURCE_DIR}/googletest-${GTEST_VERSION}.zip)
if(EXISTS ${GTEST_URL})
# do nothing
else()
execute_process(COMMAND wget -O ${GTEST_URL} https://github.com/google/googletest/archive/release-${GTEST_VERSION}.zip)
endif()
ExternalProject_Add(gtest
    URL ${GTEST_URL}
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/gtest
    INSTALL_COMMAND ""
    CMAKE_ARGS -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
)
ExternalProject_Get_Property(gtest source_dir binary_dir)
add_library(libgtest IMPORTED STATIC GLOBAL)
add_dependencies(libgtest gtest)
set_target_properties(libgtest PROPERTIES
    "IMPORTED_LOCATION" "${binary_dir}/libgtest.a"
    "IMPORTED_LINK_INTERFACE_LIBRARIES" "${CMAKE_THREAD_LIBS_INIT}"
)
include_directories("${source_dir}/include")

if(PKG_CONFIG_FOUND)
    pkg_check_modules(PMEMOBJ++ REQUIRED libpmemobj++)
else()
    find_package(PMEMOBJ++ REQUIRED)
endif()

find_package(RapidJSON)
set(LIB_ACL $ENV{HOME}/acl/lib_acl)
set(LIB_ACL_CPP $ENV{HOME}/acl/lib_acl_cpp)
set(MEMCACHED_INCLUDE $ENV{HOME}/libmemcached-1.0.18)
set(RAPIDJSON $ENV{HOME})
include_directories(${PMEMOBJ++_INCLUDE_DIRS} ${MEMKIND_INCLUDE_DIRS} ${MEMCACHED_INCLUDE} ${LIB_ACL_CPP}/include/acl_cpp ${LIB_ACL}/include ${RAPIDJSON}) 
link_directories(${PMEMOBJ++_LIBRARY_DIRS})

add_library(pmemkv SHARED ${SOURCE_FILES})
target_link_libraries(pmemkv ${PMEMOBJ++_LIBRARIES} ${MEMKIND_LIBRARIES} ${TBB_IMPORTED_TARGETS})

add_executable(pmemkv_test ${TEST_FILES})
set(LIB_PROTOCOL $ENV{HOME}/acl/lib_protocol/lib)
target_link_libraries(pmemkv_test pmemkv libgtest -L${LIB_ACL_CPP}/lib  acl_cpp -L${LIB_PROTOCOL} protocol -L${LIB_ACL}/lib acl -L../../lib/linux64  pthread memcached ${CMAKE_DL_LIBS})
