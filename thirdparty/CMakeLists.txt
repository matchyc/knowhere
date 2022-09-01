# Copyright (C) 2019-2020 Zilliz. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under the License

add_compile_options( -O3 -fPIC -Wno-error -fopenmp )

if ( NOT KNOWHERE_VERBOSE_THIRDPARTY_BUILD )
    set( EP_LOG_OPTIONS LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1 LOG_DOWNLOAD 1 )
else ()
    set( EP_LOG_OPTIONS )
endif ()

set( MAKE_BUILD_ARGS "-j6" )

include( FetchContent )
set( FETCHCONTENT_BASE_DIR  ${KNOWHERE_BINARY_DIR}/3rdparty_download )
set( FETCHCONTENT_QUIET     OFF )

if( CUSTOM_THIRDPARTY_DOWNLOAD_PATH )
    set( THIRDPARTY_DOWNLOAD_PATH ${CUSTOM_THIRDPARTY_DOWNLOAD_PATH} )
else()
    set( THIRDPARTY_DOWNLOAD_PATH ${CMAKE_BINARY_DIR}/3rdparty_download/download )
endif()
message( STATUS "Thirdparty downloaded file path: ${THIRDPARTY_DOWNLOAD_PATH}" )

# ****************************** Thirdparty googletest ***************************************
if ( KNOWHERE_BUILD_TESTS )
    if( NOT TARGET gtest::gtest_main )
        add_subdirectory( gtest )
    endif()

    if( NOT TARGET benchmark::benchmark_main )
        add_subdirectory( google_benchmark )
    endif()
endif()

if ( LINUX AND ENABLE_PROFILING )
    add_subdirectory( profiler )
endif()

# ****************************** Thirdparty Faiss ***************************************
set( FAISS_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/faiss )

macro(build_faiss)
    message( STATUS "Building FAISS-${FAISS_VERSION} from source" )
    set( FAISS_PREFIX "${CMAKE_INSTALL_PREFIX}" )
    set( FAISS_INCLUDE_DIR "${FAISS_PREFIX}/include" )
    set( FAISS_STATIC_LIB
            "${FAISS_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${CMAKE_STATIC_LIBRARY_PREFIX}faiss${CMAKE_STATIC_LIBRARY_SUFFIX}" )

    if ( MACOS )
        # MACOS always report this error when build google test in faiss unittest:
        # clang-13: error: argument unused during compilation: '-L/usr/local/opt/libomp/lib' [-Werror,-Wunused-command-line-argument]
        # so disable faiss_test for MACOS
        set ( ENABLE_FAISS_UNIT_TEST OFF )
    else ()
        set ( ENABLE_FAISS_UNIT_TEST ${BUILD_UNIT_TEST} )
    endif ()

    # header only mman-win32 for MSYS
    if (MSYS)
        set(MMAN_INCLUDE_FLAGS "-I ${KNOWHERE_THIRDPARTY_SRC}/mman-win32")
    endif ()
    set( FAISS_CMAKE_ARGS
            "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
            "-DCMAKE_BUILD_TYPE=Release"
            "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
            "-DCMAKE_CUDA_COMPILER=${CMAKE_CUDA_COMPILER}"
            "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -fpermissive -I ${KNOWHERE_SOURCE_DIR} ${MMAN_INCLUDE_FLAGS}"
            "-DFAISS_ENABLE_PYTHON=OFF"
            "-DFAISS_ENABLE_GPU=${KNOWHERE_GPU_VERSION}"
            "-DBUILD_TESTING=${ENABLE_FAISS_UNIT_TEST}"
            )
    if(KNOWHERE_GPU_VERSION)
        set(FAISS_CMAKE_ARGS "${FAISS_CMAKE_ARGS}"
            "-DCMAKE_CUDA_ARCHITECTURES=75;70;61;60"
            "-DCMAKE_CUDA_FLAGS=-Xcompiler -fPIC -std=c++11 -D_FORCE_INLINES --expt-extended-lambda -I${CMAKE_SOURCE_DIR}"
            )
    endif()
    message( STATUS "Building FAISS with configure args -${FAISS_CMAKE_ARGS}" )
    #Notice: there will not set LDFLAGS value
    externalproject_add( faiss_ep
            SOURCE_DIR
            ${FAISS_SOURCE_DIR}
            ${EP_LOG_OPTIONS}
            CMAKE_ARGS ${FAISS_CMAKE_ARGS}
            BUILD_COMMAND
            ${MAKE} ${MAKE_BUILD_ARGS} all
            BUILD_IN_SOURCE
            1
            INSTALL_COMMAND
            ${MAKE} install
            BUILD_BYPRODUCTS
            ${FAISS_STATIC_LIB} )

    if ( NOT BLAS_FOUND )
        message( STATUS "OpenBLAS BOUNDED" )
        ExternalProject_Add_StepDependencies( faiss_ep configure openblas_ep knowhere_utils )
    endif()

    file( MAKE_DIRECTORY "${FAISS_INCLUDE_DIR}" )
    add_library( faiss STATIC IMPORTED )

    set_target_properties( faiss
            PROPERTIES
                IMPORTED_GLOBAL                 TRUE
                IMPORTED_LOCATION               "${FAISS_STATIC_LIB}"
                INTERFACE_INCLUDE_DIRECTORIES   "${FAISS_INCLUDE_DIR}")

    target_link_libraries( faiss INTERFACE knowhere_utils openblas )
    
    add_dependencies( faiss faiss_ep )

