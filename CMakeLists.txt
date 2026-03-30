cmake_minimum_required(VERSION 3.10)
cmake_policy(SET CMP0074 NEW)

if(MSVC)
    if(POLICY CMP0167)
        cmake_policy(SET CMP0167 NEW)
    endif()
else()
    cmake_policy(SET CMP0148 OLD)
endif()

project(ut_vslam)

option(SUPPRESS_WARNINGS "Suppress all compiler warnings" OFF)

# ------------------------------------------------------------------------------
# Compiler flags
# ------------------------------------------------------------------------------
if(MSVC)
    add_definitions(-DNOMINMAX)
    add_definitions(-D_USE_MATH_DEFINES)
    if(SUPPRESS_WARNINGS)
        add_compile_options(/W0 /wd4267 /wd4244)
    else()
        add_compile_options(/W3)
    endif()
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /O2")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O2")
else()
    if(SUPPRESS_WARNINGS)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -w -O3")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w -O3")
    else()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -O3")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -O3")
    endif()
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -march=native")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")
endif()

# ------------------------------------------------------------------------------
# Platform-specific default for ROS
# ------------------------------------------------------------------------------
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    option(USE_ROS "Build with ROS support" ON)
else()
    option(USE_ROS "Build with ROS support" OFF)
endif()

if(USE_ROS AND NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
    message(FATAL_ERROR "ROS is only supported on Linux. Set USE_ROS=OFF to build without ROS.")
endif()

option(CONSTRAIN_ELLIPSOID_ORIENTATION "Constrain ellipsoid orientation to yaw only" ON)
option(RUN_TIMERS "Add timers to profile annotated code" ON)
option(BUILD_TESTS "Build unit tests" OFF)

# C++17 standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Definitions for options
if(CONSTRAIN_ELLIPSOID_ORIENTATION)
    message(STATUS "Constraining ellipsoid orientation")
    add_definitions(-DCONSTRAIN_ELLIPSOID_ORIENTATION)
else()
    message(STATUS "Not constraining ellipsoid orientation")
endif()

if(RUN_TIMERS)
    message(STATUS "Adding timers")
    add_definitions(-DRUN_TIMERS)
else()
    message(STATUS "Not adding timers")
endif()

# Output directories
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

# ------------------------------------------------------------------------------
# Common dependencies (always required)
# ------------------------------------------------------------------------------
find_package(Ceres REQUIRED)
find_package(OpenCV 4 REQUIRED)
find_package(Boost 1.70 REQUIRED COMPONENTS filesystem)
find_package(glog REQUIRED)
find_package(gflags REQUIRED)
find_package(yaml-cpp REQUIRED)
find_package(Eigen3 REQUIRED NO_MODULE)

# std::filesystem (for older GCC)
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.0)
    set(FILESYSTEM_LIB "stdc++fs")
else()
    set(FILESYSTEM_LIB "")
endif()

# ------------------------------------------------------------------------------
# SuiteSparse/SPQR handling
# ------------------------------------------------------------------------------
find_package(SPQR CONFIG REQUIRED)
set(SUITESPARSE_LIBRARIES SuiteSparse::SPQR)
get_target_property(SUITESPARSE_INCLUDE_DIRS SuiteSparse::SPQR INTERFACE_INCLUDE_DIRECTORIES)

# ------------------------------------------------------------------------------
# ROS handling
# ------------------------------------------------------------------------------
if(USE_ROS)
    find_program(ROS_PKG_TOOL rospack)

    if(ROS_PKG_TOOL)
        execute_process(
            COMMAND "${ROS_PKG_TOOL}" find roscpp
            OUTPUT_VARIABLE ROSCPP_PKG_DIR
            OUTPUT_STRIP_TRAILING_WHITESPACE
            RESULT_VARIABLE ROSCPP_FIND_RESULT
        )

        if(ROSCPP_FIND_RESULT EQUAL 0 AND ROSCPP_PKG_DIR)
            get_filename_component(ROS_PREFIX "${ROSCPP_PKG_DIR}/../.." ABSOLUTE)
            list(PREPEND CMAKE_PREFIX_PATH "${ROS_PREFIX}")
        endif()
    endif()

    find_package(catkin REQUIRED COMPONENTS
        roscpp
        roslib
        rosbag
        cv_bridge
        pcl_conversions
        pcl_ros
        amrl_msgs
    )

    set(ROS_LIBS ${catkin_LIBRARIES} amrl_shared_lib)
    set(ROS_INCLUDE_DIRS ${catkin_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR}/ros/include)
    file(GLOB_RECURSE ROS_SOURCES "${PROJECT_SOURCE_DIR}/ros/src/*.cpp")

    add_definitions(-DUSE_ROS)
