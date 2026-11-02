cmake_minimum_required(VERSION 3.10)
project(openslam_gmapping VERSION 0.2.1 LANGUAGES CXX)

# Plain CMake build (no ROS / catkin required).
#
# Libraries are built as gmapping_<name> (e.g. gmapping_gridfastslam) so the
# names do not clash with other projects, and each one also has an
# openslam_gmapping::<name> alias, the same name the installed package exports.
#
# Use from another CMake project either by
#   add_subdirectory(openslam_gmapping)
# or, after installing,
#   find_package(openslam_gmapping REQUIRED)
# and then
#   target_link_libraries(my_target PRIVATE openslam_gmapping::gridfastslam)

include(GNUInstallDirs)
include(GenerateExportHeader)
include(CMakePackageConfigHelpers)

if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)
  set(_gmapping_top_level ON)
else()
  set(_gmapping_top_level OFF)
endif()

option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" OFF)
option(OPENSLAM_GMAPPING_BUILD_TOOLS
  "Build the command line tools and test programs" ${_gmapping_top_level})
option(OPENSLAM_GMAPPING_INSTALL "Generate install rules" ${_gmapping_top_level})

# Static libraries are usually linked into a shared plugin, so build them as PIC.
if(NOT DEFINED CMAKE_POSITION_INDEPENDENT_CODE)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

if(MSVC)
  add_compile_definitions(_USE_MATH_DEFINES _CRT_SECURE_NO_WARNINGS NOMINMAX)
endif()

set(GMAPPING_EXPORT_HEADER_DIR "${CMAKE_CURRENT_BINARY_DIR}/include")

set(GMAPPING_LIBRARIES)

