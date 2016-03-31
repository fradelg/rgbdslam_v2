cmake_minimum_required(VERSION 2.8)
project(rgbdslam)

set(USE_SIFT_GPU   OFF CACHE BOOLEAN "Compile with SIFT GPU support")
set(SIFT_GPU_MODE  1   CACHE INTEGER "Use CUDA (1) or GLSL (2)")
set(USE_GICP_BIN   OFF CACHE BOOLEAN "Compile with GICP library")
set(USE_GICP_CODE	 OFF CACHE BOOLEAN "Use the embedded GICP source")
set(USE_PCL_ICP		 OFF CACHE BOOLEAN "Use the ICP algorithm from PCL")
set(USE_GL2PS      OFF CACHE BOOLEAN "Prints 3D Window to PostScript - requires libgl2ps-dev")

if ("${ROS_PARALLEL_JOBS}" EQUAL "")
  set(ROS_PARALLEL_JOBS 1)
endif ("${ROS_PARALLEL_JOBS}" EQUAL "")

SET(CMAKE_CXX_FLAGS "-O3 -fPIC -std=c++0x")

IF (${USE_GL2PS})
  add_definitions(-DGL2PS)
ENDIF (${USE_GL2PS})

#is used for compiling the libraries
set(USE_GICP 0)
IF (${USE_GICP_BIN} OR ${USE_GICP_CODE})
	set(USE_GICP 1)
ENDIF (${USE_GICP_BIN} OR ${USE_GICP_CODE})

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules")

# Find catkin macros and libraries
find_package(catkin
  REQUIRED COMPONENTS
    pcl_conversions pcl_ros roscpp image_transport
    cv_bridge sensor_msgs geometry_msgs visualization_msgs
    std_msgs tf message_generation message_filters rosbag rosconsole
)

# System dependencies are found with CMake's conventions
find_package(OpenCV REQUIRED)
find_package(PCL REQUIRED COMPONENTS common io)
find_package(CSparse REQUIRED)
find_package(Qt4 REQUIRED)
find_package(G2O REQUIRED)
find_package(OpenMP)
find_package(Eigen3)
find_package(Octomap)

add_service_files(
  DIRECTORY srv
  FILES rgbdslam_ros_ui.srv rgbdslam_ros_ui_b.srv rgbdslam_ros_ui_f.srv rgbdslam_ros_ui_s.srv
)

generate_messages()

catkin_package(
#  INCLUDE_DIRS include
#  LIBRARIES my_pcl_tutorial
  CATKIN_DEPENDS pcl_conversions pcl_ros roscpp sensor_msgs
  DEPENDS message_runtime std_msgs
)

link_directories(
  ${catkin_LIBRARY_DIRS}
  ${PCL_LIBRARY_DIRS}
  ${OCTOMAP_LIBRARY_DIRS}
  ${G2O_LIBRARY_DIRS}
)

add_definitions(${PCL_DEFINITIONS})
add_definitions(${EIGEN_DEFINITIONS})

# Qt ########################
# http://qtnode.net/wiki/Qt4_with_cmake
SET(QT_USE_QTXML TRUE)
SET(QT_USE_QTOPENGL TRUE)
add_definitions(-DQT_NO_KEYWORDS)
include(${QT_USE_FILE})

# OpenMP ########################
if (OPENMP_FOUND)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

