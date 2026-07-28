cmake_minimum_required(VERSION 3.29)

set(TEMPLATE_NAME "VulkanTutorial")
project(${TEMPLATE_NAME} LANGUAGES CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Build type (Debug, Release, RelWithDebInfo, MinSizeRel)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

# Options
option(ENABLE_CPP20_MODULE "Enable C++ 20 module support for Vulkan" OFF)
option(ENABLE_TESTS "Enable unit tests" OFF)

# Enable module scanning only if needed
if(ENABLE_CPP20_MODULE)
  set(CMAKE_CXX_SCAN_FOR_MODULES ON)
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../cmake")

find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)
find_package(Vulkan 1.4.335 REQUIRED)
find_package(tinyobjloader REQUIRED)
#find_package(tinygltf REQUIRED)
#find_package(KTX REQUIRED)

# Vulkan C++ module setup (unchanged but cleaned up)
if(ENABLE_CPP20_MODULE)
  add_library(VulkanCppModule)
  add_library(Vulkan::cppm ALIAS VulkanCppModule)

  target_compile_definitions(VulkanCppModule PUBLIC 
    VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 
    VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
  )
  target_include_directories(VulkanCppModule PUBLIC "${Vulkan_INCLUDE_DIR}")
  target_link_libraries(VulkanCppModule PUBLIC Vulkan::Vulkan)
  set_target_properties(VulkanCppModule PROPERTIES CXX_STANDARD 20)

  if(MSVC)
    target_compile_options(VulkanCppModule PRIVATE /std:c++latest /permissive- /Zc:__cplusplus /EHsc /Zc:preprocessor /translateInclude)
  endif()

  target_sources(VulkanCppModule
    PUBLIC
      FILE_SET cxx_modules TYPE CXX_MODULES
      BASE_DIRS "${Vulkan_INCLUDE_DIR}"
      FILES "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
  )
else()
  add_library(VulkanCppModule INTERFACE)
  add_library(Vulkan::cppm ALIAS VulkanCppModule)
  target_link_libraries(VulkanCppModule INTERFACE Vulkan::Vulkan)
  target_compile_definitions(VulkanCppModule INTERFACE 
    VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 
    VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
  )
endif()

# STB
set(STB_INCLUDEDIR ${stb_INCLUDE_DIRS})

find_program(SLANGC_EXECUTABLE slangc HINTS $ENV{VULKAN_SDK}/bin REQUIRED)

# ==================== MULTIPLE SOURCE FILES ====================
set(SOURCES
  main.cpp
  # Add your other files here:
  # src/VulkanContext.cpp
  # src/Renderer.cpp
  # src/Utils.cpp
  # src/Window.cpp
  # etc.
)

# If you prefer automatic globbing (less recommended for large projects):
# file(GLOB_RECURSE SOURCES "src/*.cpp" "src/*.cxx")

add_executable(${TEMPLATE_NAME} ${SOURCES})

# Organize sources in IDEs (Visual Studio, Xcode, etc.)
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${SOURCES})

set_target_properties(${TEMPLATE_NAME} PROPERTIES 
  RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME}
  CXX_STANDARD 20
)

target_link_libraries(${TEMPLATE_NAME} PRIVATE Vulkan::cppm glfw)
target_include_directories(${TEMPLATE_NAME} PRIVATE ${STB_INCLUDEDIR})

target_compile_definitions(${TEMPLATE_NAME} PRIVATE 
  "VULKAN_HPP_HANDLE_ERROR_OUT_OF_DATE_AS_SUCCESS"
)

if(ENABLE_CPP20_MODULE)
  target_compile_definitions(${TEMPLATE_NAME} PRIVATE USE_CPP20_MODULES=1)
endif()

# ==================== BUILD TYPE FLAGS ====================
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  target_compile_definitions(${TEMPLATE_NAME} PRIVATE DEBUG)
  # Extra debug flags (already handled by CMake, but you can add more)
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  target_compile_definitions(${TEMPLATE_NAME} PRIVATE NDEBUG)
endif()

# ==================== TESTING ====================
if(ENABLE_TESTS)
  enable_testing()
  add_subdirectory(tests)  # Create a tests/ folder with its own CMakeLists.txt
endif()

# ==================== Slang Shaders ====================
set(SHADER_TARGET ${TEMPLATE_NAME}Shader)
set(SHADERS_DIR ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME}/shaders)
set(SHADER_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/shader.slang")

add_custom_command(
  OUTPUT ${SHADERS_DIR}
  COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADERS_DIR}
  COMMENT "Creating shaders directory"
)

add_custom_command(
  OUTPUT ${SHADERS_DIR}/slang.spv
  COMMAND ${SLANGC_EXECUTABLE} 
          ${SHADER_SOURCES} 
          -target spirv 
          -profile spirv_1_4 
          -emit-spirv-directly 
          -fvk-use-entrypoint-name 
          -entry vertMain 
          -entry fragMain 
          -o ${SHADERS_DIR}/slang.spv
  DEPENDS ${SHADER_SOURCES} ${SHADERS_DIR}
  COMMENT "Compiling Slang Shaders"
  VERBATIM
)

function (add_slang_shader_target TARGET)
  cmake_parse_arguments ("SHADER" "" "" "SOURCES" ${ARGN})
  set (SHADERS_DIR ${CMAKE_CURRENT_LIST_DIR}/shaders)
  set (ENTRY_POINTS -entry vertMain -entry fragMain)
  add_custom_command (
          OUTPUT ${SHADERS_DIR}
          COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADERS_DIR}
  )
  add_custom_command (
          OUTPUT  ${SHADERS_DIR}/slang.spv
          COMMAND ${SLANGC_EXECUTABLE} ${SHADER_SOURCES} -target spirv -profile spirv_1_4 -emit-spirv-directly -fvk-use-entrypoint-name ${ENTRY_POINTS} -o slang.spv
          WORKING_DIRECTORY ${SHADERS_DIR}
          DEPENDS ${SHADERS_DIR} ${SHADER_SOURCES}
          COMMENT "Compiling Slang Shaders"
          VERBATIM
  )
  add_custom_target (${TARGET} DEPENDS ${SHADERS_DIR}/slang.spv)
endfunction()

add_custom_target(${SHADER_TARGET} ALL DEPENDS ${SHADERS_DIR}/slang.spv)
add_dependencies(${TEMPLATE_NAME} ${SHADER_TARGET})

add_slang_shader_target( foo SOURCES ${SHADER_SLANG_SOURCES})

# Assets
set(ASSET_DIR "${CMAKE_CURRENT_SOURCE_DIR}/assets/")
file(COPY ${ASSET_DIR}/viking_room.obj DESTINATION ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME}/models)
file(COPY ${ASSET_DIR}/viking_room.png DESTINATION ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME}/textures)

# Windows-specific
if(WIN32 AND ${CMAKE_GENERATOR} MATCHES "Visual Studio.*")
  set_target_properties(${TEMPLATE_NAME} PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME})
endif()
