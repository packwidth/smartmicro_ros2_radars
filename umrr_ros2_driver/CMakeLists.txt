# Copyright 2021 Apex.AI, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# The initial version of the code was developed by Apex.AI and
# was thereafter adapted and extended by smartmicro.

cmake_minimum_required(VERSION 3.11)

project(umrr_ros2_driver)
include(FetchContent)

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wall -Wextra -Wnarrowing -Wpedantic -Wno-unused-parameter -g)
endif()

find_package(ament_cmake_auto REQUIRED)
find_package(Threads REQUIRED)
find_package(visualization_msgs REQUIRED)

fetchcontent_declare(json
    URL https://github.com/nlohmann/json/releases/download/v3.12.0/include.zip
    URL_HASH SHA256=b8cb0ef2dd7f57f18933997c9934bb1fa962594f701cd5a8d3c2c80541559372
)

fetchcontent_getproperties(json)
if(NOT json_POPULATED)
    fetchcontent_populate(json)
    add_library(nlohmann_json INTERFACE)
    target_include_directories(nlohmann_json INTERFACE ${json_SOURCE_DIR}/include)
    add_library(nlohmann_json::nlohmann_json ALIAS nlohmann_json)
endif()

ament_auto_find_build_dependencies()

# The smartmicro SDK ships prebuilt libraries in directories named
# lib-<os>-<arch>-<compiler>_<major>. Derive it from the current toolchain
# unless the user pinned one explicitly via -DSMARTMICRO_LIB_DIR=...
if(NOT SMARTMICRO_LIB_DIR)
    string(TOLOWER "${CMAKE_SYSTEM_NAME}" _sm_os)

    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64|amd64)$")
        set(_sm_arch "x86_64")
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)$")
        set(_sm_arch "armv8")
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^armv7")
        set(_sm_arch "armv7hf")
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(i[3-6]86|x86)$")
        set(_sm_arch "x86")
    else()
        message(FATAL_ERROR "No smartmicro libraries for architecture '${CMAKE_SYSTEM_PROCESSOR}'")
    endif()

    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        set(_sm_compiler "gcc")
    elseif(MSVC)
        set(_sm_compiler "msvc")
    else()
        # Clang and friends are ABI-compatible with the gcc builds on Linux.
        set(_sm_compiler "gcc")
    endif()

    if(MSVC)
        # Directories use the Visual Studio version (16 == VS2019 == toolset 142).
        math(EXPR _sm_compiler_major "${MSVC_TOOLSET_VERSION} - 126")
    else()
        string(REGEX MATCH "^[0-9]+" _sm_compiler_major "${CMAKE_CXX_COMPILER_VERSION}")
    endif()

    # Pick the newest shipped toolchain that is not newer than ours.
    file(GLOB _sm_candidates
        RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/smartmicro"
        "${CMAKE_CURRENT_SOURCE_DIR}/smartmicro/lib-${_sm_os}-${_sm_arch}-${_sm_compiler}_*")

    set(_sm_best_version 0)
    foreach(_sm_candidate IN LISTS _sm_candidates)
        string(REGEX MATCH "[0-9]+$" _sm_candidate_version "${_sm_candidate}")
        if(_sm_candidate_version
           AND NOT _sm_candidate_version GREATER _sm_compiler_major
           AND _sm_candidate_version GREATER _sm_best_version)
            set(_sm_best_version "${_sm_candidate_version}")
            set(SMARTMICRO_LIB_DIR "${_sm_candidate}")
        endif()
    endforeach()

    if(NOT SMARTMICRO_LIB_DIR)
        message(FATAL_ERROR
            "No smartmicro libraries found for lib-${_sm_os}-${_sm_arch}-${_sm_compiler}_<=${_sm_compiler_major}. "
            "Available: ${_sm_candidates}")
    endif()
endif()

message(STATUS "Using smartmicro libraries from ${SMARTMICRO_LIB_DIR}")

