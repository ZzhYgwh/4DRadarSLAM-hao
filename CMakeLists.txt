# SPDX-License-Identifier: BSD-2-Clause
cmake_minimum_required(VERSION 2.8.3)
project(radar_graph_slam)

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Compiler flags
add_definitions(-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")

# 启用 AddressSanitizer
# if (CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
#     set(SANITIZER_FLAGS "-fsanitize=address")
#     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${SANITIZER_FLAGS}")
#     set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${SANITIZER_FLAGS}")
# endif()

# Build type
set(CMAKE_BUILD_TYPE "RELEASE")

# Find required packages
find_package(catkin REQUIRED COMPONENTS
  roscpp
  rospy
  pcl_ros
  geodesy
  nmea_msgs
  sensor_msgs
  geometry_msgs
  message_generation
  interactive_markers
  ndt_omp
  fast_gicp
  image_transport
  cv_bridge
  barometer_bmp388
)

catkin_python_setup()

find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
message(STATUS "OpenCV version: ${OpenCV_VERSION}")

find_package(PCL REQUIRED)
include_directories(${PCL_INCLUDE_DIRS})
link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

list(APPEND CMAKE_MODULE_PATH /home/hao/Downloads/g2o-20200410_git/cmake_modules)
find_package(G2O REQUIRED)
include_directories(SYSTEM ${G2O_INCLUDE_DIR} ${G2O_INCLUDE_DIRS})
link_directories(${G2O_LIBRARY_DIRS})

find_package(OpenMP)
if (OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

find_package(GTSAM REQUIRED)
include_directories(${GTSAM_INCLUDE_DIRS})
link_directories(${GTSAM_LIBRARY_DIRS})

find_library(VGICP_CUDA_FOUND NAMES fast_vgicp_cuda)
message(STATUS "VGICP_CUDA_FOUND: ${VGICP_CUDA_FOUND}")
if(VGICP_CUDA_FOUND)
  add_definitions(-DUSE_VGICP_CUDA)
endif()

########################
## message generation ##
########################
add_message_files(FILES
  FloorCoeffs.msg
  ScanMatchingStatus.msg
)

add_service_files(FILES
  SaveMap.srv
  DumpGraph.srv
)

generate_messages(
  DEPENDENCIES
  std_msgs
  geometry_msgs
)

###################################
## catkin specific configuration ##
###################################
catkin_package(
  INCLUDE_DIRS include
  LIBRARIES radar_graph_slam_nodelet
  CATKIN_DEPENDS pcl_ros roscpp sensor_msgs
  DEPENDS system_lib GTSAM OpenCV barometer_bmp388
)

###########
## Build ##
###########
file(GLOB G2O_SRC "src/g2o/*.cpp")
file(GLOB RADAR_GRAPH_SLAM_SRC "src/radar_graph_slam/*.cpp")
file(GLOB OTHER_SRC "src/radar_ego_velocity_estimator.cpp")
add_library(4dradarslam SHARED
  ${G2O_SRC}
  ${RADAR_GRAPH_SLAM_SRC}
  ${OTHER_SRC}
)
target_link_libraries(4dradarslam 
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${Boost_LIBRARIES}
  ${G2O_TYPES_DATA}
  ${G2O_CORE_LIBRARY}
  ${G2O_STUFF_LIBRARY}
  ${G2O_SOLVER_PCG}
  ${G2O_SOLVER_CSPARSE}   # be aware of that CSPARSE is released under LGPL
  ${G2O_SOLVER_CHOLMOD}   # be aware of that cholmod is released under GPL
  ${G2O_TYPES_SLAM3D}
  ${G2O_TYPES_SLAM3D_ADDONS}
)

include_directories(
  include
  ${PCL_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS}
  ${GTSAM_INCLUDE_DIRS}
)

# Nodelets
add_library(preprocessing_nodelet
  apps/preprocessing_nodelet.cpp
  src/radar_ego_velocity_estimator.cpp
)
target_link_libraries(preprocessing_nodelet
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${Boost_LIBRARIES}
)
add_dependencies(preprocessing_nodelet ${PROJECT_NAME}_gencpp)

add_library(scan_matching_odometry_nodelet
  apps/scan_matching_odometry_nodelet.cpp
  src/radar_graph_slam/registrations.cpp
)
target_link_libraries(scan_matching_odometry_nodelet
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
)
add_dependencies(scan_matching_odometry_nodelet ${PROJECT_NAME}_gencpp)

add_library(radar_graph_slam_nodelet
  apps/radar_graph_slam_nodelet.cpp
  src/radar_graph_slam/graph_slam.cpp
  src/radar_graph_slam/loop_detector.cpp
  src/radar_graph_slam/Scancontext.cpp
  src/radar_graph_slam/keyframe.cpp
  src/radar_graph_slam/map_cloud_generator.cpp
  src/radar_graph_slam/registrations.cpp
  src/radar_graph_slam/information_matrix_calculator.cpp
  src/g2o/robust_kernel_io.cpp
)
target_link_libraries(radar_graph_slam_nodelet
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
  ${G2O_TYPES_DATA}
  ${G2O_CORE_LIBRARY}
  ${G2O_STUFF_LIBRARY}
  ${G2O_SOLVER_PCG}
  ${G2O_SOLVER_CSPARSE}   # be aware of that CSPARSE is released under LGPL
  ${G2O_SOLVER_CHOLMOD}   # be aware of that cholmod is released under GPL
  ${G2O_TYPES_SLAM3D}
  ${G2O_TYPES_SLAM3D_ADDONS}
)
add_dependencies(radar_graph_slam_nodelet ${PROJECT_NAME}_gencpp)

# Adjust Groundtruth
add_executable(gt_adjust src/gt_adjust.cpp )
target_link_libraries(gt_adjust
  radar_graph_slam_nodelet
  ${catkin_LIBRARIES}
  ${Boost_LIBRARIES}
  ${G2O_TYPES_DATA}
  ${G2O_CORE_LIBRARY}
  ${G2O_STUFF_LIBRARY}
  ${G2O_SOLVER_PCG}
  ${G2O_SOLVER_CSPARSE}   # be aware of that CSPARSE is released under LGPL
  ${G2O_SOLVER_CHOLMOD}   # be aware of that cholmod is released under GPL
  ${G2O_TYPES_SLAM3D}
  ${G2O_TYPES_SLAM3D_ADDONS}
)

# Calculate UTM->world Transform
add_executable(gps_traj_align src/gps_traj_align.cpp )
target_link_libraries(gps_traj_align
  radar_graph_slam_nodelet
  ${catkin_LIBRARIES}
  ${Boost_LIBRARIES}
  ${G2O_TYPES_DATA}
  ${G2O_CORE_LIBRARY}
  ${G2O_STUFF_LIBRARY}
  ${G2O_SOLVER_PCG}
  ${G2O_SOLVER_CSPARSE}   # be aware of that CSPARSE is released under LGPL
  ${G2O_SOLVER_CHOLMOD}   # be aware of that cholmod is released under GPL
  ${G2O_TYPES_SLAM3D}
  ${G2O_TYPES_SLAM3D_ADDONS}
)

# Install python scripts
catkin_install_python(
  PROGRAMS
    src/${PROJECT_NAME}/bag_player.py
    src/${PROJECT_NAME}/ford2bag.py
    src/${PROJECT_NAME}/map2odom_publisher.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# Install nodelet plugins and targets
install(FILES nodelet_plugins.xml
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)

install(TARGETS
  preprocessing_nodelet
)

# Executables
add_executable(preprocessing_node
  nodes/preprocessing.cpp
)
target_link_libraries(preprocessing_node
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${Boost_LIBRARIES}
  4dradarslam
)
add_dependencies(preprocessing_node ${PROJECT_NAME}_gencpp)

add_executable(scan_matching_odometry_node
  nodes/scan_matching_odometry.cpp
)
target_link_libraries(scan_matching_odometry_node
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  4dradarslam
)
add_dependencies(scan_matching_odometry_node ${PROJECT_NAME}_gencpp)

add_executable(radar_graph_slam_node
  nodes/radar_graph_slam.cpp
)
target_link_libraries(radar_graph_slam_node
  ${catkin_LIBRARIES}
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
  ${G2O_TYPES_DATA}
  ${G2O_CORE_LIBRARY}
  ${G2O_STUFF_LIBRARY}
  ${G2O_SOLVER_PCG}
  ${G2O_SOLVER_CSPARSE}   # be aware of that CSPARSE is released under LGPL
  ${G2O_SOLVER_CHOLMOD}   # be aware of that cholmod is released under GPL
  ${G2O_TYPES_SLAM3D}
  ${G2O_TYPES_SLAM3D_ADDONS}
  4dradarslam
)
add_dependencies(radar_graph_slam_node ${PROJECT_NAME}_gencpp)