else()
    set(ROS_LIBS "")
    set(ROS_INCLUDE_DIRS "")
    set(ROS_SOURCES "")
endif()

# ------------------------------------------------------------------------------
# Include directories
# ------------------------------------------------------------------------------
include_directories(include)
include_directories(${PROJECT_SOURCE_DIR}/src/shared)
include_directories(src)
if(USE_ROS)
    include_directories(${ROS_INCLUDE_DIRS})
endif()
if(SUITESPARSE_INCLUDE_DIRS)
    include_directories(${SUITESPARSE_INCLUDE_DIRS})
endif()

# ------------------------------------------------------------------------------
# Core library sources (without ROS-dependent files)
# ------------------------------------------------------------------------------
set(UT_VSLAM_SOURCES
    src/evaluation/evaluation_utils.cpp
    src/evaluation/object_evaluation_utils.cpp
    # src/evaluation/trajectory_evaluation_utils.cpp  # moved to ros
    # src/evaluation/trajectory_interpolation_utils.cpp  # moved to ros
    src/refactoring/bounding_box_frontend/pending_object_estimator.cpp
    src/refactoring/factors/bounding_box_factor.cpp
    src/refactoring/factors/pairwise_2d_feature_cost_functor.cpp
    src/refactoring/factors/independent_object_map_factor.cpp
    src/refactoring/factors/relative_pose_factor.cpp
    src/refactoring/factors/reprojection_cost_functor_analytic_jacobian.cpp
    src/refactoring/factors/reprojection_cost_functor.cpp
    src/refactoring/factors/shape_prior_factor.cpp
    # src/refactoring/image_processing/debugging_image_utils.cpp  # moved to ros
    src/refactoring/long_term_map/long_term_object_map_extraction.cpp
    src/refactoring/optimization/jacobian_extraction.cpp
    src/refactoring/types/vslam_obj_opt_types_refactor.cpp
    src/refactoring/visual_feature_processing/orb_output_low_level_feature_reader.cpp
    src/types/timestamped_data_to_frames_utils.cpp
    src/shared/util/timer.cc
)

# Add ROS sources if enabled
if(USE_ROS)
    list(APPEND UT_VSLAM_SOURCES ${ROS_SOURCES})
endif()

add_library(ut_vslam ${UT_VSLAM_SOURCES})
target_compile_features(ut_vslam PRIVATE cxx_std_17)

# Common libraries
set(COMMON_LIBS
    ${FILESYSTEM_LIB}
    glog::glog
    gflags::gflags
    Ceres::ceres
    ${OpenCV_LIBS}
    Boost::filesystem
    yaml-cpp::yaml-cpp
    Eigen3::Eigen
    ${SUITESPARSE_LIBRARIES}
)

target_link_libraries(ut_vslam ${COMMON_LIBS} ${ROS_LIBS})

# ------------------------------------------------------------------------------
# Executables that do NOT depend on ROS (always built)
# ------------------------------------------------------------------------------
# These executables are known not to use ROS (they remain in original src/ and were not moved to ros)
set(NON_ROS_EXECS
    sequence_file_writer
)

# For each non-ROS executable, define its source path (original location)
foreach(exec ${NON_ROS_EXECS})
    if(exec STREQUAL "sequence_file_writer")
        set(src_file src/data_preprocessing_utils/sequence_file_writer.cpp)
    else()
        message(FATAL_ERROR "Unknown non-ROS executable: ${exec}")
    endif()
    add_executable(${exec} ${src_file})
    target_link_libraries(${exec} ut_vslam ${COMMON_LIBS})
endforeach()

