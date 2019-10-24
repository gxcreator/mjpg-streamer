cmake_minimum_required(VERSION 2.8.3)
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

project("mjpg-streamer" C)
#-march=armv7-a -mcpu=cortex-a9 -mfpu=neon -mfloat-abi=softfp -ftree-vectorize -mvectorize-with-neon-quad -ftree-vectorizer-verbose=2
set (CMAKE_C_FLAGS "-fno-omit-frame-pointer -O3 -mtune=cortex-a7 -mcpu=cortex-a7 -mfloat-abi=hard -mfpu=neon-vfpv4 -funsafe-loop-optimizations -funsafe-math-optimizations -ffast-math -fopt-info-vec-optimized -mvectorize-with-neon-quad ")

find_package(Threads)

# If the user doesn't manually specify a build type, use 'Release'
message("CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")
if("${CMAKE_BUILD_TYPE}" STREQUAL "")
  SET(CMAKE_BUILD_TYPE "Release")
endif()

SET(COMPILE_DEFINITIONS -Werror -Wall)

include(CheckLibraryExists) 
include(CheckIncludeFiles)
include(FeatureSummary)

include(mjpg_streamer_utils)

#
# Get the current git hash
#
execute_process(
  COMMAND git rev-parse HEAD
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  RESULT_VARIABLE GIT_RESULT
  OUTPUT_VARIABLE GIT_HASH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

if(GIT_RESULT EQUAL 0)
  add_definitions("-DGIT_HASH=\"${GIT_HASH}\"")
endif()

#
# Options
#
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DDEBUG")

add_feature_option(WXP_COMPAT "Enable compatibility with WebcamXP" OFF)

if (WXP_COMPAT)
    add_definitions(-DWXP_COMPAT)
endif (WXP_COMPAT)

set (MJPG_STREAMER_PLUGIN_INSTALL_PATH "lib/mjpg-streamer")

#
# Global dependencies
#


set(ENABLE_SHARED FALSE)
set(WITH_TURBOJPEG FALSE)
set(WITH_12BIT FALSE)
#find_library(JPEG_LIB jpeg)
add_subdirectory(libjpeg-turbo)
#link_directories(libjpeg-turbo)

# set_property(TARGET libjpet-turbo PROPERTY IMPORTED_LOCATION libjpeg-turbo/libjpeg.a)

#find_library(JPEG_LIB NAMES "libjpeg.a" PATHS ${PROJECT_BIN_DIR}/libjpeg-turbo NO_DEFAULT_PATH NO_SYSTEM_ENVIRONMENT_PATH NO_CMAKE_ENVIROMENT_PATH NO_CMAKE_SYSTEM_PATH ONLY_CMAKE_FIND_ROOT_PATH)
set(JPEG_LIB ${CMAKE_BINARY_DIR}/libjpeg-turbo/libjpeg.a)
message(STATUS ${JPEG_LIB})#
# Input plugins
#

add_subdirectory(plugins/input_file)
add_subdirectory(plugins/input_http)
add_subdirectory(plugins/input_opencv)
add_subdirectory(plugins/input_raspicam)
add_subdirectory(plugins/input_ptp2)
add_subdirectory(plugins/input_uvc)

#
# Output plugins
#

add_subdirectory(plugins/output_file)
add_subdirectory(plugins/output_http)
add_subdirectory(plugins/output_rtsp)
add_subdirectory(plugins/output_udp)
add_subdirectory(plugins/output_viewer)
add_subdirectory(plugins/output_zmqserver)

#
# mjpg_streamer executable
#

# This adds the plugin installation directory to the default DT_RUNPATH, so
# that the user shouldn't need to set LD_LIBRARY_PATH if using 'make install'
# ... however, DT_RUNPATH allows overriding via LD_LIBRARY_PATH if you really
#     need to do it

set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--enable-new-dtags")
set (CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set (CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_PREFIX}/${MJPG_STREAMER_PLUGIN_INSTALL_PATH})
set (CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)


add_executable(mjpg_streamer mjpg_streamer.c
                             utils.c)

target_link_libraries(mjpg_streamer pthread dl)
install(TARGETS mjpg_streamer DESTINATION bin)

#
# www directory
#

install(DIRECTORY www DESTINATION share/mjpg-streamer)


#
# Show enabled/disabled features
#

feature_summary(WHAT ALL)

#
# Final warning
#

if("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
  message(WARNING "The source directory is the same as binary directory. \"make clean\" may damage the source tree")
endif()
