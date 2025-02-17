cmake_minimum_required(VERSION 3.10)

set(RUCKIG_VERSION 0.11.0)
project(ruckig VERSION ${RUCKIG_VERSION} LANGUAGES CXX)


include(GNUInstallDirs)


option(BUILD_EXAMPLES "Build example programs" ON)
option(BUILD_PYTHON_MODULE "Build python module with pybind11" OFF)
option(BUILD_CLOUD_CLIENT "Build cloud client to calculate Ruckig Pro trajectories remotely" ON)
option(BUILD_TESTS "Build tests" ON)
option(BUILD_BENCHMARK "Build benchmark" OFF)
option(BUILD_SHARED_LIBS "Build as shared library" ON)

if(WIN32 AND BUILD_SHARED_LIBS)
  option(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS "On Windows, export all symbols when building a shared library." ON)
endif()


add_library(ruckig
  src/ruckig/brake.cpp
  src/ruckig/position_first_step1.cpp
  src/ruckig/position_first_step2.cpp
  src/ruckig/position_second_step1.cpp
  src/ruckig/position_second_step2.cpp
  src/ruckig/position_third_step1.cpp
  src/ruckig/position_third_step2.cpp
  src/ruckig/velocity_second_step1.cpp
  src/ruckig/velocity_second_step2.cpp
  src/ruckig/velocity_third_step1.cpp
  src/ruckig/velocity_third_step2.cpp
)
add_library(ruckig::ruckig ALIAS ruckig)

target_compile_features(ruckig PUBLIC cxx_std_17)
target_include_directories(ruckig PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_link_libraries(ruckig PUBLIC)


if(MSVC)
  target_compile_definitions(ruckig PUBLIC _USE_MATH_DEFINES)
endif()


if(BUILD_CLOUD_CLIENT)
  target_sources(ruckig PRIVATE src/ruckig/cloud_client.cpp)
  target_include_directories(ruckig PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third_party>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )
  target_compile_definitions(ruckig PUBLIC WITH_CLOUD_CLIENT)
endif()


if(BUILD_EXAMPLES)
  set(EXAMPLES_LIST 01_position 02_position_offline 05_velocity 06_stop 07_minimum_duration 09_dynamic_dofs 12_custom_vector_type 13_custom_vector_type_dynamic_dofs)
  if(TARGET Eigen3::Eigen)
    list(APPEND EXAMPLES_LIST 11_eigen_vector_type)
  endif()

  if(BUILD_CLOUD_CLIENT)
    list(APPEND EXAMPLES_LIST 03_waypoints 04_waypoints_online 08_per_section_minimum_duration 10_dynamic_dofs_waypoints)
  endif()

  foreach(example IN LISTS EXAMPLES_LIST)
    add_executable(example-${example} examples/${example}.cpp)
    target_link_libraries(example-${example} PRIVATE ruckig)
  endforeach()
endif()


if(BUILD_PYTHON_MODULE)
  # Check if pybind11 exists as a subdirectory
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/pybind11)
    add_subdirectory(pybind11)
  else()
    find_package(pybind11 2.6 REQUIRED)
  endif()

  pybind11_add_module(python_ruckig src/ruckig/python.cpp)
  target_compile_features(python_ruckig PUBLIC cxx_std_17)
  target_link_libraries(python_ruckig PUBLIC ruckig)
  if(BUILD_CLOUD_CLIENT)
    target_compile_definitions(python_ruckig PUBLIC WITH_CLOUD_CLIENT)
  endif()

  set_target_properties(python_ruckig PROPERTIES OUTPUT_NAME ruckig)
  set_target_properties(python_ruckig PROPERTIES ARCHIVE_OUTPUT_NAME python_ruckig)
endif()


# Add support for installation
include(CMakePackageConfigHelpers)

# Install headers
install(DIRECTORY include/ruckig DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Install library
install(TARGETS ruckig
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Install CMake config files
set(ruckig_INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})

install(EXPORT ${PROJECT_NAME}-targets
  DESTINATION ${ruckig_INSTALL_CONFIGDIR}
  NAMESPACE ruckig::
)

configure_package_config_file(cmake/ruckig-config.cmake.in ruckig-config.cmake
  INSTALL_DESTINATION ${ruckig_INSTALL_CONFIGDIR}
)

write_basic_package_version_file(ruckig-config-version.cmake
  VERSION ${ruckig_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/ruckig-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/ruckig-config-version.cmake"
  DESTINATION ${ruckig_INSTALL_CONFIGDIR}
)

install(FILES
  "${CMAKE_CURRENT_SOURCE_DIR}/package.xml"
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PROJECT_NAME}
)


if(BUILD_TESTS)
  enable_testing()

  add_executable(test-target test/test_target.cpp test/test_target_known.cpp)

  target_link_libraries(test-target PRIVATE ruckig)
  target_include_directories(test-target PRIVATE third_party)
  if(MSVC)
    target_compile_options(test-target PRIVATE /W4)
  else()
    target_compile_options(test-target PRIVATE -Wall -Wextra)
    find_program(MEMORYCHECK_COMMAND valgrind)
    set(MEMORYCHECK_COMMAND_OPTIONS
       "--tool=memcheck \
       --leak-check=full \
       --trace-children=yes \
       --track-origins=yes \
       --keep-debuginfo=yes \
       --error-exitcode=100")
    include(CTest)
  endif()

  add_test(NAME test-target COMMAND test-target)
endif()


if(BUILD_BENCHMARK)
  add_executable(benchmark-target test/benchmark_target.cpp)

  target_link_libraries(benchmark-target PRIVATE ruckig)
endif()


# Enable Packaging
set(CPACK_PACKAGE_VENDOR "Lars Berscheid")
set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGE_VERSION ${RUCKIG_VERSION})
set(CPACK_SYSTEM_NAME ${CMAKE_HOST_SYSTEM_PROCESSOR})

# Settings for Debian package
set(CPACK_DEBIAN_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION}-1)
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Lars Berscheid")
set(CPACK_DEBIAN_PACKAGE_CONFLICTS "ros-melodic-ruckig, ros-noetic-ruckig, ros-foxy-ruckig, ros-galactic-ruckig, ros-humble-ruckig, ros-rolling-ruckig")
include(CPack)