# ------------------------------------------------------------------------------
# Executables that depend on ROS (built only if USE_ROS=ON)
# ------------------------------------------------------------------------------
if(USE_ROS)
    set(ROS_EXECS
        offline_object_visual_slam_main
        initialize_traj_and_feats_from_orb_out
        orb_trajectory_sparsifier
        localization_rosbag_extraction_and_interpolation_for_bounding_boxes
        test_det_call
        interpolate_poses_with_required_nodes
        oa_slam_data_generator
        trajectory_metrics_generator
        object_metrics_generator
        utvslam_out_to_poses_by_timestamp
        visualize_jacobian_problem_feats
        debug_jacobian_hessian_diagonal
        ltm_extraction_only
        display_ltm
        run_opt_from_pg_state
        write_bounding_boxes_for_rosbag_to_file
        waypoint_timestamp_extractor
    )

    foreach(exec ${ROS_EXECS})
        # Determine source path inside ros/src (mirroring original structure)
        if(exec STREQUAL "offline_object_visual_slam_main")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/offline_object_visual_slam_main.cpp)
        elseif(exec STREQUAL "initialize_traj_and_feats_from_orb_out")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/data_preprocessing_utils/unproject_main.cpp)
        elseif(exec STREQUAL "orb_trajectory_sparsifier")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/data_preprocessing_utils/orb_trajectory_sparsifier.cpp)
        elseif(exec STREQUAL "localization_rosbag_extraction_and_interpolation_for_bounding_boxes")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/data_preprocessing_utils/localization_rosbag_extraction_and_interpolation_for_bounding_boxes.cpp)
        elseif(exec STREQUAL "test_det_call")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/testing/test_obj_det_service.cpp)
        elseif(exec STREQUAL "interpolate_poses_with_required_nodes")
            set(src_file ${PROJECT_SOURCE_DIR}/src/evaluation/interpolate_poses_with_required_nodes.cpp)
        elseif(exec STREQUAL "oa_slam_data_generator")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/evaluation/oa_slam/oa_slam_data_generator.cpp)
        elseif(exec STREQUAL "trajectory_metrics_generator")
            set(src_file ${PROJECT_SOURCE_DIR}/src/evaluation/trajectory_metrics_generator.cpp)
        elseif(exec STREQUAL "object_metrics_generator")
            set(src_file ${PROJECT_SOURCE_DIR}/src/evaluation/object_metrics_generator.cpp)
        elseif(exec STREQUAL "utvslam_out_to_poses_by_timestamp")
            set(src_file ${PROJECT_SOURCE_DIR}/src/evaluation/trajectory_formatting/utvslam_out_to_poses_by_timestamp.cpp)
        elseif(exec STREQUAL "visualize_jacobian_problem_feats")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/visualize_jacobian_problem_feats.cpp)
        elseif(exec STREQUAL "debug_jacobian_hessian_diagonal")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/debugging_utils/debug_jacobian_hessian_diagonal.cpp)
        elseif(exec STREQUAL "ltm_extraction_only")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/ltm_extraction_only.cpp)
        elseif(exec STREQUAL "display_ltm")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/display_ltm.cpp)
        elseif(exec STREQUAL "run_opt_from_pg_state")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/run_opt_from_pg_state.cpp)
        elseif(exec STREQUAL "write_bounding_boxes_for_rosbag_to_file")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/data_preprocessing_utils/write_bounding_boxes_for_rosbag_to_file.cpp)
        elseif(exec STREQUAL "waypoint_timestamp_extractor")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/data_preprocessing_utils/waypoint_timestamp_extractor.cpp)
        elseif(exec STREQUAL "convert_object_detections_from_yaml")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/evaluation/objects/convert_object_detections_from_yaml.cpp)
        elseif(exec STREQUAL "gt_transformer")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/evaluation/gt_transformer.cpp)
        elseif(exec STREQUAL "lego_loam_overlay_generator")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/evaluation/lego_loam_overlay_generator.cpp)
        elseif(exec STREQUAL "lego_loam_traj_length_evaluator")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/evaluation/lego_loam_traj_length_evaluator.cpp)
        elseif(exec STREQUAL "write_configuration")
            set(src_file ${PROJECT_SOURCE_DIR}/ros/src/refactoring/configuration/write_configuration.cpp)
        else()
            message(FATAL_ERROR "Unknown ROS executable: ${exec}")
        endif()
        if(NOT EXISTS ${src_file})
            message(FATAL_ERROR "Source file for ${exec} not found at ${src_file}")
        endif()
        add_executable(${exec} ${src_file})
        target_link_libraries(${exec} ut_vslam ${COMMON_LIBS} ${ROS_LIBS})
    endforeach()
endif()

# ------------------------------------------------------------------------------
# Unit tests
# ------------------------------------------------------------------------------
if(USE_ROS)
    if(BUILD_TESTS)
        find_package(GTest CONFIG REQUIRED)
        add_executable(ut_vslam_tests
            ros/test/file_io/cv_file_storage/config_file_storage_io_tests.cpp
            ros/test/file_io/cv_file_storage/sequence_file_storage_io_tests.cpp
            ros/test/file_io/cv_file_storage/object_and_reprojection_feature_pose_graph_file_storage_io_tests.cpp
        )
        target_link_libraries(ut_vslam_tests
            GTest::gtest
            GTest::gtest_main
            ut_vslam
            ${COMMON_LIBS}
            ${ROS_LIBS}
        )
    endif()
endif()