endmacro()

if ( KNOWHERE_WITH_FAISS AND NOT TARGET faiss_ep )
    if ( FAISS_SOURCE STREQUAL "AUTO" )
        # find_package(FAISS MODULE)
        if ( NOT FAISS_FOUND )
            build_faiss()
        endif ()
    elseif ( FAISS_SOURCE STREQUAL "BUNDLED")
        build_faiss()
    elseif ( FAISS_SOURCE STREQUAL "SYSTEM" )
        find_package( FAISS REQUIRED )
    endif ()

    get_target_property( FAISS_INCLUDE_DIR faiss INTERFACE_INCLUDE_DIRECTORIES )
    include_directories( SYSTEM "${FAISS_INCLUDE_DIR}" )
    link_directories( SYSTEM ${FAISS_PREFIX}/${CMAKE_INSTALL_LIBDIR}/ )
endif ()

# ****************************** Thirdparty DiskANN ***************************************

set( DISKANN_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/DiskANN )

macro(build_diskann)
    message( STATUS "Building DiskANN from source" )
    set( DISKANN_PREFIX "${CMAKE_INSTALL_PREFIX}" )
    set( DISKANN_INCLUDE_DIR "${DISKANN_PREFIX}/include" )
    set( DISKANN_STATIC_LIB
            "${DISKANN_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${CMAKE_STATIC_LIBRARY_PREFIX}diskann${CMAKE_STATIC_LIBRARY_SUFFIX}" )
    set( DISKANN_CMAKE_ARGS
            "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
            "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}"
            "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
            "-DCMAKE_CXX_FLAGS=-O3 -DELPP_THREAD_SAFE -fpermissive -I ${KNOWHERE_SOURCE_DIR} -I ${KNOWHERE_SOURCE_DIR}/thirdparty -I ${CMAKE_INSTALL_PREFIX}/include"       
            )
    message( STATUS "Building DISKANN with configure args -${DISKANN_CMAKE_ARGS}" )
    set( DISKANN_BUILD_DIR "${DISKANN_SOURCE_DIR}/build")
    ExternalProject_Add( diskann_ep
        SOURCE_DIR
        ${DISKANN_SOURCE_DIR}
        PATCH_COMMAND  mkdir -p build
        BINARY_DIR ${DISKANN_BUILD_DIR}
        CMAKE_ARGS ${DISKANN_CMAKE_ARGS}
        BUILD_COMMAND cmake .. ${DISKANN_CMAKE_ARGS} COMMAND ${MAKE} ${MAKE_BUILD_ARGS}
        INSTALL_COMMAND
        ${MAKE} install
        BUILD_BYPRODUCTS
        ${DISKANN_STATIC_LIB} 
        BUILD_ALWAYS 1)

        if ( NOT BLAS_FOUND )
            message( STATUS "OpenBLAS BOUNDED" )
            ExternalProject_Add_StepDependencies( diskann_ep configure openblas_ep knowhere_utils )
        endif()

        add_library( diskann STATIC IMPORTED )

        set_target_properties( diskann
        PROPERTIES
            IMPORTED_GLOBAL                 TRUE
            IMPORTED_LOCATION               "${DISKANN_STATIC_LIB}"
            INTERFACE_INCLUDE_DIRECTORIES   "${DISKANN_INCLUDE_DIR}")

        target_link_libraries( diskann INTERFACE knowhere_utils openblas )
        add_dependencies( diskann diskann_ep )
endmacro()

