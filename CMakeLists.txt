# Copyright (c) 2016 PaddlePaddle Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License

cmake_minimum_required(VERSION 3.0)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(PADDLE_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(PADDLE_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})

include(system)

project(paddle CXX C)
message(STATUS "CXX compiler: ${CMAKE_CXX_COMPILER}, version: "
        "${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "C compiler: ${CMAKE_C_COMPILER}, version: "
        "${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION}")

find_package(Sphinx)
if(NOT CMAKE_CROSSCOMPILING)
    find_package(CUDA QUIET)
endif(NOT CMAKE_CROSSCOMPILING)
find_package(Git REQUIRED)
find_package(Threads REQUIRED)

include(simd)

################################ Configurations #######################################
option(WITH_GPU         "Compile PaddlePaddle with NVIDIA GPU"          ${CUDA_FOUND})
option(WITH_AMD_GPU     "Compile PaddlePaddle with AMD GPU"             OFF)
option(WITH_AVX         "Compile PaddlePaddle with AVX intrinsics"      ${AVX_FOUND})
option(WITH_MKL         "Compile PaddlePaddle with MKL support."        ${AVX_FOUND})
option(WITH_TENSORRT    "Compile PaddlePaddle with TensorRT support."   OFF)
option(WITH_DSO         "Compile PaddlePaddle with dynamic linked CUDA" ON)
option(WITH_TESTING     "Compile PaddlePaddle with unit testing"        OFF)
option(WITH_SWIG_PY     "Compile PaddlePaddle with inference api"       ON)
option(WITH_STYLE_CHECK "Compile PaddlePaddle with style check"         ON)
option(WITH_PYTHON      "Compile PaddlePaddle with python interpreter"  ON)
option(WITH_DOUBLE      "Compile PaddlePaddle with double precision"    OFF)
option(WITH_RDMA        "Compile PaddlePaddle with RDMA support"        OFF)
option(WITH_TIMER       "Compile PaddlePaddle with stats timer"         OFF)
option(WITH_PROFILER    "Compile PaddlePaddle with GPU profiler"        OFF)
option(WITH_DOC         "Compile PaddlePaddle with documentation"       OFF)
option(WITH_COVERAGE    "Compile PaddlePaddle with code coverage"       OFF)
option(COVERALLS_UPLOAD "Package code coverage data to coveralls"       OFF)
option(ON_TRAVIS        "Exclude special unit test on Travis CI"        OFF)
option(WITH_C_API       "Compile PaddlePaddle with C-API(Prediction)"   OFF)
option(WITH_FLUID_ONLY  "Compile PaddlePaddle fluid only"               OFF)
option(WITH_GOLANG      "Compile PaddlePaddle with GOLANG"              OFF)
option(GLIDE_INSTALL    "Download and install go dependencies "         ON)
option(USE_NNPACK       "Compile PaddlePaddle with NNPACK library"      OFF)
option(WITH_DISTRIBUTE  "Compile with grpc distributed support"         OFF)
option(USE_EIGEN_FOR_BLAS   "Use matrix multiplication in Eigen"        OFF)
option(WITH_ARM_FP16    "Use half precision support on armv8.2-a cpu"   OFF)
option(WITH_FAST_BUNDLE_TEST    "Bundle tests that can be run in a single process together to reduce launch overhead"   OFF)

# CMAKE_BUILD_TYPE
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING
      "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel"
      FORCE)
endif()

if(ANDROID OR IOS)
    if(ANDROID)
        if(${CMAKE_SYSTEM_VERSION} VERSION_LESS "16")
            message(FATAL_ERROR "Unsupport standalone toolchains with Android API level lower than 16")
        endif()
    endif()

    set(WITH_GPU OFF CACHE STRING
        "Disable GPU when cross-compiling for Android and iOS" FORCE)
    set(WITH_AVX OFF CACHE STRING
        "Disable AVX when cross-compiling for Android and iOS" FORCE)
    set(WITH_PYTHON OFF CACHE STRING
        "Disable PYTHON when cross-compiling for Android and iOS" FORCE)
    set(WITH_RDMA OFF CACHE STRING
        "Disable RDMA when cross-compiling for Android and iOS" FORCE)
    set(WITH_MKL OFF CACHE STRING
        "Disable MKL when cross-compiling for Android and iOS" FORCE)
    set(WITH_GOLANG OFF CACHE STRING
        "Disable golang when cross-compiling for Android and iOS" FORCE)

    # Compile PaddlePaddle mobile inference library
    if (NOT WITH_C_API)
        set(WITH_C_API ON CACHE STRING
            "Always compile the C_API when cross-compiling for Android and iOS" FORCE)
    endif()
    set(MOBILE_INFERENCE ON)
    add_definitions(-DPADDLE_MOBILE_INFERENCE)
