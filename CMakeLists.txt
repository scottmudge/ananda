cmake_minimum_required(VERSION 3.1)
project(ananda)

option(STATIC_RUNTIME "This links the target binary against the static C/C++ runtime. Usefull for making portable \
         binaries" ON)

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to Release")
    set(CMAKE_BUILD_TYPE "Release")
endif ()

find_package(OpenCL)
if (NOT OpenCL_FOUND)
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
    include(BuildIcd)
    find_package(OpenCL REQUIRED)
    if (UNIX)
        set(OpenCL_LIBRARIES ${OpenCL_LIBRARIES} pthread dl)
    elseif (WIN32)
        set(OpenCL_LIBRARIES ${OpenCL_LIBRARIES} cfgmgr32.lib)
    endif ()
endif ()

FIND_PATH(HPP_FOUND CL/cl.hpp PATHS ${OpenCL_INCLUDE_DIRS})
if (NOT HPP_FOUND)
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
    include(BuildCLHpp)
    set(OpenCL_INCLUDE_DIRS ${OpenCL_INCLUDE_DIRS} "${CLHPP_ROOT}/include")
endif ()

if (NOT OpenCL_FOUND)
    message(FATAL_ERROR "Could not find OpenCL include/libs. Set OCL_ROOT to your OpenCL SDK. Download AMD APP SDK "
            "http://developer.amd.com/tools-and-sdks/heterogeneous-computing/amd-accelerated-parallel-processing-app-sdk/ for x86/x64 "
            "or pocl http://pocl.sourceforge.net/ for ARM systems")
else ()
    message(STATUS "Selected OpenCL includes from ${OpenCL_INCLUDE_DIRS}")
    message(STATUS "Selected OpenCL lib ${OpenCL_LIBRARIES}")
endif ()

set(CMAKE_CXX_STANDARD "11")
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (STATIC_RUNTIME)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -static -static-libgcc")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static -static-libgcc -static-libstdc++")
    set(CMAKE_LINK_LIBRARY_FLAG "${CMAKE_LINK_LIBRARY_FLAG} -s -static -static-libgcc")
endif(STATIC_RUNTIME)

set(SOURCE_FILES
        src/common.cpp
        src/main.cpp
        src/options.cpp
        src/logger.cpp
        src/global_bandwidth.cpp
        src/compute_sp.cpp
        src/compute_hp.cpp
        src/compute_dp.cpp
        src/compute_integer.cpp
        src/transfer_bandwidth.cpp
        src/kernel_latency.cpp
        src/entry.cpp
        )

add_executable(${PROJECT_NAME} ${SOURCE_FILES})
target_link_libraries(${PROJECT_NAME} ${OpenCL_LIBRARIES})
target_include_directories(${PROJECT_NAME} PRIVATE ${OpenCL_INCLUDE_DIRS} "include" "src/kernels")
target_compile_definitions(${PROJECT_NAME} PRIVATE CL_TARGET_OPENCL_VERSION=120)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR
        CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR
        CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    target_compile_options(${PROJECT_NAME} PRIVATE -fPIC -Wall -Wextra -Wconversion -Wno-deprecated-declarations -Wno-unused-parameter -Wno-ignored-attributes)
elseif (MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /wd4996)
endif ()

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION bin)
install(FILES LICENSE DESTINATION share/${PROJECT_NAME})

