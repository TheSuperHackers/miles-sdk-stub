cmake_minimum_required(VERSION 3.24.0...4.0.1)

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "No build type selected, default to Debug")
    set(CMAKE_BUILD_TYPE "Debug")
endif()

project(milesstub VERSION 1.0.0 LANGUAGES C)

option(MILES_NOFLOAT "Don't use float for some volume functions." OFF)

# Ensure CMAKE_DLLTOOL is available for MinGW builds
if(MINGW AND NOT CMAKE_DLLTOOL)
    find_program(CMAKE_DLLTOOL
        NAMES
            ${CMAKE_C_COMPILER_TARGET}-dlltool
            i686-w64-mingw32-dlltool
            x86_64-w64-mingw32-dlltool
            dlltool
    )
    if(NOT CMAKE_DLLTOOL)
        message(WARNING "CMAKE_DLLTOOL not found. Import library may have incorrect symbol decoration. Please set CMAKE_DLLTOOL in your toolchain file.")
    endif()
endif()

add_library(milescleanup STATIC cleanup.c)
add_library(milesstub SHARED miles.c)
set_target_properties(milesstub PROPERTIES OUTPUT_NAME mss32)
set_target_properties(milesstub PROPERTIES SOVERSION 1.0 VERSION 1.0.0)
if(WIN32)
    set_target_properties(milesstub PROPERTIES PREFIX "")
endif()
if(WIN32 AND CMAKE_SIZEOF_VOID_P EQUAL 4 AND MINGW)
    target_sources(milesstub PRIVATE miles.def)
    # Regenerate import library with correct underscore handling
    add_custom_command(TARGET milesstub POST_BUILD
        COMMAND ${CMAKE_DLLTOOL} --no-leading-underscore
            -D $<TARGET_FILE_NAME:milesstub>
            -d ${CMAKE_CURRENT_SOURCE_DIR}/miles.def
            -l $<TARGET_LINKER_FILE:milesstub>
        COMMENT "Regenerating import library with correct symbols"
    )
endif()
target_include_directories(milesstub PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>      #include <mss/mss.h>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/mss>  #include <mss.h>
)
target_compile_definitions(milesstub PRIVATE BUILD_STUBS)
target_link_libraries(milesstub INTERFACE milescleanup)

if(MILES_NOFLOAT)
    target_compile_definitions(milesstub PUBLIC MILES_NOFLOAT)
endif()
