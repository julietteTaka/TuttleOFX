# CMakeLists.txt for Tuttle host library 
# This file needs to be split in several simple one
# The folder tree must be reorganized

# Add custom cmake macros
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

# All libraries will be put in INSTALL_PREFIX/lib 
# RPATH of host points INSTALL_PREFIX/lib
# see: http://www.cmake.org/Wiki/CMake_RPATH_handling
set(CMAKE_MACOSX_RPATH 1)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Tuttle source code
include_directories(src)
file(GLOB_RECURSE TUTTLE_HOST_SRC_FILES src/tuttle/host/*.?pp src/tuttle/common/*.?pp)

# Boost dependencies
include(UseTuttleBoost)
if (TuttleBoost_FOUND) 
  include_directories(${Boost_INCLUDE_DIRS}) 
  
  # OpenFX and Terry rely on boost
  include(UseOfxpp)
  include(UseTerry)
  
  # SequenceParser, as a separate project
  find_package(sequenceParser)
  include_directories(${SEQUENCEPARSER_INCLUDE_DIR})
  
  # Target is a shared library
  add_library(tuttleHost SHARED ${TUTTLE_HOST_SRC_FILES})

  # Extract tuttleHost_VERSION_MAJOR/MINOR/MICRO from version.hpp
  file(STRINGS "src/tuttle/host/version.hpp" _tuttleHost_VERSION_HPP_CONTENTS REGEX "#define TUTTLE_HOST_VERSION_")
  foreach(v MAJOR MINOR MICRO)
    if("${_tuttleHost_VERSION_HPP_CONTENTS}" MATCHES "#define TUTTLE_HOST_VERSION_${v} ([0-9]+)")
      set(tuttleHost_VERSION_${v} "${CMAKE_MATCH_1}")
    else()
      set(tuttleHost_RETRIEVE_VERSION_FAILED 1)
    endif()
  endforeach()
  unset(_tuttleHost_VERSION_HPP_CONTENTS)

  if(tuttleHost_RETRIEVE_VERSION_FAILED)
    message(WARNING "Failed to retrieve tuttleHost version (from version.hpp): "
		   "${tuttleHost_VERSION_MAJOR}.${tuttleHost_VERSION_MINOR}.${tuttleHost_VERSION_MICRO}")
  else()
    message(STATUS "tuttleHost version is "
                 "${tuttleHost_VERSION_MAJOR}.${tuttleHost_VERSION_MINOR}.${tuttleHost_VERSION_MICRO} (retrieved from version.hpp)")

    set_target_properties(
      tuttleHost
    PROPERTIES
      VERSION "${tuttleHost_VERSION_MAJOR}.${tuttleHost_VERSION_MINOR}.${tuttleHost_VERSION_MICRO}"
  )
  endif()

  target_link_libraries(tuttleHost ${Boost_LIBRARIES})
  if(APPLE)
      set_target_properties(tuttleHost 
            PROPERTIES LINK_FLAGS "-framework CoreFoundation -w")
      set_target_properties(tuttleHost 
            PROPERTIES INSTALL_RPATH "@loader_path")
  endif(APPLE)
  if (UNIX AND NOT APPLE)
      target_link_libraries(tuttleHost dl)
      set_target_properties(tuttleHost
            PROPERTIES INSTALL_RPATH "$ORIGIN")
  endif(UNIX AND NOT APPLE)
  
  # TODO: if cmake >= 2.8.11
  # TODO: declare BOOST Atomic optional.
  # There is a basic fallback implementation for boost_atomic:
  # libraries/tuttle/src/tuttle/common/atomic.hpp
  # target_compile_definitions(tuttleHost PUBLIC WITHOUT_BOOST_ATOMIC)

  # Add a target with only object files to avoid recompiling sequence parser
  # multiple times
  add_library(sequenceParser SHARED ${SEQUENCEPARSER_FILES}) 
  set_target_properties(sequenceParser
                        PROPERTIES COMPILE_FLAGS "-fPIC")
  target_link_libraries(sequenceParser ${Boost_LIBRARIES})
  
  # Adds a support library for common and plugins
  # only compile object files, and use them at link time
  add_library(tuttleCommon STATIC ${TUTTLE_COMMON_FILES}) 
  set_target_properties(tuttleCommon
                        PROPERTIES COMPILE_FLAGS "-fPIC")
  add_library(tuttlePluginLib STATIC ${OFXPP_FILES})
  find_package(Threads)
  target_link_libraries(tuttleCommon ${CMAKE_THREAD_LIBS_INIT})

  # Utilities for OFX plugins
  target_link_libraries(tuttlePluginLib sequenceParser) 
  set_target_properties(tuttlePluginLib
                        PROPERTIES COMPILE_FLAGS "-fPIC")


  # Install tuttle host library in INSTALL_PREFIX/lib/
  install(TARGETS tuttleHost DESTINATION lib/)
  install(TARGETS sequenceParser DESTINATION lib/)

  # Install tuttle host headers
  install(
    DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/src/tuttle"
    DESTINATION "include"
    FILES_MATCHING
    PATTERN "*.hpp"
    PATTERN "*.i"
  )

  # Install boost library along with tuttle library
  # Need to copy the files pointed by symlinks instead of the symlinks
  include(TuttleMacros)
  tuttle_install_shared_libs("${Boost_LIBRARIES}")

  # Python bindings with Swig 
  find_package(SWIG)
  if (SWIG_FOUND)
    
    # Load Modules/UseSWIG.cmake
    include(${SWIG_USE_FILE})
  
    # Find python c libs
    find_package(PythonLibs)
    include_directories(${PYTHON_INCLUDE_DIRS})

    # Numpy
    if(WITHOUT_NUMPY)
        # Add swig flag to create binding without Numpy
        set(SWIG_NUMPY_FLAG "-DWITHOUT_NUMPY")
    else()
        # Find numpy headers
        find_package(Numpy)
        if(NUMPY_FOUND)
            include_directories(${NUMPY_INCLUDE_DIR})
        else()
            set(SWIG_NUMPY_FLAG "-DWITHOUT_NUMPY")
        endif()
    endif()

    # Swig to compile c++ -> python
    set(CMAKE_SWIG_FLAGS -c++ -threads -Wall -small -fcompact -O -modern ${SWIG_NUMPY_FLAG}) #-doxygen

    # pyTuttle
    set(TUTTLE_PYTHON_BINDING_FILE src/tuttle/host/tuttle.i)
    # FIXME : do we need a python package as before ?
    set_source_files_properties(${TUTTLE_PYTHON_BINDING_FILE} PROPERTIES CPLUSPLUS ON) 

    # Create the output dir as it's not done automatically by cmake
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/src/tuttle/host)
    swig_add_module(tuttle python ${TUTTLE_PYTHON_BINDING_FILE})
    swig_link_libraries(tuttle tuttleHost)
    swig_link_libraries(tuttle ${Boost_LIBRARIES})
    swig_link_libraries(tuttle ${PYTHON_LIBRARIES})
    # Force the library to look in the project lib folder relatively
    if (APPLE)
      set_target_properties(${SWIG_MODULE_tuttle_REAL_NAME} 
            PROPERTIES INSTALL_RPATH "@loader_path/../../..")
    else(APPLE)
        set_target_properties(${SWIG_MODULE_tuttle_REAL_NAME}
            PROPERTIES INSTALL_RPATH "$ORIGIN/../../..:$ORIGIN")
    endif(APPLE)

    # Install python libs and wrapper in INSTALL_PREFIX/lib/python
    install(TARGETS ${SWIG_MODULE_tuttle_REAL_NAME} 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pyTuttle)
    install(FILES ${CMAKE_BINARY_DIR}/libraries/tuttle/tuttle.py 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pyTuttle)
    file(WRITE "__init__.py" "")
    install(FILES "__init__.py" 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pyTuttle)

    # pySequenceParser
    set(SEQUENCEPARSER_PYTHON_BINDING_FILE ../sequenceParser/src/sequenceParser.i)
    set_source_files_properties(${SEQUENCEPARSER_PYTHON_BINDING_FILE} PROPERTIES CPLUSPLUS ON) 

    # Create the output dir as it's not done automatically by cmake
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/../sequenceParser/src)
    swig_add_module(sequenceParser python ${SEQUENCEPARSER_PYTHON_BINDING_FILE})
    swig_link_libraries(sequenceParser sequenceParser)
    swig_link_libraries(sequenceParser ${Boost_LIBRARIES})
    swig_link_libraries(sequenceParser ${PYTHON_LIBRARIES})
    # Force the library to look in the project lib folder relatively
    if (APPLE)
      set_target_properties(${SWIG_MODULE_sequenceParser_REAL_NAME} 
            PROPERTIES INSTALL_RPATH "@loader_path/../../..")
    else(APPLE)
        set_target_properties(${SWIG_MODULE_sequenceParser_REAL_NAME}
            PROPERTIES INSTALL_RPATH "$ORIGIN/../../..:$ORIGIN")
    endif(APPLE)

    # Install python libs and wrapper in INSTALL_PREFIX/lib/python
    install(TARGETS ${SWIG_MODULE_sequenceParser_REAL_NAME} 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pySequenceParser)
    install(FILES ${CMAKE_BINARY_DIR}/libraries/tuttle/sequenceParser.py 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pySequenceParser)
    file(WRITE "__init__.py" "")
    install(FILES "__init__.py" 
        DESTINATION lib/python${PYTHONLIBS_VERSION_STRING}/site-packages/pySequenceParser)

  else(SWIG_FOUND)
    message("SWIG not found, will not build python bindings")
  endif(SWIG_FOUND)

endif(TuttleBoost_FOUND)
