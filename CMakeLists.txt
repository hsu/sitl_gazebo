cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
project(mavlink_sitl_gazebo VERSION 1.0.0)

if(NOT CMAKE_CXX_STANDARD)
 set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
 add_compile_options(-Wall -Wextra -Wpedantic)
endif()

option(BUILD_GSTREAMER_PLUGIN "enable gstreamer plugin" ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

find_package(ament_cmake REQUIRED)
find_package(Boost 1.58 REQUIRED COMPONENTS system thread filesystem)
find_package(gazebo REQUIRED)
find_package(OpenCV REQUIRED)
find_package(TinyXML REQUIRED)

if (BUILD_GSTREAMER_PLUGIN)
  pkg_check_modules(GSTREAMER REQUIRED gstreamer-1.0)
  if (GSTREAMER_FOUND)
    if("${GAZEBO_VERSION}" VERSION_LESS "8.0")
      find_package (Qt4)
      include (${QT_USE_FILE})
    else()
      # In order to find Qt5 in macOS, the Qt5 path needs to be added to the CMake prefix path.
      if(APPLE)
        execute_process(COMMAND brew --prefix qt5
                        ERROR_QUIET
                        OUTPUT_VARIABLE QT5_PREFIX_PATH
                        OUTPUT_STRIP_TRAILING_WHITESPACE
                       )
        list(APPEND CMAKE_PREFIX_PATH "${QT5_PREFIX_PATH}/lib/cmake")
      endif()
      find_package(Qt5 COMPONENTS Core Widgets REQUIRED)
    endif()
  endif()
endif()

set(PROTOBUF_IMPORT_DIRS "")
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(PROTOBUF_IMPORT_DIRS "${ITR}/gazebo/msgs/proto")
  endif()
endforeach()
# PROTOBUF_IMPORT_DIRS has to be set before
# find_package is called
find_package(Protobuf REQUIRED)
pkg_check_modules(PROTOBUF protobuf)

if ("${PROTOBUF_VERSION}" VERSION_LESS "2.5.0")
  message(FATAL_ERROR "protobuf version: ${PROTOBUF_VERSION} not compatible, must be >= 2.5.0")
endif()

if("${GAZEBO_VERSION}" VERSION_LESS "6.0")
  message(FATAL_ERROR "You need at least Gazebo 6.0. Your version: ${GAZEBO_VERSION}")
else()
  message(STATUS "Gazebo version: ${GAZEBO_VERSION}")
endif()

find_package(Eigen3 QUIET)
if(NOT EIGEN3_FOUND)
  # Fallback to cmake_modules
  find_package(Eigen QUIET)
  if(NOT EIGEN_FOUND)
    pkg_check_modules(EIGEN3 REQUIRED eigen3)
  else()
    set(EIGEN3_INCLUDE_DIRS ${EIGEN_INCLUDE_DIRS})
    set(EIGEN3_LIBRARIES ${EIGEN_LIBRARIES})
  endif()
else()
  set(EIGEN3_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})
endif()

find_package(MAVLink)

###########
## Build ##
###########

set(GAZEBO_MSG_INCLUDE_DIRS)
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(GAZEBO_MSG_INCLUDE_DIRS "${ITR}/gazebo/msgs")
  endif()
endforeach()

