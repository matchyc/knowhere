# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT license.

set(CMAKE_CXX_STANDARD 14)
# expected path for manual intel mkl installs

set(OMP_PATH /opt/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin/)
set(MKL_ROOT /opt/intel/oneapi/mkl/latest)

link_directories(${OMP_PATH} ${MKL_ROOT}/lib/intel64)
include_directories(${MKL_ROOT}/include)

# expected path for apt packaged intel mkl installs
link_directories(/usr/lib/x86_64-linux-gnu/mkl)
include_directories(/usr/include/mkl)

# compile flags and link libraries
# add_compile_options(-m64 -Wl,--no-as-needed) 
set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_RPATH} ${MKL_ROOT}/lib/intel64 ${OMP_PATH})
# link_libraries(mkl_intel_ilp64 mkl_intel_thread mkl_core iomp5 pthread m dl)
if(MSVC)
add_subdirectory(dll)
else()
#file(GLOB CPP_SOURCES *.cpp)
set(CPP_SOURCES ann_exception.cpp aux_utils.cpp distance.cpp index.cpp
linux_aligned_file_reader.cpp math_utils.cpp memory_mapper.cpp
partition_and_pq.cpp  pq_flash_index.cpp logger.cpp utils.cpp)
add_library(${PROJECT_NAME} SHARED ${CPP_SOURCES})
add_library(${PROJECT_NAME}_static STATIC ${CPP_SOURCES})
set_target_properties(${PROJECT_NAME}_static PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
set_property(TARGET ${PROJECT_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)
# add_library(${PROJECT_NAME}_s STATIC ${CPP_SOURCES})
endif()

# target_link_libraries(${PROJECT_NAME} ${DISKANN_TOOLS_TCMALLOC_LINK_OPTIONS} Boost::program_options ${DISKANN_ASYNC_LIB})
target_link_libraries(${PROJECT_NAME} ${DISKANN_ASYNC_LIB})

install()
# install( TARGETS  ${PROJECT_NAME}_static ${PROJECT_NAME} ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
install( TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)