endif()

set(THIRD_PARTY_PATH "${CMAKE_BINARY_DIR}/third_party" CACHE STRING
  "A path setting third party libraries download & build directories.")

if (WITH_C_API AND WITH_PYTHON)
  message(WARNING "It is suggest not embedded a python interpreter in Paddle "
    "when using C-API. It will give an unpredictable behavior when using a "
    "different Python interpreter from compiling.")
endif()

if (WITH_C_API)
  set(WITH_FLUID_ONLY OFF CACHE STRING "Disable install fluid when compile the C_API" FORCE)
endif()

if(MOBILE_INFERENCE)
    set(THIRD_PARTY_BUILD_TYPE MinSizeRel)
else()
    set(THIRD_PARTY_BUILD_TYPE Release)
endif()

set(WITH_MKLML ${WITH_MKL})
if (WITH_MKL AND AVX2_FOUND)
    set(WITH_MKLDNN ON)
else()
    message(STATUS "Do not have AVX2 intrinsics and disabled MKL-DNN")
    set(WITH_MKLDNN OFF)
endif()

########################################################################################

include(external/mklml)     # download mklml package
include(external/zlib)      # download, build, install zlib
include(external/gflags)    # download, build, install gflags
include(external/glog)      # download, build, install glog
include(external/gtest)     # download, build, install gtest
include(external/protobuf)  # download, build, install protobuf
include(external/python)    # download, build, install python
include(external/openblas)  # download, build, install openblas
include(external/mkldnn)    # download, build, install mkldnn
include(external/swig)      # download, build, install swig
include(external/warpctc)   # download, build, install warpctc
include(external/boost)     # download boost
include(external/any)       # download libn::any
include(external/eigen)     # download eigen3
include(external/pybind11)  # download pybind11
include(external/cares)
include(external/grpc)
include(external/snappy)    # download snappy
include(external/snappystream)
include(external/threadpool)

include(cudnn)              # set cudnn libraries, must before configure
include(cupti)
include(configure)          # add paddle env configuration
include(generic)            # simplify cmake module
include(package)            # set paddle packages
include(cpplint)            # set paddle c++ style
include(ccache)             # set ccache for compilation
include(util)               # set unittest and link libs
include(rdma)               # set rdma libraries
include(flags)              # set paddle compile flags
include(version)            # set PADDLE_VERSION
include(coveralls)          # set code coverage
include(inference_lib)      # add paddle fluid inference libraries


include_directories("${PADDLE_SOURCE_DIR}")
include_directories("${PADDLE_SOURCE_DIR}/paddle/cuda/include")
include_directories("${CMAKE_CURRENT_BINARY_DIR}/proto")
include_directories("${CMAKE_CURRENT_BINARY_DIR}/go/pserver/client/c")

set(EXTERNAL_LIBS
    gflags
    glog
    ${CBLAS_LIBRARIES}
    protobuf
    zlib
    ${PYTHON_LIBRARIES}
)

if(WITH_GPU)
    include(cuda)
endif(WITH_GPU)

# TensorRT depends on GPU.
if (NOT WITH_GPU)
  set(WITH_TENSORRT OFF)
endif()

if(WITH_AMD_GPU)
    find_package(HIP)
    include(hip)
endif(WITH_AMD_GPU)

if(WITH_MKLML)
    list(APPEND EXTERNAL_LIBS ${MKLML_IOMP_LIB})
endif()

if(WITH_MKLDNN)
    list(APPEND EXTERNAL_LIBS ${MKLDNN_LIB})
endif()

if(USE_NNPACK)
    include(external/nnpack)
    list(APPEND EXTERNAL_LIBS ${NNPACK_LIBS})
endif(USE_NNPACK)

add_subdirectory(proto)

if(NOT MOBILE_INFERENCE)
    # "add_subdirectory(go)" should be placed after the following loine,
    # because it depends on paddle/optimizer.
    add_subdirectory(paddle/optimizer)
endif()

# "add_subdirectory(paddle)" and "add_subdirectory(python)" should be
# placed after this block, because they depends on it.
if(WITH_GOLANG)
    enable_language(Go)
    add_subdirectory(go)
endif(WITH_GOLANG)

set(PADDLE_PYTHON_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/python/build")

set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O3 -g -DNDEBUG")
set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O3 -g -DNDEBUG")

add_subdirectory(paddle)
if(WITH_PYTHON)
    add_subdirectory(python)
endif()

if(WITH_DOC)
    add_subdirectory(doc)
endif()
