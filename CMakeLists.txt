#
# CMakeLists.txt
#

CMAKE_MINIMUM_REQUIRED(VERSION 3.20)

PROJECT(vuda LANGUAGES CXX C)

INCLUDE(CTest)

INCLUDE(ProcessorCount)

PROCESSORCOUNT(PROCESSOR_COUNT)

OPTION(NO_CUDA "Disable CUDA, fallback all Cuda code to Vulkan" OFF)

IF(PROCESSOR_COUNT EQUAL 0)
    SET(PROCESSOR_COUNT 1)
ENDIF()

SET(PROCESSOR_COUNT
    ${PROCESSOR_COUNT}
    CACHE STRING "" FORCE
)

STRING(TOLOWER ${CMAKE_SYSTEM_NAME} CMAKE_SYSTEM_NAME_LOWER)
SET(CMAKE_SYSTEM_NAME_LOWER
    ${CMAKE_SYSTEM_NAME_LOWER}
    CACHE STRING "" FORCE
)

# TYPE OF BUILD DEFAULT RELEASE
IF(NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE
        Release
        CACHE STRING "" FORCE
    )
ENDIF()

FIND_PACKAGE(CUDA QUIET)

IF(NOT NO_CUDA AND CUDA_FOUND)
    STRING(REPLACE "." ";" CUDA_VERSION_LIST ${CUDA_VERSION})
    LIST(GET CUDA_VERSION_LIST 0 CUDA_MAJOR)
    LIST(GET CUDA_VERSION_LIST 1 CUDA_MINOR)
    SET(CMAKE_CUDA_ARCHITECTURES
        60 61 70 75 80 86
        CACHE STRING "CUDA architectures" FORCE
    )

    ENABLE_LANGUAGE(CUDA)

    SET(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} --compiler-options -fPIC")
    SET(CUDA_LIBRARIES_NAME nvToolsExt;nvrtc;cusparse;curand;cublas;CUDART;cufft;cudnn;cuda;)
    SET(CUDA_ALL_LIBRARIES)

    ADD_LIBRARY(${TARGET_GLOBAL_NAMESPACE}_cuda SHARED IMPORTED GLOBAL)
    ADD_LIBRARY(${TARGET_GLOBAL_NAMESPACE}::cuda ALIAS ${TARGET_GLOBAL_NAMESPACE}_cuda)
    TARGET_INCLUDE_DIRECTORIES(${TARGET_GLOBAL_NAMESPACE}_cuda INTERFACE ${CUDA_INCLUDE_DIRS})

    FOREACH(lib ${CUDA_LIBRARIES_NAME})
        SET(${lib}_LIBNAME ${lib} lib${lib}.so lib${lib}.a lib${lib}.dylib ${lib}64_1.lib)
        FIND_LIBRARY(
            CUDA_${lib}_LIBRARY
            NAMES ${${lib}_LIBNAME}
            HINTS ${${lib}_SEARCH_DIRS}
            PATH_SUFFIXES lib lib64 cuda/lib cuda/lib64 lib/x64
        )

        UNSET(${lib}_SEARCH_DIRS)
        UNSET(${lib}_LIBNAME)

        IF(NOT EXISTS ${CUDA_${lib}_LIBRARY})
            MESSAGE(WARNING "Could NOT found ${lib}")
        ELSE()
            MESSAGE(STATUS "Found ${lib}: ${CUDA_${lib}_LIBRARY}")
            SET(CUDA_ALL_LIBRARIES ${CUDA_ALL_LIBRARIES};${CUDA_${lib}_LIBRARY})
        ENDIF()

        SET_PROPERTY(TARGET ${TARGET_GLOBAL_NAMESPACE}_cuda PROPERTY IMPORTED_LOCATION ${CUDA_${lib}_LIBRARY})
        TARGET_LINK_LIBRARIES(
            ${TARGET_GLOBAL_NAMESPACE}_cuda INTERFACE "-Wl,--no-as-needed ${CUDA_${lib}_LIBRARY} -Wl,-as-needed"
        )
    ENDFOREACH()

    SET(NO_CUDA
        OFF
        CACHE BOOL "" FORCE
    )
    SET(CUDA_VERSION_LIST
        ${CUDA_VERSION_LIST}
        CACHE STRING "" FORCE
    )

    SET(Thread_LIBRARIES pthread)

    SET(LIBS ${CUDA_ALL_LIBRARIES} ${Thread_LIBRARIES})

    SET(INCLUDES ${CUDA_INCLUDE_DIRS})

    ADD_DEFINITIONS(-D__NVCC__)

    MESSAGE(
        STATUS
            "System: '${CMAKE_SYSTEM_NAME_LOWER}' - Cuda: '${CUDA_MAJOR}.${CUDA_MINOR}' - Processor: '${PROCESSOR_COUNT}' - Build: '${CMAKE_BUILD_TYPE}' - Install: '${CMAKE_INSTALL_PREFIX}'"
    )
ELSE()
    SET(CUDA_INCLUDE_DIRS
        ""
        CACHE STRING "" FORCE
    )
    SET(CUDA_TOOLKIT_INCLUDE
        ""
        CACHE STRING "" FORCE
    )
    SET(CUDA_CUDART_LIBRARY
        ""
        CACHE STRING "" FORCE
    )
    SET(NO_CUDA
        ON
        CACHE BOOL "" FORCE
    )

    FIND_PACKAGE(Vulkan)

    IF(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
        ADD_DEFINITIONS(-DVK_USE_PLATFORM_MACOS_MVK)
    ELSE()
        SET(Thread_LIBRARIES pthread)
    ENDIF()

    SET(LIBS ${Vulkan_LIBRARIES} ${Thread_LIBRARIES})

    SET(INCLUDES ${Vulkan_INCLUDE_DIR})

    MESSAGE(
        STATUS
            "System: '${CMAKE_SYSTEM_NAME_LOWER}' - Vulkan - Processor: '${PROCESSOR_COUNT}' - Build: '${CMAKE_BUILD_TYPE}' - Install: '${CMAKE_INSTALL_PREFIX}'"
    )
ENDIF()

SET(CMAKE_OBJECT_PATH_MAX 200)

SET(CMAKE_USE_RELATIVE_PATHS TRUE)

SET(CMAKE_SKIP_BUILD_RPATH FALSE)

SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)

SET(CMAKE_CXX_EXTENSIONS ON)

SET(CMAKE_CXX_STANDARD 17)

SET(CMAKE_CXX_STANDARD_REQUIRED TRUE)

SET(CMAKE_CUDA_STANDARD 14)

SET(CMAKE_CUDA_STANDARD_REQUIRED TRUE)

SET(CMAKE_POSITION_INDEPENDENT_CODE ON)

INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/inc)

INCLUDE_DIRECTORIES(${INCLUDES})

LINK_LIBRARIES(${LIBS})

ADD_SUBDIRECTORY(samples)
