cmake_minimum_required(VERSION 3.12)
project(Lua C)

set(CMAKE_C_STANDARD 99)

# Options
option(BUILD_SHARED_LIBS "Build Lua as shared library" OFF)
option(BUILD_TESTING "Enable Lua test suite" ON)

# Set default config for multi-configuration generators (e.g., Visual Studio)
set(CTEST_CONFIGURATION_TYPE "Release")

# Include CTest for test discovery
include(CTest)

# Enable testing if requested
if(BUILD_TESTING)
  enable_testing()
endif()

# Source directory
set(LUA_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

# Collect all Lua core sources (excluding interpreter, compiler, and single-file builds)
file(GLOB LUA_CORE_SOURCES "${LUA_ROOT}/*.c")
# Exclude lua.c, luac.c, onelua.c
list(FILTER LUA_CORE_SOURCES EXCLUDE REGEX "/lua\\.c$|/luac\\.c$|/onelua\\.c$")

# Core library target
add_library(luacore STATIC ${LUA_CORE_SOURCES})
set_target_properties(luacore PROPERTIES
  OUTPUT_NAME lua
  POSITION_INDEPENDENT_CODE ON
)
target_include_directories(luacore PUBLIC ${LUA_ROOT})

# Shared library configuration
if(BUILD_SHARED_LIBS)
  add_library(lua_shared SHARED ${LUA_CORE_SOURCES})
  if(MSVC)
    target_link_libraries(lua_shared PRIVATE luacore)
  else()
    target_link_libraries(lua_shared PRIVATE luacore m)
  endif()
  set_target_properties(lua_shared PROPERTIES
    OUTPUT_NAME lua
    PREFIX ""
    WINDOWS_EXPORT_ALL_SYMBOLS ON
  )
endif()

# Lua interpreter executable
add_executable(lua ${LUA_ROOT}/lua.c)
if(MSVC)
  target_link_libraries(lua PRIVATE luacore)
else()
  target_link_libraries(lua PRIVATE luacore m)
endif()
target_include_directories(lua PRIVATE ${LUA_ROOT})

# Install rules
install(TARGETS luacore lua
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)
install(DIRECTORY ${LUA_ROOT}/
  DESTINATION include/lua
  FILES_MATCHING PATTERN "*.h"
)

# Tests: discover .lua scripts under testes directory
if(BUILD_TESTING)
  file(GLOB TEST_SCRIPTS "${LUA_ROOT}/testes/*.lua")
  foreach(test_script ${TEST_SCRIPTS})
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
