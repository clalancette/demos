cmake_minimum_required(VERSION 3.5)

project(pendulum_control)

if(APPLE OR WIN32)
  message(STATUS "The pendulum_control demo is only supported on Linux. Package will not be built.")
  return()
endif()

find_package(ament_cmake REQUIRED)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -Wall -Wextra")

find_package(rclcpp REQUIRED)
find_package(pendulum_msgs REQUIRED)
find_package(rttest)
find_package(tlsf_cpp)

include_directories(include)

if(NOT rttest_FOUND)
  message(STATUS "rttest not found. pendulum_control package will not be built.")
  return()
endif()

add_executable(pendulum_demo${target_suffix}
  src/pendulum_demo.cpp)
ament_target_dependencies(pendulum_demo${target_suffix}
  "pendulum_msgs"
  "rclcpp${target_suffix}"
  "rttest"
  "tlsf_cpp")

add_executable(pendulum_logger${target_suffix}
  src/pendulum_logger.cpp)
ament_target_dependencies(pendulum_logger${target_suffix}
  "pendulum_msgs"
  "rclcpp${target_suffix}"
  "rttest")

add_executable(pendulum_teleop${target_suffix}
  src/pendulum_teleop.cpp)
ament_target_dependencies(pendulum_teleop${target_suffix}
  "pendulum_msgs"
  "rclcpp${target_suffix}"
  "rttest")

install(TARGETS
  pendulum_demo${target_suffix}
  pendulum_logger${target_suffix}
  pendulum_teleop${target_suffix}
  DESTINATION bin)

install(
  PROGRAMS "scripts/pendulum_launch.bash"
  DESTINATION bin
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()

  find_package(rmw_implementation_cmake REQUIRED)

  set(RCLCPP_DEMO_PENDULUM_LOGGER_EXPECTED_OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/test/pendulum_logger")
  set(RCLCPP_DEMO_PENDULUM_DEMO_EXPECTED_OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/test/pendulum_demo")
  set(RCLCPP_DEMO_PENDULUM_DEMO_TELEOP_EXPECTED_OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/test/pendulum_demo_teleop")
  set(RCLCPP_DEMO_PENDULUM_TELEOP_EXPECTED_OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/test/pendulum_teleop")

  get_available_rmw_implementations(rmw_implementations)
  foreach(rmw_implementation ${rmw_implementations})
    # FastRTPS doesn't (yet) support the use of custom allocators, so it will
    # fail this test, which requires that no actual allocation happens during
    # execution.
    if(NOT ${rmw_implementation} STREQUAL "rmw_fastrtps_cpp")
      set(RCLCPP_DEMO_PENDULUM_LOGGER_EXECUTABLE "${CMAKE_CURRENT_BINARY_DIR}/pendulum_logger")
      set(RCLCPP_DEMO_PENDULUM_DEMO_EXECUTABLE "${CMAKE_CURRENT_BINARY_DIR}/pendulum_demo")
      set(RCLCPP_DEMO_PENDULUM_TELEOP_EXECUTABLE "${CMAKE_CURRENT_BINARY_DIR}/pendulum_teleop")

      configure_file(
        test/test_pendulum_demo.py.in
        test_pendulum__${rmw_implementation}.py
        @ONLY
      )

      configure_file(
        test/test_pendulum_teleop.py.in
        test_pendulum_teleop__${rmw_implementation}.py
        @ONLY
      )

      configure_file(
        test/execute_with_delay.py
        execute_with_delay.py
      )

      file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/test_demo_pendulum__${rmw_implementation}")
      ament_add_nose_test(test_demo_pendulum__${rmw_implementation}
        "${CMAKE_CURRENT_BINARY_DIR}/test_pendulum__${rmw_implementation}.py"
        TIMEOUT 20
        ENV
        RCL_ASSERT_RMW_ID_MATCHES=${rmw_implementation}
        RMW_IMPLEMENTATION=${rmw_implementation}
        # since the test generates a .csv file in the cwd
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/test_demo_pendulum__${rmw_implementation}")
      set_tests_properties(test_demo_pendulum__${rmw_implementation}
        PROPERTIES DEPENDS "test_demo_pendulum__${rmw_implementation} test_demo_pendulum__${rmw_implementation}")

      ament_add_nose_test(test_demo_pendulum_teleop__${rmw_implementation}
        "${CMAKE_CURRENT_BINARY_DIR}/test_pendulum_teleop__${rmw_implementation}.py"
        TIMEOUT 20
        ENV
        RCL_ASSERT_RMW_ID_MATCHES=${rmw_implementation}
        RMW_IMPLEMENTATION=${rmw_implementation})
      set_tests_properties(test_demo_pendulum_teleop__${rmw_implementation}
        PROPERTIES DEPENDS "test_demo_pendulum_teleop__${rmw_implementation} test_demo_pendulum_teleop__${rmw_implementation}")
    endif()
  endforeach()
endif()

ament_package()