if ( KNOWHERE_WITH_DISKANN AND NOT TARGET diskann_ep )
    if ( DiskANN_SOURCE STREQUAL "AUTO")
        if (NOT DiskANN_FOUND)
            build_diskann()
        endif ()
    endif()
    include_directories( SYSTEM ${DISKANN_INCLUDE_DIR})
    link_directories( SYSTEM  "${DISKANN_PREFIX}/${CMAKE_INSTALL_LIBDIR}/" )
endif()

# ****************************** Thirdparty Folly ***************************************

set ( FOLLY_VERSION v2022.08.29.00 )
if ( DEFINED ENV{MILVUS_FOLLY_URL} )
    set( FOLLY_SOURCE_URL "$ENV{MILVUS_FOLLY_URL}" )
else()
    set( FOLLY_SOURCE_URL "https://github.com/facebook/folly/releases/download/${FOLLY_VERSION}/folly-${FOLLY_VERSION}.tar.gz" )
endif()

message( STATUS "Building folly-${FOLLY_VERSION} from source" )

set( FOLLY_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/folly )
set( FOLLY_PREFIX "${CMAKE_INSTALL_PREFIX}" )
set( FOLLY_INCLUDE_DIR "${FOLLY_PREFIX}/include" )
set( FOLLY_INSTALLED_DIR "${FOLLY_SOURCE_DIR}/installed/folly" )
set( FOLLY_OUTPUT_DIR "${FOLLY_PREFIX}/${CMAKE_INSTALL_LIBDIR}/")
ExternalProject_Add( folly_ep
    # GIT_REPOSITORY https://github.com/facebook/folly.git
    # GIT_TAG main
    URL "${FOLLY_SOURCE_URL}"
    # URL_MD5 "0b6de34c0ca84a4f89d72a3380de9391"
    BINARY_DIR ${FOLLY_SOURCE_DIR}
    SOURCE_DIR ${FOLLY_SOURCE_DIR}
    # CMAKE_ARGS "-DCMAKE_CXX_FLAGS= -I ${FOLLY_SOURCE_DIR} -I ${FOLLY_INSTALLED_DIR}"
    # PATCH_COMMAND bash /build/fbcode_builder/getdeps.py install-system-deps --recursive folly
    CONFIGURE_COMMAND python3 ${FOLLY_SOURCE_DIR}/build/fbcode_builder/getdeps.py --scratch-path=${FOLLY_SOURCE_DIR} --allow-system-packages -v build folly --extra-b2-args=cxxflags=-fPIC --no-test --extra-cmake-defines={"CMAKE_POSITION_INDEPENDENT_CODE":"ON"}
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    BUILD_ALWAYS 1)


ExternalProject_Add_StepDependencies( folly_ep configure knowhere_utils )

add_library( folly STATIC IMPORTED )

set_target_properties( folly
    PROPERTIES
        IMPORTED_GLOBAL                 TRUE
        IMPORTED_LOCATION               "${FOLLY_OUTPUT_DIR}folly/${CMAKE_STATIC_LIBRARY_PREFIX}folly${CMAKE_STATIC_LIBRARY_SUFFIX}"
        INTERFACE_INCLUDE_DIRECTORIES   "${FOLLY_INCLUDE_DIR}")

target_link_libraries( folly INTERFACE knowhere_utils )
add_dependencies( folly folly_ep )

install(DIRECTORY "${FOLLY_INSTALLED_DIR}/include/folly"
        DESTINATION "include"
        FILES_MATCHING
        PATTERN "*.h"
        )

install(DIRECTORY "${FOLLY_INSTALLED_DIR}/lib/"
        DESTINATION "lib/folly"
        )

install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/annoy/"
        DESTINATION "include/annoy"
        FILES_MATCHING
        PATTERN "*.h"
        )

install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/faiss/"
        DESTINATION "include/faiss"
        FILES_MATCHING
        PATTERN "*.h"
        )

install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/hnswlib/"
        DESTINATION "include/hnswlib"
        FILES_MATCHING
        PATTERN "*.h"
        )

install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/easyloggingpp/"
        DESTINATION "include/easyloggingpp"
        FILES_MATCHING 
        PATTERN "*.h" 
)


if ( KNOWHERE_WITH_DISKANN AND TARGET diskann)
    install( DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/DiskANN/"
    DESTINATION "include/DiskANN"
    FILES_MATCHING
    PATTERN "*.h"
    PATTERN ".git*" EXCLUDE
    )
endif()
