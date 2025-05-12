cmake_minimum_required(VERSION 3.10)
project(orb_slam3 VERSION 1.0.0)


# list(APPEND CMAKE_PREFIX_PATH "/home/gsk/vcpkg/installed/x64-linux")

IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE Release)
ENDIF()

MESSAGE("Build type: " ${CMAKE_BUILD_TYPE})

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  -Wall   -O3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall   -O3")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -march=native")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")

# Check C++11 or C++0x support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
   add_definitions(-DCOMPILEDWITHC11)
   message(STATUS "Using flag -std=c++11.")
elseif(COMPILER_SUPPORTS_CXX0X)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
   add_definitions(-DCOMPILEDWITHC0X)
   message(STATUS "Using flag -std=c++0x.")
else()
   message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

find_package(OpenCV 4.4)
   if(NOT OpenCV_FOUND)
      message(FATAL_ERROR "OpenCV > 4.4 not found.")
   endif()

MESSAGE("OPENCV VERSION:")
MESSAGE(${OpenCV_VERSION})

find_package(Eigen3 3.1.0 REQUIRED)
find_package(Pangolin CONFIG REQUIRED)
find_package(realsense2)
find_package(Boost REQUIRED COMPONENTS serialization)


set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

file(GLOB_RECURSE ORB_SLAM3_SOURCES src/*.cc src/*.cpp)
file(GLOB_RECURSE ORB_SLAM3_HEADERS include/*.h include/*.hpp)

add_library(${PROJECT_NAME} SHARED
${ORB_SLAM3_SOURCES}
${ORB_SLAM3_HEADERS}
)

add_subdirectory(Thirdparty/g2o)
add_subdirectory(Thirdparty/DBoW2)
add_subdirectory(Thirdparty/Sophus)

target_link_libraries(${PROJECT_NAME}
${OpenCV_LIBS}
${EIGEN3_LIBS}
${Pangolin_LIBRARIES}
DBoW2
g2o
${Boost_LIBRARIES}
)

target_compile_options(${PROJECT_NAME} PRIVATE -Wno-maybe-uninitialized -Wno-unused-variable -Wno-unused-but-set-variable
-Wno-sign-compare -Wno-dangling-else -Wno-sign-compare -Wno-deprecated-declarations
-Wno-reorder -Wno-conversion-null -Wno-deprecated -Wno-unused-local-typedefs -Wno-aggressive-loop-optimizations
)

# If RealSense SDK is found the library is added and its examples compiled
if(realsense2_FOUND)
    include_directories(${PROJECT_NAME}
    ${realsense_INCLUDE_DIR}
    )
    target_link_libraries(${PROJECT_NAME}
    ${realsense2_LIBRARY}
    )
endif()

target_include_directories(
    ${PROJECT_NAME}
PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/CameraModels>
  $<INSTALL_INTERFACE:include>
  $<INSTALL_INTERFACE:include/${PROJECT_NAME}>
  $<INSTALL_INTERFACE:include/${PROJECT_NAME}/CameraModels>
  $<INSTALL_INTERFACE:include/${PROJECT_NAME}/Thirdparty/Sophus>
PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/Thirdparty/Sophus
  ${EIGEN3_INCLUDE_DIR}
  ${Pangolin_INCLUDE_DIRS}
)



include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

install(
  TARGETS
        ${PROJECT_NAME} g2o DBoW2
  EXPORT
        ${PROJECT_NAME}-targets
  ARCHIVE
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT lib
  RUNTIME
    DESTINATION ${CMAKE_INSTALL_BINDIR}
    COMPONENT bin
  LIBRARY
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT lib
  PUBLIC_HEADER
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
    COMPONENT dev
)

install(
  EXPORT
    ${PROJECT_NAME}-targets
  NAMESPACE
      ${PROJECT_NAME}::
  DESTINATION
      ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
  COMPONENT
      dev
)

# 添加头文件安装规则
install(
  DIRECTORY include/ 
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
  COMPONENT dev
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)

install(
  DIRECTORY  Thirdparty/DBoW2 Thirdparty/g2o Thirdparty/Sophus
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/Thirdparty
  COMPONENT dev
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)

write_basic_package_version_file (
  ${PROJECT_NAME}-configVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
  orb_slam3Config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

# Install cmake targets
install(
  FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-configVersion.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
)

# Build examples
option(BUILD_EXAMPLES "Build examples." OFF)
if(BUILD_EXAMPLES)
    add_subdirectory(Examples)
    add_subdirectory(Examples_old)
endif()