include_directories(
  include
  ${CMAKE_CURRENT_BINARY_DIR}
  ${EIGEN3_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIRS}/eigen3	# Workaround for Eigen3
  ${GAZEBO_MSG_INCLUDE_DIRS}
  ${GAZEBO_INCLUDE_DIRS}
  ${MAVLINK_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
)

link_libraries(
  ${OpenCV_LIBRARIES}
)

if (GSTREAMER_FOUND)
  include_directories(
    ${GSTREAMER_INCLUDE_DIRS}
  )
endif()

if (GSTREAMER_FOUND)
  link_libraries(
    ${GSTREAMER_LIBRARIES}
    glib-2.0
    gobject-2.0
  )
endif()

function(glob_generate target file_glob)
  file(GLOB_RECURSE glob_files RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ${file_glob})
	set(gen_files)
	foreach(glob_file ${glob_files})
		string(REGEX REPLACE "\\.[^.]*$" "" file_name ${glob_file})
		string(REGEX MATCH "[^.]*$" file_ext ${glob_file})
		get_filename_component(file_dir ${glob_file} DIRECTORY)
		set(in_file ${CMAKE_CURRENT_SOURCE_DIR}/${glob_file})
		file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/${file_dir})
		set(out_file ${CMAKE_CURRENT_SOURCE_DIR}/${file_name})
		string(REGEX REPLACE ".sdf" "-gen.sdf" out_file ${out_file})
		if (${file_ext} STREQUAL "jinja")
			add_custom_command(OUTPUT ${out_file}
        COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/jinja_gen.py
        ${in_file} ${CMAKE_CURRENT_SOURCE_DIR}
				DEPENDS ${in_file}
				VERBATIM
				)
			list(APPEND gen_files_${target} ${out_file})
		endif()
	endforeach()
	add_custom_target(${target} ALL DEPENDS ${gen_files_${target}})
endfunction()