# specify the path to search for the libraries
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR})

install(FILES
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libsmart_access.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libosal.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libcom_lib.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libbasev1.0.2_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr11_t132_automotivev1.1.2_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr96_t153_automotivev1.2.2_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev1.1.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev2.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev2.1.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev2.2.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev2.4.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev3.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_automotivev3.2.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9d_t152_automotivev1.0.3_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9d_t152_automotivev1.2.2_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9d_t152_automotivev1.4.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9d_t152_automotivev1.5.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9d_t152_automotivev1.7.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_automotivev1.0.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_automotivev1.2.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_automotivev1.4.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_automotivev1.6.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_msev1.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_msev1.1.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_msev1.3.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrr9f_t169_msev2.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_msev1.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_msev2.1.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra4_msev3.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra1_t166_b_automotivev1.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra1_t166_b_automotivev2.0.0_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra1_t166_b_automotivev2.0.1_user_interface.so"
    "${PROJECT_SOURCE_DIR}/smartmicro/${SMARTMICRO_LIB_DIR}/libumrra1_t166_b_automotivev3.0.0_user_interface.so"
DESTINATION lib)

set(LIB_PATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CONFIG_FOLDER_PATH "${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME}/config")
set(CONFIG_FILE_PATH "${CONFIG_FOLDER_PATH}/smart_access_config.json")
set(HW_INVENTORY_FILE_PATH "${CONFIG_FOLDER_PATH}/hw_inventory.json")
set(ROUTING_TABLE_FILE_PATH "${CONFIG_FOLDER_PATH}/routing_table.json")
configure_file(
    "${PROJECT_SOURCE_DIR}/cmake/smart_access_config.json.in" "${CONFIG_FILE_PATH}" @ONLY)
configure_file(
    "${PROJECT_SOURCE_DIR}/cmake/config_path.hpp.in"
    "${CMAKE_INSTALL_PREFIX}/include/${PROJECT_NAME}/config_path.hpp" @ONLY)

ament_auto_add_library(smartmicro_radar_node SHARED 
    "src/smartmicro_radar_node.cpp"
    "src/update_service.cpp"
    )

target_include_directories(smartmicro_radar_node PUBLIC ${CMAKE_INSTALL_PREFIX}/include/
smartmicro/include/
smartmicro/include/umrr11_t132_automotive_v1_1_2
smartmicro/include/umrr96_t153_automotive_v1_2_2
smartmicro/include/umrr9f_t169_automotive_v1_1_1
smartmicro/include/umrr9f_t169_automotive_v2_0_0
smartmicro/include/umrr9f_t169_automotive_v2_1_1
smartmicro/include/umrr9f_t169_automotive_v2_2_1
smartmicro/include/umrr9f_t169_automotive_v2_4_1
smartmicro/include/umrr9f_t169_automotive_v3_0_0
smartmicro/include/umrr9f_t169_automotive_v3_2_0
smartmicro/include/umrr9d_t152_automotive_v1_0_3
smartmicro/include/umrr9d_t152_automotive_v1_2_2
smartmicro/include/umrr9d_t152_automotive_v1_4_1
smartmicro/include/umrr9d_t152_automotive_v1_5_0
smartmicro/include/umrr9d_t152_automotive_v1_7_0
smartmicro/include/umrra4_automotive_v1_0_1
smartmicro/include/umrra4_automotive_v1_2_1
smartmicro/include/umrra4_automotive_v1_4_0
smartmicro/include/umrra4_automotive_v1_6_0
smartmicro/include/umrr9f_t169_mse_v1_0_0
smartmicro/include/umrr9f_t169_mse_v1_1_0
smartmicro/include/umrr9f_t169_mse_v1_3_0
smartmicro/include/umrr9f_t169_mse_v2_0_0
smartmicro/include/umrra4_mse_v1_0_0
smartmicro/include/umrra4_mse_v2_1_0
smartmicro/include/umrra4_mse_v3_0_0
smartmicro/include/umrra1_t166_b_automotive_v1_0_0
smartmicro/include/umrra1_t166_b_automotive_v2_0_0
smartmicro/include/umrra1_t166_b_automotive_v2_0_1
smartmicro/include/umrra1_t166_b_automotive_v3_0_0)

