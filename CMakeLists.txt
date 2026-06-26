cmake_minimum_required(VERSION 3.10)
project(ndt_omp)

add_definitions(-std=c++14)
set(CMAKE_CXX_FLAGS "-std=c++14")

if (BUILD_WITH_MARCH_NATIVE)
  add_compile_options(-march=native)
else()
  add_definitions(-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2)
  set(CMAKE_CXX_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")
endif()

# (removed hardcoded RELEASE — PCL 1.7 debug segfault no longer relevant with PCL 1.15+)

find_package(PCL 1.7 REQUIRED)
include_directories(${PCL_INCLUDE_DIRS})
link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

message(STATUS "PCL_INCLUDE_DIRS:" ${PCL_INCLUDE_DIRS})
message(STATUS "PCL_LIBRARY_DIRS:" ${PCL_LIBRARY_DIRS})
message(STATUS "PCL_DEFINITIONS:" ${PCL_DEFINITIONS})

find_package(OpenMP)
if (OPENMP_FOUND)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

# ── Detect build mode ──────────────────────────────────────────────────────
# Priority: standalone CMake > ROS1 > ROS2
if(NOT DEFINED ENV{ROS_VERSION} OR "$ENV{ROS_VERSION}" STREQUAL "")
  # ====================================================================
  #  STANDALONE CMake mode (no ROS dependency — PCL + OpenMP only)
  # ====================================================================
  message(STATUS "ndt_omp: standalone mode (no ROS detected)")

  add_library(ndt_omp
    src/pclomp/voxel_grid_covariance_omp.cpp
    src/pclomp/ndt_omp.cpp
    src/pclomp/gicp_omp.cpp
  )

  target_include_directories(ndt_omp PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
  )
  target_link_libraries(ndt_omp PUBLIC ${PCL_LIBRARIES})

  if(OpenMP_CXX_FOUND)
    target_link_libraries(ndt_omp PUBLIC OpenMP::OpenMP_CXX)
    target_compile_options(ndt_omp PRIVATE ${OpenMP_CXX_FLAGS})
  else()
    message(WARNING "ndt_omp: OpenMP not found — will be single-threaded")
  endif()

  install(TARGETS ndt_omp
    EXPORT ndt_omp
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
  )
  install(DIRECTORY include/pclomp
    DESTINATION include)

elseif($ENV{ROS_VERSION} EQUAL 1)
  # ====================================================================
  #  ROS1 mode
  # ====================================================================
  find_package(catkin REQUIRED COMPONENTS
    roscpp
    pcl_ros
  )

  catkin_package(
    INCLUDE_DIRS include
    LIBRARIES ndt_omp
  )

  include_directories(include)
  include_directories(${catkin_INCLUDE_DIRS})

  add_library(ndt_omp
    src/pclomp/voxel_grid_covariance_omp.cpp
    src/pclomp/ndt_omp.cpp
    src/pclomp/gicp_omp.cpp
  )

  add_executable(align apps/align.cpp)
  add_dependencies(align ndt_omp)
  target_link_libraries(align ${catkin_LIBRARIES} ${PCL_LIBRARIES} ndt_omp)

  install(TARGETS ndt_omp
    ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
    RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
    LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )
  install(DIRECTORY include/pclomp
    DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION})
else()
  # ====================================================================
  #  ROS2 mode
  # ====================================================================
  find_package(ament_cmake_auto REQUIRED)
  ament_auto_find_build_dependencies()

  include_directories(include)

  add_library(ndt_omp
    SHARED
    src/pclomp/voxel_grid_covariance_omp.cpp
    src/pclomp/ndt_omp.cpp
    src/pclomp/gicp_omp.cpp
  )

  ament_export_targets(ndt_omp HAS_LIBRARY_TARGET)
  target_link_libraries(ndt_omp ${PCL_LIBRARIES})

  if(OpenMP_CXX_FOUND)
    target_link_libraries(ndt_omp OpenMP::OpenMP_CXX)
  endif()

  install(TARGETS ndt_omp
    EXPORT ndt_omp
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
  )

  ament_auto_package()
endif()
