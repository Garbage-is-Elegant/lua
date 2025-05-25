cmake_minimum_required(VERSION 3.12)
project(Lua C)

set(CMAKE_C_STANDARD 99)

# Build options
option(BUILD_SHARED_LIBS "Build Lua as shared library" OFF)
option(BUILD_TESTING "Enable Lua test suite" ON)

# Set default config for multi-configuration generators (e.g., Visual Studio)
# Force Release configuration for CTest
set(CTEST_CONFIGURATION_TYPE "Release" CACHE STRING "Configuration for CTest" FORCE)

# Include CTest for test discovery
include(CTest)

# Enable testing if requested
if(BUILD_TESTING)
  enable_testing()
endif()

# Define source root
set(LUA_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

# Gather core Lua sources, excluding interpreter/compiler/single-file builds
file(GLOB LUA_CORE_SOURCES "${LUA_ROOT}/*.c")
list(FILTER LUA_CORE_SOURCES EXCLUDE REGEX "/lua\\.c$|/luac\\.c$|/onelua\\.c$")

# Core static library
add_library(luacore STATIC ${LUA_CORE_SOURCES})
target_include_directories(luacore PUBLIC ${LUA_ROOT})
set_target_properties(luacore PROPERTIES
  OUTPUT_NAME lua
  POSITION_INDEPENDENT_CODE ON
)

# Shared library (optional)
if(BUILD_SHARED_LIBS)
  add_library(lua_shared SHARED ${LUA_CORE_SOURCES})
  target_include_directories(lua_shared PUBLIC ${LUA_ROOT})
  if(WIN32)
    target_link_libraries(lua_shared PRIVATE luacore)
  else()
    find_library(M_LIB m)
    target_link_libraries(lua_shared PRIVATE luacore ${M_LIB})
  endif()
  set_target_properties(lua_shared PROPERTIES
    OUTPUT_NAME lua
    PREFIX ""
    WINDOWS_EXPORT_ALL_SYMBOLS ON
  )
endif()

# Lua interpreter executable
add_executable(lua ${LUA_ROOT}/lua.c)
target_include_directories(lua PRIVATE ${LUA_ROOT})
if(WIN32)
  target_link_libraries(lua PRIVATE luacore)
else()
  find_library(M_LIB m)
  target_link_libraries(lua PRIVATE luacore ${M_LIB})
endif()

# Installation rules
install(TARGETS luacore lua
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)
install(DIRECTORY ${LUA_ROOT}/
  DESTINATION include/lua
  FILES_MATCHING PATTERN "*.h"
)

# Register Lua tests if enabled
if(BUILD_TESTING)
  file(GLOB TEST_SCRIPTS "${LUA_ROOT}/testes/*.lua")
  foreach(test_script IN LISTS TEST_SCRIPTS)
    get_filename_component(test_name ${test_script} NAME_WE)
    add_test(NAME ${test_name}
      COMMAND $<TARGET_FILE:lua> ${test_script}
    )
    set_tests_properties(${test_name} PROPERTIES
      WORKING_DIRECTORY "${LUA_ROOT}/testes"
      ENVIRONMENT "LUA_PATH=${LUA_ROOT}/testes/?.lua;${LUA_ROOT}/?.lua;./?.lua"
    )
  endforeach()
endif()