# link smart_access_lib-linux-armv8-gcc_9 to the node
target_link_libraries(smartmicro_radar_node
    smart_access
    nlohmann_json::nlohmann_json
    com_lib
    osal
    umrr11_t132_automotivev1.1.2_user_interface
    umrr96_t153_automotivev1.2.2_user_interface
    umrr9f_t169_automotivev1.1.1_user_interface
    umrr9f_t169_automotivev2.0.0_user_interface
    umrr9f_t169_automotivev2.1.1_user_interface
    umrr9f_t169_automotivev2.2.1_user_interface
    umrr9f_t169_automotivev2.4.1_user_interface
    umrr9f_t169_automotivev3.0.0_user_interface
    umrr9f_t169_automotivev3.2.0_user_interface
    umrr9d_t152_automotivev1.0.3_user_interface
    umrr9d_t152_automotivev1.2.2_user_interface
    umrr9d_t152_automotivev1.4.1_user_interface
    umrr9d_t152_automotivev1.5.0_user_interface
    umrr9d_t152_automotivev1.7.0_user_interface
    umrra4_automotivev1.0.1_user_interface
    umrra4_automotivev1.2.1_user_interface
    umrra4_automotivev1.4.0_user_interface
    umrra4_automotivev1.6.0_user_interface
    umrr9f_t169_msev1.0.0_user_interface
    umrr9f_t169_msev1.1.0_user_interface
    umrr9f_t169_msev1.3.0_user_interface
    umrr9f_t169_msev2.0.0_user_interface
    umrra4_msev1.0.0_user_interface
    umrra4_msev2.1.0_user_interface
    umrra4_msev3.0.0_user_interface
    umrra1_t166_b_automotivev1.0.0_user_interface
    umrra1_t166_b_automotivev2.0.0_user_interface
    umrra1_t166_b_automotivev2.0.1_user_interface
    umrra1_t166_b_automotivev3.0.0_user_interface)

rclcpp_components_register_node(smartmicro_radar_node
    PLUGIN "smartmicro::drivers::radar::SmartmicroRadarNode"
    EXECUTABLE smartmicro_radar_node_exe
)

if(BUILD_TESTING)
    find_package(launch_testing_ament_cmake)
    add_launch_test(test/radar_node_test.launch.py TIMEOUT "100")
       
    # Not applying ros2 linters on external sources
    list(APPEND AMENT_LINT_AUTO_EXCLUDE
        ament_cmake_copyright
        ament_cmake_lint_cmake
        ament_cmake_uncrustify
        ament_cmake_cpplint
        ament_cmake_flake8
        ament_cmake_cppcheck
        ament_cmake_pep257
    )

    find_package(ament_lint_auto REQUIRED)
    ament_lint_auto_find_test_dependencies()

    set(PROJECT_CPP_FILES
      src/smartmicro_radar_node.cpp
      src/update_service.cpp
      include/umrr_ros2_driver/smartmicro_radar_node.hpp
      include/umrr_ros2_driver/update_service.hpp
    )

    set(PROJECT_PY_FILES
      launch/radar.launch.py
      test/radar_node_test.launch.py
    )

    # C++ linting
    ament_cppcheck(${PROJECT_CPP_FILES})

    # Python linting
    ament_pep257(${PROJECT_PY_FILES})

    # Copyright should cover all first-party text sources.
    ament_copyright(
      CMakeLists.txt
      ${PROJECT_CPP_FILES}
      ${PROJECT_PY_FILES}
    )

endif()

ament_auto_package(INSTALL_TO_SHARE config launch param)
