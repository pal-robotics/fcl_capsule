cmake_minimum_required(VERSION 2.8.3)
project(fcl_capsule)

#if (DEFINED CMAKE_CXX_FLAGS)
#   message("cmake cxx flags found")
#   string(REPLACE "-Werror=uninitialized" "" CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
#else ()
#   message("CMAKE FLAGS ARE NOT FOUND!")
#endif()

string(REPLACE "-Werror=uninitialized" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")

find_package(catkin REQUIRED)
find_package(octomap REQUIRED)

###################################
## catkin specific configuration ##
###################################
## The catkin_package macro generates cmake config files for your package
## Declare things to be passed to dependent projects
## INCLUDE_DIRS: uncomment this if you package contains header files
## LIBRARIES: libraries you create in this project that dependent projects also need
## CATKIN_DEPENDS: catkin_packages dependent projects also need
## DEPENDS: system dependencies of this project that dependent projects also need

# set the default build type
if (NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE Release)
endif()

# This shouldn't be necessary, but there has been trouble
# with MSVC being set off, but MSVCXX ON.
if(MSVC OR MSVC90 OR MSVC10)
    set(MSVC ON)
endif (MSVC OR MSVC90 OR MSVC10)

set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
include(CompilerSettings)
include(FCLVersion)

if(MSVC OR IS_ICPC)
  option(FCL_STATIC_LIBRARY "Whether the FCL library should be static rather than shared" ON)
else()
  option(FCL_STATIC_LIBRARY "Whether the FCL library should be static rather than shared" OFF)
endif()

# Whether to enable SSE
option(FCL_USE_SSE "Whether FCL should SSE instructions" OFF)
set(FCL_HAVE_SSE 0)
if(FCL_USE_SSE)
  if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    set(FCL_HAVE_SSE 0) #always disable, for now
    add_definitions(-march=native)
  endif()
  # TODO: do something similar for other compilers
endif()

# Find Octomap (optional)
find_package(PkgConfig QUIET)

# Find FLANN (optional)
find_package(flann QUIET)
if (FLANN_FOUND)
  set(FCL_HAVE_FLANN 1)
  include_directories(${FLANN_INCLUDE_DIRS})
  message(STATUS "FCL uses Flann")
else()
  message(STATUS "FCL does not use Flann")
endif()


# find_package(tinyxml QUIET)
# if (TINYXML_FOUND)
#   set(FCL_HAVE_TINYXML 1)
#   include_directories(${TINYXML_INCLUDE_DIRS})
#   link_directories(${TINYXML_LIBRARY_DIRS})
#   message(STATUS "FCL uses tinyxml")
# else()
#   message(STATUS "FCL does not use tinyxml")
# endif()


find_package(Boost COMPONENTS thread date_time filesystem system unit_test_framework REQUIRED)
include_directories(${Boost_INCLUDE_DIR})

if(MSVC)
    add_definitions(-DBOOST_ALL_NO_LIB)
endif()
add_definitions(-DBOOST_TEST_DYN_LINK)

# FCL's own include dir should be at the front of the include path
include_directories(BEFORE "include")

if(PKG_CONFIG_FOUND)
  pkg_check_modules(CCD ccd)
  # check to see if the pkg is installed under the libccd name
  if(NOT CCD_FOUND)
    pkg_check_modules(CCD libccd)
  endif()
endif()

if(NOT CCD_FOUND)
  message(FATAL_ERROR "Libccd is required by FCL")
endif()

set(libs "")
foreach(libr "${CCD_LIBRARIES}")
  set(lib "${libr}-NOTFOUND")
  find_library(lib ${libr}
    PATHS "${CCD_LIBRARY_DIRS}"
    NO_DEFAULT_PATH NO_CMAKE_FIND_ROOT_PATH)
  if(lib)
    list(APPEND libs ${lib})
  else()
    find_library(lib ${libr})
    if(lib)
      list(APPEND libs ${lib})
    endif()
  endif()
endforeach()

set(CCD_LIBRARIES ${libs})

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES fcl_capsule
  DEPENDS CCD OCTOMAP
)

include_directories(${CCD_INCLUDE_DIRS})
include_directories(${OCTOMAP_INCLUDE_DIRS})

add_subdirectory(include/fcl_capsule)
add_subdirectory(src)

set(pkg_conf_file "${CMAKE_CURRENT_SOURCE_DIR}/fcl_capsule.pc")
	configure_file("${pkg_conf_file}.in" "${pkg_conf_file}" @ONLY)

install(DIRECTORY include/ DESTINATION include
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hxx"
  PATTERN ".DS_Store" EXCLUDE
)

install(FILES "${pkg_conf_file}" DESTINATION lib/pkgconfig/ COMPONENT pkgconfig)


#enable_testing()
#add_subdirectory(test)
