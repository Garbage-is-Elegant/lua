cmake_minimum_required(VERSION 3.12)
project(Lua C)

set(CMAKE_C_STANDARD 99)

# Options
option(BUILD_SHARED_LIBS "Build Lua as shared library" OFF)
option(BUILD_TESTING "Build Lua test suite" ON)

# Include CTest for test discovery
include(CTest)

# Enable testing if requested
if(BUILD_TESTING)
    enable_testing()
endif()

# Source directory
set(LUA_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

# Collect all Lua core sources (excluding interpreter and compiler)
file(GLOB LUA_CORE_SOURCES "${LUA_ROOT}/*.c")
# Exclude lua.c (interpreter) and luac.c (compiler, if present)
list(FILTER LUA_CORE_SOURCES EXCLUDE REGEX "/lua\.c$|/luac\.c$")

# Library target
add_library(luacore STATIC ${LUA_CORE_SOURCES})
set_target_properties(luacore PROPERTIES
    OUTPUT_NAME lua
    POSITION_INDEPENDENT_CODE ON
)

# Shared library configuration
if(BUILD_SHARED_LIBS)
    set_target_properties(luacore PROPERTIES
        PREFIX ""
        WINDOWS_EXPORT_ALL_SYMBOLS ON
        SUFFIX ".so"
    )
    add_library(lua_shared SHARED ${LUA_CORE_SOURCES})
    target_link_libraries(lua_shared PRIVATE m)
    set_target_properties(lua_shared PROPERTIES OUTPUT_NAME lua)
endif()

# Include directories per target
target_include_directories(luacore PUBLIC ${LUA_ROOT})

# Lua interpreter
add_executable(lua ${LUA_ROOT}/lua.c)
target_link_libraries(lua PRIVATE luacore m)

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

# Tests: register each .lua in testes as a ctest
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