# This is necessary as all ui files etc will get dumped in the bottom of then binary directory.
include_directories(
  ${CMAKE_CURRENT_BINARY_DIR}
  ${QT_QTOPENGL_INCLUDE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/external/gicp/ann_1.1.2/include/ANN
  ${OCTOMAP_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS}
  ${PCL_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${EIGEN_INCLUDE_DIRS}
  ${G2O_INCLUDE_DIRS}
  ${CSPARSE_INCLUDE_DIR}
)

# Only possible when not subscribing to point cloud topic.
# Possibly dangerous anyway, but halves storage space per point cloud
#add_definitions(-DRGB_IS_4TH_DIM)
#add_definitions(-DDO_FEATURE_OPTIMIZATION)
#add_definitions(-DHEMACLOUDS)
add_definitions(-DROSCONSOLE_MIN_SEVERITY=ROSCONSOLE_SEVERITY_INFO)

#############################
# SiftGPU ###################
#############################
if (${USE_SIFT_GPU})
  add_subdirectory(external)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/external/SiftGPU/src)
  link_directories(${CMAKE_CURRENT_SOURCE_DIR}/external/SiftGPU/bin)
  add_definitions(-DUSE_SIFT_GPU)
  add_definitions(-DSIFT_GPU_MODE=${SIFT_GPU_MODE})
endif (${USE_SIFT_GPU})

#############################
# GICP ######################
#############################
IF (${USE_GICP})
	SET(gicp_build_path ${CMAKE_SOURCE_DIR}/external/gicp/ann_1.1.2)
  FILE(MAKE_DIRECTORY ${gicp_build_path}/lib/)
  MESSAGE("\n------------------------------------------------------------------\n")
	MESSAGE("Compiling GICP...")
	EXECUTE_PROCESS(
	    COMMAND make ${ROS_PARALLEL_JOBS} linux-g++
	    WORKING_DIRECTORY ${gicp_build_path}
	    RESULT_VARIABLE gicp_return
    )
  MESSAGE("\n------------------------------------------------------------------\n")

	#on error
	IF (NOT ${gicp_return} EQUAL 0)
		MESSAGE(FATAL_ERROR "GICP cannot be compiled. Returned: ${gicp_return}")
	ENDIF (NOT ${gicp_return} EQUAL 0)

	FILE(COPY ${gicp_build_path}/lib/libANN.a DESTINATION ${CMAKE_SOURCE_DIR}/lib/)
ENDIF (${USE_GICP})

IF (${USE_GICP_BIN})
  add_definitions(-DUSE_ICP_BIN)
ENDIF (${USE_GICP_BIN})

IF (${USE_GICP_CODE})
  add_definitions(-DUSE_ICP_CODE)
ENDIF (${USE_GICP_CODE})
IF (${USE_PCL_ICP})
  add_definitions(-DUSE_PCL_ICP)
ENDIF (${USE_PCL_ICP})

link_directories(
  ${OpenCV_LIBRARY_DIRS}
)

##############################################################################
# Some QT Magic (MOC Creation)
##############################################################################

file(GLOB QT_FORMS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ui/*.ui)
file(GLOB QT_RESOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} resources/*.qrc)
file(GLOB_RECURSE QT_MOC RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} FOLLOW_SYMLINKS include/local/*.hpp src/qtros.h src/openni_listener.h src/qt_gui.h src/graph_manager.h src/glviewer.h src/ros_service_ui.h)

QT4_ADD_RESOURCES(QT_RESOURCES_CPP ${QT_RESOURCES})
QT4_WRAP_UI(QT_FORMS_HPP ${QT_FORMS})
QT4_WRAP_CPP(QT_MOC_HPP ${QT_MOC})

##############################################################################
# Sources to Compile
##############################################################################
SET(SOURCES
  src/main.cpp
  src/gicp-fallback.cpp
  src/qtros.cpp
  src/openni_listener.cpp
  src/qt_gui.cpp
  src/node.cpp
  src/glviewer.cpp
  src/parameter_server.cpp
  src/ros_service_ui.cpp
  src/misc.cpp
  src/landmark.cpp
  src/loop_closing.cpp
  src/ColorOctomapServer.cpp
  src/scoped_timer.cpp
  src/icp.cpp
  src/matching_result.cpp
#GraphManager Files
  src/graph_manager.cpp
  src/graph_mgr_io.cpp
  src/graph_manager2.cpp
  src/graph_mgr_odom.cpp
  src/transformation_estimation_euclidean.cpp
  src/transformation_estimation.cpp
  src/graph_manager2.cpp
  src/header.cpp
#Computation of meaningful covariances
  src/covariance_estimation.cpp
#Detector Threshold Adapter
  src/feature_adjuster.cpp
  src/aorb.cpp
  src/features.cpp
)

IF (${USE_SIFT_GPU})
  SET(SOURCES ${SOURCES} src/sift_gpu_wrapper.cpp)
ENDIF (${USE_SIFT_GPU})

file(GLOB_RECURSE QT_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} FOLLOW_SYMLINKS)

add_executable(rgbdslam
  ${QT_SOURCES}
  ${QT_RESOURCES_CPP}
  ${QT_FORMS_HPP}
  ${QT_MOC_HPP}
  ${SOURCES}
)

add_dependencies(rgbdslam rgbdslam_gencpp)

SET(LIBS_LINK GL GLU cholmod)
IF (${USE_SIFT_GPU})
 	SET(LIBS_LINK ${LIBS_LINK} siftgpu)
ENDIF (${USE_SIFT_GPU})
IF (${USE_GL2PS})
  SET(LIBS_LINK ${LIBS_LINK} gl2ps)
ENDIF (${USE_GL2PS})
IF (${USE_GICP})
 	SET(LIBS_LINK ${LIBS_LINK} gicp ANN gsl gslcblas)
ENDIF (${USE_GICP})

message(STATUS "${G2O_LIBRARIES}")

target_link_libraries(rgbdslam
  ${LIBS_LINK}
  ${catkin_LIBRARIES}
  ${OCTOMAP_LIBRARIES}
  ${CSPARSE_LIBRARY}
  ${G2O_LIBRARIES}
  ${QT_LIBRARIES}
  ${QT_QTOPENGL_LIBRARY}
  ${GLUT_LIBRARY}
  ${OPENGL_LIBRARY}
  ${OpenCV_LIBS}
)

IF (${USE_GICP})
  set(ROS_COMPILE_FLAGS ${ROS_COMPILE_FLAGS} -fpermissive)
  add_library(gicp
    external/gicp/bfgs_funcs.cpp
    external/gicp/gicp.cpp
    external/gicp/optimize.cpp
    external/gicp/scan.cpp
    external/gicp/transform.cpp
  )
ENDIF (${USE_GICP})

# Mark executables and/or libraries for installation
install(TARGETS rgbdslam
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# install launch files in their own subdirectory
install(DIRECTORY
  launch/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
)

###############################################################################
## Bag Viewer
###############################################################################
#
#file(GLOB QT_FORMS2 RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ui/*.ui)
#file(GLOB QT_RESOURCES2 RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} resources/*.qrc)
#file(GLOB_RECURSE QT_MOC2 RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} FOLLOW_SYMLINKS include/local/*.hpp src/qtros.h src/bagloader.h src/qt_gui.h src/glviewer.h)
#
#QT4_ADD_RESOURCES(QT_RESOURCES_CPP2 ${QT_RESOURCES2})
#QT4_WRAP_UI(QT_FORMS_HPP2 ${QT_FORMS2})
#QT4_WRAP_CPP(QT_MOC_HPP2 ${QT_MOC2})
#
#
#SET(ADDITIONAL_SOURCES2 src/result-bag-viewer.cpp src/bagloader.cpp src/qtros.cpp  src/qt_gui.cpp src/glviewer.cpp src/parameter_server.cpp src/misc.cpp src/ColorOctomapServer.cpp src/scoped_timer.cpp)
#add_executable(bagviewer ${QT_SOURCES2} ${QT_RESOURCES_CPP2} ${QT_FORMS_HPP2} ${QT_MOC_HPP2} ${ADDITIONAL_SOURCES2})
#target_link_libraries(bagviewer ${LIBS_LINK})
#
