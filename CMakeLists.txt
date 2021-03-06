cmake_minimum_required(VERSION 2.8.0 FATAL_ERROR)

#-----------------------------------------------------------------------
# CPM configuration
#-----------------------------------------------------------------------
set(CPM_MODULE_NAME "gl_batch_context")
set(CPM_LIB_TARGET_NAME ${CPM_MODULE_NAME})

if ((DEFINED CPM_DIR) AND (DEFINED CPM_UNIQUE_ID) AND (DEFINED CPM_TARGET_NAME))
  set(CPM_LIB_TARGET_NAME ${CPM_TARGET_NAME})
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CPM_DIR})
  include(CPM)
else()
  set (CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm-packages" CACHE TYPE STRING)
  find_package(Git)
  if(NOT GIT_FOUND)
    message(FATAL_ERROR "CPM requires Git.")
  endif()
  if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
    message(STATUS "Cloning repo (https://github.com/CIBC-Internal/cpm)")
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" clone https://github.com/CIBC-Internal/cpm ${CPM_DIR}
      RESULT_VARIABLE error_code
      OUTPUT_QUIET ERROR_QUIET)
    if(error_code)
      message(FATAL_ERROR "CPM failed to get the hash for HEAD")
    endif()
  endif()
  include(${CPM_DIR}/CPM.cmake)
endif()

#-----------------------------------------------------------------------
# CPM Modules
#-----------------------------------------------------------------------

if (WIN32)
  # We need GLEW, and we might as well get the platform headers as well.
  CPM_AddModule("gl_platform"
    GIT_REPOSITORY "https://github.com/CIBC-Internal/cpm-gl-platform"
    GIT_TAG "origin/master"
    USE_EXISTING_VER TRUE)
endif()

# This call will ensure all include directories and definitions are present
# in the target. These correspond to the modules that we added above.
CPM_InitModule(${CPM_MODULE_NAME})

#-----------------------------------------------------------------------
# Module specific CMake options
#-----------------------------------------------------------------------
# Find OpenGL
find_package(OpenGL REQUIRED)

# Compiler options
if (UNIX)
  if (APPLE)

    # Find necessary apple libraries (cocoa).
    FIND_LIBRARY(COCOA_LIBRARY Cocoa)
    set(OS_SPECIFIC_LIBS ${OS_SPECIFIC_LIBS} ${COCOA_LIBRARY})

    # Apple specific sources
    set (SpecificSources
      gl-batch-context/src/CGLContext.cpp
      gl-batch-context/src/CGLContext.hpp
      gl-batch-context/src/NSContext.mm
      gl-batch-context/src/NSContext.h
      )
    add_definitions(-DGL_CONTEXT_USING_OSX)
  else ()
    # Linux specific sources
    option(USE_OS_MESA "Use an OS mesa context." OFF)
    if (USE_OS_MESA)
      message(STATUS "Using OSMesa software rendering.")
      add_definitions(-DUSE_OS_MESA)
      set(MESA_LIBRARIES "OSMesa32")
      set(OS_SPECIFIC_LIBS ${OS_SPECIFIC_LIBS} ${MESA_LIBRARIES})
      set(OPENGL_LIBRARIES) # Remove OpenGL libraries from OSMesa build.
      set (SpecificSources
        gl-batch-context/src/OSMesaContext.cpp
        gl-batch-context/src/OSMesaContext.hpp
        )
    else()
      set (SpecificSources
        gl-batch-context/src/GLXContext.cpp
        gl-batch-context/src/GLXContext.hpp
        )
    endif()
    add_definitions(-DGL_CONTEXT_USING_LINUX)
  endif ()
else()
  if (WIN32)

    # Windows specific sources
    set (SpecificSources
      gl-batch-context/src/WGLContext.cpp
      gl-batch-context/src/WGLContext.hpp
      )

    add_definitions(-DGL_CONTEXT_USING_WIN)
  endif(WIN32)
endif ()


#-----------------------------------------------------------------------
# Source
#-----------------------------------------------------------------------

# Globbing has some downsides, but the advantages outweigh the
# disadvantages.
file (GLOB Sources
  "gl-batch-context/*.cpp"
  "gl-batch-context/*.h"
  )

#-----------------------------------------------------------------------
# Library setup
#-----------------------------------------------------------------------

# Build the library.
add_library(${CPM_LIB_TARGET_NAME} ${Sources} ${SpecificSources})
target_link_libraries(${CPM_LIB_TARGET_NAME} 
  ${OS_SPECIFIC_LIBS}
  ${OPENGL_LIBRARIES}
  ${CPM_LIBRARIES})

