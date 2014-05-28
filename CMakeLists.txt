CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
# taken from http://geant4.cern.ch/support/source/geant4/CMakeLists.txt
IF(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(STATUS "Stub requires an out-of-source build.")
    message(STATUS "Please remove these files from ${CMAKE_BINARY_DIR} first:")
    message(STATUS "CMakeCache.txt")
    message(STATUS "CMakeFiles")
    message(STATUS "Once these files are removed, create a separate directory")
    message(STATUS "and run CMake from there")
    message(FATAL_ERROR "in-source build detected")
ENDIF()

# This project name is stub.
PROJECT( stub )

# quiets fortify_source warnings when not compiling with optimizations
# in linux distros where compilers were compiled with fortify_source enabled by
# default (e.g. Arch linux).
MESSAGE("${CMAKE_BUILD_TYPE}")
STRING(TOLOWER "${CMAKE_BUILD_TYPE}" BUILD_TYPE)
IF(NOT ${BUILD_TYPE} STREQUAL "release")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0")
ENDIF()
# no overflow warnings because of silly coin-ness
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-overflow")

# Direct any out-of-source builds to this directory
SET( STUB_SOURCE_DIR ${CMAKE_SOURCE_DIR} )

# Direct any binary installation paths to this directory
SET( STUB_BINARY_DIR ${CMAKE_BINARY_DIR} )

# This makes all the libraries build as SHARED
SET(BUILD_SHARED_LIBS true)

# Setup build locations.
IF(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${STUB_BINARY_DIR}/bin)
endif()
IF(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${STUB_BINARY_DIR}/lib)
endif()
IF(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${STUB_BINARY_DIR}/lib)
ENDIF()

SET(STUB_EXECUTABLE_DIR ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

# use, i.e. don't skip the full RPATH for the build tree
SET(CMAKE_SKIP_BUILD_RPATH  FALSE)

# when building, don't use the install RPATH already
# (but later on when installing)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 

SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

# add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# the RPATH to be used when installing, but only if it's not a system directory
LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
IF("${isSystemDir}" STREQUAL "-1")
   SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
ENDIF("${isSystemDir}" STREQUAL "-1")

# Tell CMake where the modules are
SET( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_DIR}/share/cmake-2.8/Modules")
SET( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${STUB_SOURCE_DIR}/cmake )
MESSAGE("CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}")

# Include macros
INCLUDE(UseCyclus)

# Find cyclus
FIND_PACKAGE( Cyclus REQUIRED )
SET(STUB_INCLUDE_DIR ${STUB_INCLUDE_DIR} ${CYCLUS_CORE_INCLUDE_DIR})
SET(LIBS ${LIBS} ${CYCLUS_CORE_LIBRARIES})

# Find LibXML++ and dependencies
FIND_PACKAGE( LibXML++ REQUIRED )
SET(STUB_INCLUDE_DIR ${STUB_INCLUDE_DIR} ${LibXML++_INCLUDE_DIR} ${Glibmm_INCLUDE_DIRS} ${LibXML++Config_INCLUDE_DIR})
SET(LIBS ${LIBS} ${LibXML++_LIBRARIES})
MESSAGE("--LIBS: ${LIBS}")

MESSAGE("--LD_LIBRARY_PATH: $ENV{LD_LIBRARY_PATH}")

# Include the boost header files, system, and filesystem libraries
SET(Boost_USE_STATIC_LIBS       OFF)
SET(Boost_USE_STATIC_RUNTIME    OFF)
FIND_PACKAGE( Boost COMPONENTS filesystem system REQUIRED)
SET(STUB_INCLUDE_DIR ${STUB_INCLUDE_DIR} ${Boost_INCLUDE_DIR})
SET(LIBS ${LIBS} ${Boost_FILESYSTEM_LIBRARY})
SET(LIBS ${LIBS} ${Boost_SYSTEM_LIBRARY})

# find lapack and link to it
FIND_PACKAGE( LAPACK REQUIRED )
set(LIBS ${LIBS} ${LAPACK_LIBRARIES})
MESSAGE("\tFound LAPACK Libraries: ${LAPACK_LIBRARIES}")

# Find HDF5
FIND_PACKAGE( HDF5 REQUIRED)
ADD_DEFINITIONS(${HDF5_DEFINITIONS})
set(LIBS ${LIBS} ${HDF5_LIBRARIES} )

# find coin and link to it
FIND_PACKAGE( COIN REQUIRED )
set(LIBS ${LIBS} ${COIN_LIBRARIES})

# include the agent directories
SET(STUB_INCLUDE_DIR ${STUB_INCLUDE_DIR} tests 
    ${CYCLUS_CORE_INCLUDE_DIR}/..)

# include all the directories we just found
INCLUDE_DIRECTORIES(${STUB_INCLUDE_DIR} 
                    ${COIN_INCLUDE_DIRS})


# ------------------------- Add the Agents -----------------------------------
ADD_SUBDIRECTORY(src)

# ------------------------- Google Test -----------------------------------

# Be sure to clear these each time
SET( GENERATED_TEST_LIST "" CACHE INTERNAL "" )
SET( GENERATED_FILTER_LIST "" CACHE INTERNAL "" )
SET( GENERATED_TEST_SOURCE_LIST "" CACHE INTERNAL "" )

# The gtest include file directories.
SET(GTEST_INCLUDE_DIRS "${STUB_INCLUDE_DIR}")

# Alerts CMake to any tests we need to worry about
INCLUDE_DIRECTORIES(${GTEST_INCLUDE_DIRS})

# Create an executable for all the gtests
# Options for testing
OPTION( USE_TESTING "Build testing" ON )
IF( USE_TESTING )
  enable_testing()
  INCLUDE(CTest)
ENDIF()

FIND_PACKAGE(Threads)
IF(CMAKE_USE_PTHREADS_INIT)  # The pthreads library is available.
  SET(cxx_base_flags "${cxx_base_flags} -DGTEST_HAS_PTHREAD=1")
ENDIF()

# Build stub_unit_tests
ADD_EXECUTABLE( stub_unit_tests
  src/stub_unit_test_driver.cc 
  ${TestSource}
)

TARGET_LINK_LIBRARIES( stub_unit_tests
  dl ${LIBS} ${CYCLUS_TEST_LIBRARIES})

INSTALL(TARGETS stub_unit_tests
  RUNTIME DESTINATION bin
  COMPONENT testing
  )

# read tests after building the driver, and add them to ctest
set( tgt "stub_unit_tests")
set( script "${STUB_SOURCE_DIR}/cmake/generate_test_macros.py")
set( exec "--executable=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${tgt}")
set( reg_dir "--reg_dir=${STUB_SOURCE_DIR}/../${test_dir}")
set( out "--output=${CMAKE_BINARY_DIR}/CTestTestfile.cmake")
add_custom_command(TARGET ${tgt}
  POST_BUILD
  COMMAND python ${script} ${exec} ${reg_dir} ${out}
  COMMENT "adding tests from ${tgt}"
  DEPENDS
  VERBATIM
  )

# for post test commands
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/CTestCustom.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/CTestCustom.cmake"
    IMMEDIATE @ONLY)


SET(ROOT_DIR ${CYCLUS_ROOT_DIR})  
SET(PROJ_DIR ${PROJECT_BINARY_DIR}) 
SET(CORE_SHARE_DIR ${CYCLUS_CORE_SHARE_DIR}) 


# uninstall target
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)