glob_generate(models_gen ${CMAKE_CURRENT_SOURCE_DIR}/models/*.jinja)

#--------------------#
# Message Generation #
#--------------------#

set(mav_msgs
  msgs/CommandMotorSpeed.proto
  msgs/MotorSpeed.proto
  )
set(nav_msgs msgs/Odometry.proto)
set(physics_msgs msgs/Wind.proto)
set(std_msgs msgs/Int32.proto)
set(sensor_msgs
  msgs/Imu.proto
  msgs/IRLock.proto
  msgs/Float.proto
  msgs/Groundtruth.proto
  msgs/Range.proto
  msgs/SITLGps.proto
  msgs/OpticalFlow.proto
  msgs/MagneticField.proto
  msgs/Pressure.proto
)

PROTOBUF_GENERATE_CPP(MAV_PROTO_SRCS MAV_PROTO_HDRS ${mav_msgs})
PROTOBUF_GENERATE_CPP(NAV_PROTO_SRCS NAV_PROTO_HDRS ${nav_msgs})
PROTOBUF_GENERATE_CPP(PHY_PROTO_SRCS PHY_PROTO_HDRS ${physics_msgs})
PROTOBUF_GENERATE_CPP(STD_PROTO_SRCS STD_PROTO_HDRS ${std_msgs})
PROTOBUF_GENERATE_CPP(SEN_PROTO_SRCS SEN_PROTO_HDRS ${sensor_msgs})


#---------#
# Plugins #
#---------#

add_library(mav_msgs SHARED ${MAV_PROTO_SRCS})
add_library(nav_msgs SHARED ${NAV_PROTO_SRCS})
add_library(physics_msgs SHARED ${PHY_PROTO_SRCS})
add_library(std_msgs SHARED ${STD_PROTO_SRCS})
add_library(sensor_msgs SHARED ${SEN_PROTO_SRCS})

link_libraries(mav_msgs nav_msgs std_msgs sensor_msgs physics_msgs)

add_library(gazebo_geotagged_images_plugin SHARED src/gazebo_geotagged_images_plugin.cpp)
add_library(gazebo_gps_plugin SHARED src/gazebo_gps_plugin.cpp)
add_library(gazebo_irlock_plugin SHARED src/gazebo_irlock_plugin.cpp)
add_library(gazebo_lidar_plugin SHARED src/gazebo_lidar_plugin.cpp)
# add_library(gazebo_opticalflow_plugin SHARED src/gazebo_opticalflow_plugin.cpp)
add_library(gazebo_sonar_plugin SHARED src/gazebo_sonar_plugin.cpp)
add_library(gazebo_uuv_plugin SHARED src/gazebo_uuv_plugin.cpp)
add_library(gazebo_vision_plugin SHARED src/gazebo_vision_plugin.cpp)
add_library(gazebo_controller_interface SHARED src/gazebo_controller_interface.cpp)
add_library(gazebo_gimbal_controller_plugin SHARED src/gazebo_gimbal_controller_plugin.cpp)
add_library(gazebo_imu_plugin SHARED src/gazebo_imu_plugin.cpp)
add_library(gazebo_mavlink_interface SHARED src/gazebo_mavlink_interface.cpp )
add_library(gazebo_motor_model SHARED src/gazebo_motor_model.cpp)
add_library(gazebo_multirotor_base_plugin SHARED src/gazebo_multirotor_base_plugin.cpp)
add_library(gazebo_wind_plugin SHARED src/gazebo_wind_plugin.cpp)
add_library(gazebo_magnetometer_plugin SHARED src/gazebo_magnetometer_plugin.cpp src/geo_mag_declination.cpp)
add_library(gazebo_barometer_plugin SHARED src/gazebo_barometer_plugin.cpp)

set(plugins
  gazebo_geotagged_images_plugin
  gazebo_gps_plugin
  gazebo_irlock_plugin
  gazebo_lidar_plugin
  # gazebo_opticalflow_plugin
  gazebo_sonar_plugin
  gazebo_uuv_plugin
  gazebo_vision_plugin
  gazebo_controller_interface
  gazebo_gimbal_controller_plugin
  gazebo_imu_plugin
  gazebo_mavlink_interface
  gazebo_motor_model
  gazebo_multirotor_base_plugin
  gazebo_wind_plugin
  gazebo_magnetometer_plugin
  gazebo_barometer_plugin
)

foreach(plugin ${plugins})
  target_link_libraries(${plugin} ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})
endforeach()

if (GSTREAMER_FOUND)
  add_library(gazebo_gst_camera_plugin SHARED src/gazebo_gst_camera_plugin.cpp)
  set(plugins
    ${plugins}
    gazebo_gst_camera_plugin
  )
  message(STATUS "Found GStreamer: adding gst_camera_plugin")
  if("${GAZEBO_VERSION}" VERSION_LESS "8.0")
    QT4_WRAP_CPP(headers_MOC include/gazebo_video_stream_widget.h)
    add_library(gazebo_video_stream_widget SHARED ${headers_MOC} src/gazebo_video_stream_widget.cpp)
    target_link_libraries(gazebo_video_stream_widget ${GAZEBO_LIBRARIES} ${QT_LIBRARIES} ${PROTOBUF_LIBRARIES})
    set(plugins
      ${plugins}
      gazebo_video_stream_widget
    )
    message(STATUS "Found GStreamer: adding gst_video_stream_widget")
  else()
    QT5_WRAP_CPP(headers_MOC include/gazebo_video_stream_widget.h)
    add_library(gazebo_video_stream_widget SHARED ${headers_MOC} src/gazebo_video_stream_widget.cpp)
    target_link_libraries(gazebo_video_stream_widget ${GAZEBO_LIBRARIES} ${Qt5Core_LIBRARIES} ${Qt5Widgets_LIBRARIES} ${PROTOBUF_LIBRARIES} ${Qt5Test_LIBRARIES})
    set(plugins
      ${plugins}
      gazebo_video_stream_widget
    )
    message(STATUS "Found GStreamer: adding gst_video_stream_widget")
  endif()
endif()

install(DIRECTORY models
 DESTINATION share/${PROJECT_NAME}
)

install(TARGETS
  ${plugins}
  mav_msgs
  nav_msgs
  std_msgs
  sensor_msgs
  physics_msgs
 ARCHIVE DESTINATION lib
 LIBRARY DESTINATION lib
 RUNTIME DESTINATION bin)

ament_environment_hooks(cmake/50_sitl_gazebo_setup.sh.in)
ament_package()