# gmapping_add_library(<name> <export header path under include/> <sources...>)
function(gmapping_add_library name export_header)
  set(target gmapping_${name})
  add_library(${target} ${ARGN})
  add_library(openslam_gmapping::${name} ALIAS ${target})
  set_target_properties(${target} PROPERTIES EXPORT_NAME ${name})
  target_include_directories(${target} PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${GMAPPING_EXPORT_HEADER_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
  string(TOUPPER ${name} upper)
  generate_export_header(${target}
    BASE_NAME ${name}
    EXPORT_FILE_NAME ${GMAPPING_EXPORT_HEADER_DIR}/${export_header})
  if(NOT BUILD_SHARED_LIBS)
    target_compile_definitions(${target} PUBLIC ${upper}_STATIC_DEFINE)
  endif()
  set(GMAPPING_LIBRARIES ${GMAPPING_LIBRARIES} ${target} PARENT_SCOPE)
endfunction()

function(gmapping_add_tool name)
  add_executable(${name} ${ARGN})
  set(GMAPPING_TOOLS ${GMAPPING_TOOLS} ${name} PARENT_SCOPE)
endfunction()

# utils/
gmapping_add_library(utils gmapping/utils/utils_export.h
  utils/stat.cpp
  utils/movement.cpp)

# sensor/
gmapping_add_library(sensor_base gmapping/sensor/sensor_base/sensor_base_export.h
  sensor/sensor_base/sensor.cpp
  sensor/sensor_base/sensorreading.cpp)

gmapping_add_library(sensor_odometry gmapping/sensor/sensor_odometry/sensor_odometry_export.h
  sensor/sensor_odometry/odometrysensor.cpp
  sensor/sensor_odometry/odometryreading.cpp)
target_link_libraries(gmapping_sensor_odometry PUBLIC gmapping_sensor_base)

gmapping_add_library(sensor_range gmapping/sensor/sensor_range/sensor_range_export.h
  sensor/sensor_range/rangesensor.cpp
  sensor/sensor_range/rangereading.cpp)
target_link_libraries(gmapping_sensor_range PUBLIC gmapping_sensor_base)

# log/
gmapping_add_library(log gmapping/log/log_export.h
  log/configuration.cpp
  log/carmenconfiguration.cpp
  log/sensorlog.cpp
  log/sensorstream.cpp)
target_link_libraries(gmapping_log PUBLIC
  gmapping_sensor_range gmapping_sensor_odometry gmapping_sensor_base)

# configfile/
gmapping_add_library(configfile gmapping/configfile/configfile_export.h
  configfile/configfile.cpp)

# scanmatcher/
gmapping_add_library(scanmatcher gmapping/scanmatcher/scanmatcher_export.h
  scanmatcher/smmap.cpp
  scanmatcher/scanmatcher.cpp
  scanmatcher/scanmatcherprocessor.cpp
  scanmatcher/eig3.cpp)
target_link_libraries(gmapping_scanmatcher PUBLIC
  gmapping_log gmapping_sensor_range gmapping_sensor_odometry gmapping_sensor_base gmapping_utils)

# gridfastslam/
gmapping_add_library(gridfastslam gmapping/gridfastslam/gridfastslam_export.h
  gridfastslam/gridslamprocessor_tree.cpp
  gridfastslam/motionmodel.cpp
  gridfastslam/gridslamprocessor.cpp
  gridfastslam/gfsreader.cpp)
target_link_libraries(gmapping_gridfastslam PUBLIC
  gmapping_scanmatcher gmapping_log gmapping_sensor_range gmapping_sensor_odometry
  gmapping_sensor_base gmapping_utils)

if(OPENSLAM_GMAPPING_BUILD_TOOLS)
  set(GMAPPING_TOOLS)
  gmapping_add_tool(autoptr_test utils/autoptr_test.cpp)
  target_link_libraries(autoptr_test gmapping_utils)

  gmapping_add_tool(log_test log/log_test.cpp)
  gmapping_add_tool(log_plot log/log_plot.cpp)
  gmapping_add_tool(scanstudio2carmen log/scanstudio2carmen.cpp)
  gmapping_add_tool(rdk2carmen log/rdk2carmen.cpp)
  foreach(tool log_test log_plot scanstudio2carmen rdk2carmen)
    target_link_libraries(${tool} gmapping_log)
  endforeach()

  gmapping_add_tool(configfile_test configfile/configfile_test.cpp)
  target_link_libraries(configfile_test gmapping_configfile)

  gmapping_add_tool(scanmatch_test scanmatcher/scanmatch_test.cpp)
  gmapping_add_tool(icptest scanmatcher/icptest.cpp)
  target_link_libraries(scanmatch_test gmapping_scanmatcher)
  target_link_libraries(icptest gmapping_scanmatcher)

  gmapping_add_tool(gfs2log gridfastslam/gfs2log.cpp)
  gmapping_add_tool(gfs2rec gridfastslam/gfs2rec.cpp)
  gmapping_add_tool(gfs2neff gridfastslam/gfs2neff.cpp)
  foreach(tool gfs2log gfs2rec gfs2neff)
    target_link_libraries(${tool} gmapping_gridfastslam)
  endforeach()
endif()

if(OPENSLAM_GMAPPING_INSTALL)
  set(GMAPPING_CMAKE_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/openslam_gmapping)

  install(TARGETS ${GMAPPING_LIBRARIES}
    EXPORT openslam_gmappingTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

  if(OPENSLAM_GMAPPING_BUILD_TOOLS)
    install(TARGETS ${GMAPPING_TOOLS}
      RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
  endif()

  install(DIRECTORY include/gmapping ${GMAPPING_EXPORT_HEADER_DIR}/gmapping
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hxx")

  install(EXPORT openslam_gmappingTargets
    NAMESPACE openslam_gmapping::
    DESTINATION ${GMAPPING_CMAKE_DIR})

  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/openslam_gmappingConfig.cmake
    "include(\"\${CMAKE_CURRENT_LIST_DIR}/openslam_gmappingTargets.cmake\")\n")
  write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/openslam_gmappingConfigVersion.cmake
    COMPATIBILITY SameMajorVersion)
  install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/openslam_gmappingConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/openslam_gmappingConfigVersion.cmake
    DESTINATION ${GMAPPING_CMAKE_DIR})

  install(FILES LICENSE DESTINATION ${CMAKE_INSTALL_DATADIR}/doc/openslam_gmapping)
endif()
