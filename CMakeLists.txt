cmake_minimum_required(VERSION 3.16)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

# See docs/release_checklist.md
set(MAJOR_VERSION 1)
set(MINOR_VERSION 0)
set(MICRO_VERSION 4)
set(SDL_REQUIRED_VERSION 2.0.10)

include(PrivateSdlFunctions)
sdl_calculate_derived_version_variables()

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    message(FATAL_ERROR "Prevented in-tree built. Please create a build directory outside of the SDL_gfx source code and call cmake from there")
endif()

project(SDL2_gfx
    LANGUAGES C
    VERSION "${FULL_VERSION}"
)

message(STATUS "Configuring ${PROJECT_NAME} ${PROJECT_VERSION}")

if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)
    set(SDL2GFX_ROOTPROJECT ON)
else()
    set(SDL2GFX_ROOTPROJECT OFF)
endif()

# Set defaults preventing destination file conflicts
set(SDL2GFX_DEBUG_POSTFIX "d"
    CACHE STRING "Name suffix for debug builds")
mark_as_advanced(SDL2GFX_DEBUG_POSTFIX)

if(ANDROID OR MSVC)
    set(vendored_default TRUE)
else()
    set(vendored_default FALSE)
endif()

set(sdl2gfx_install_enableable ON)
if ((TARGET SDL2 OR TARGET SDL2-static) AND SDL2_DISABLE_INSTALL)
    set(sdl2gfx_install_enableable OFF)
endif()

include(CMakeDependentOption)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)
include(CheckSymbolExists)

option(CMAKE_POSITION_INDEPENDENT_CODE "Build static libraries with -fPIC" ON)
option(BUILD_SHARED_LIBS "Build the library as a shared library" ON)

option(SDL2GFX_SAMPLES "Build the SDL2_gfx sample program(s)" ${SDL2GFX_ROOTPROJECT})
cmake_dependent_option(SDL2GFX_INSTALL "Enable SDL2_gfx install target" ${SDL2GFX_ROOTPROJECT} "${sdl2gfx_install_enableable}" OFF)
option(SDL2GFX_VENDORED "Use vendored third-party libraries" ${vendored_default})

# Save BUILD_SHARED_LIBS variable
set(SDL2GFX_BUILD_SHARED_LIBS "${BUILD_SHARED_LIBS}")

if(SDL2GFX_BUILD_SHARED_LIBS)
    set(sdl2_gfx_export_name SDL2_gfx)
    set(sdl2_gfx_install_name_infix shared)
    set(sdl2_target_name SDL2::SDL2)
else()
    set(sdl2_gfx_export_name SDL2_gfx-static)
    set(sdl2_gfx_install_name_infix static)
    set(sdl2_target_name SDL2::SDL2-static)
endif()

sdl_find_sdl2(${sdl2_target_name} ${SDL_REQUIRED_VERSION})

# Enable large file support on 32-bit glibc, so that the vendored libraries
# can access files with large inode numbers
check_symbol_exists("__GLIBC__" "stdlib.h" LIBC_IS_GLIBC)
if (LIBC_IS_GLIBC AND CMAKE_SIZEOF_VOID_P EQUAL 4)
    add_compile_definitions(_FILE_OFFSET_BITS=64)
endif()

add_library(SDL2_gfx
    SDL2_gfxPrimitives.c
    SDL2_gfxPrimitives.h
    SDL2_framerate.c
    SDL2_framerate.h
    SDL2_rotozoom.c
    SDL2_rotozoom.h
)
add_library(SDL2_gfx::${sdl2_gfx_export_name} ALIAS SDL2_gfx)
target_include_directories(SDL2_gfx PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL2>"
)
target_compile_definitions(SDL2_gfx PRIVATE
    BUILD_SDL
    SDL_BUILD_MAJOR_VERSION=${MAJOR_VERSION}
    SDL_BUILD_MINOR_VERSION=${MINOR_VERSION}
    SDL_BUILD_MICRO_VERSION=${MICRO_VERSION}
)
target_link_libraries(SDL2_gfx PRIVATE $<BUILD_INTERFACE:${sdl2_target_name}>)
if(WIN32 AND SDL2GFX_BUILD_SHARED_LIBS)
    target_sources(SDL2_gfx PRIVATE
        version.rc
    )
endif()
set_target_properties(SDL2_gfx PROPERTIES
    DEFINE_SYMBOL DLL_EXPORT
    EXPORT_NAME ${sdl2_gfx_export_name}
    C_VISIBILITY_PRESET "hidden"
)
if(NOT ANDROID)
    set_target_properties(SDL2_gfx PROPERTIES
        DEBUG_POSTFIX "${SDL2GFX_DEBUG_POSTFIX}"
    )
    if(APPLE)
        set_target_properties(SDL2_gfx PROPERTIES
            SOVERSION "${DYLIB_COMPATIBILITY_VERSION}"
            VERSION "${DYLIB_CURRENT_VERSION}"
        )
    else()
        set_target_properties(SDL2_gfx PROPERTIES
            SOVERSION "${LT_MAJOR}"
            VERSION "${LT_VERSION}"
        )
    endif()
endif()
if(SDL2GFX_BUILD_SHARED_LIBS AND (APPLE OR (UNIX AND NOT ANDROID)))
    add_custom_command(TARGET SDL2_gfx POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E create_symlink "$<TARGET_SONAME_FILE_NAME:SDL2_gfx>" "libSDL2_gfx$<$<CONFIG:Debug>:${SDL2GFX_DEBUG_POSTFIX}>$<TARGET_FILE_SUFFIX:SDL2_gfx>"
        WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
    )
endif()

if(SDL2GFX_BUILD_SHARED_LIBS)
    sdl_target_link_options_no_undefined(SDL2_gfx)
endif()

if(SDL2GFX_INSTALL)
    install(
        TARGETS SDL2_gfx
        EXPORT SDL2_gfxTargets
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT library
    )
    install(FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/SDL_gfxPrimitives.h"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL2" COMPONENT devel
    )

    if(WIN32 AND NOT MINGW)
        set(SDLGFX_INSTALL_CMAKEDIR_DEFAULT "cmake")
    else()
        set(SDLGFX_INSTALL_CMAKEDIR_DEFAULT "${CMAKE_INSTALL_LIBDIR}/cmake/SDL2_gfx")
    endif()
    set(SDLGFX_INSTALL_CMAKEDIR "${SDLGFX_INSTALL_CMAKEDIR_DEFAULT}" CACHE STRING "Location where to install SDL2_gfxConfig.cmake")

    configure_package_config_file(SDL2_gfxConfig.cmake.in SDL2_gfxConfig.cmake
        INSTALL_DESTINATION "${SDLGFX_INSTALL_CMAKEDIR}"
    )
    write_basic_package_version_file("${PROJECT_BINARY_DIR}/SDL2_gfxConfigVersion.cmake"
        VERSION ${FULL_VERSION}
        COMPATIBILITY AnyNewerVersion
    )
    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/SDL2_gfxConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/SDL2_gfxConfigVersion.cmake"
        DESTINATION ${SDLGFX_INSTALL_CMAKEDIR}
        COMPONENT devel
    )
    install(EXPORT SDL2_gfxTargets
        FILE SDL2_gfx-${sdl2_gfx_install_name_infix}-targets.cmake
        NAMESPACE SDL2_gfx::
        DESTINATION "${SDLGFX_INSTALL_CMAKEDIR}"
        COMPONENT devel
    )
endif()

if(SDL2GFX_SAMPLES)
    add_executable(testframerate test/testframerate.c)
    add_executable(testgfx test/testgfx.c)
    add_executable(testimagefilter test/testimagefilter.c)
    add_executable(testrotozoom test/testrotozoom.c)
    
    find_package(SDL2main QUIET)
    
    foreach(prog testframerate testgfx testimagefilter testrotozoom)
        if(MINGW)
            target_link_libraries(${prog} PRIVATE mingw32)
            target_link_options(${prog} PRIVATE -mwindows)
        endif()
        target_link_libraries(${prog} PRIVATE SDL2_gfx::${sdl2_gfx_export_name})
        if(TARGET SDL2::SDL2main)
            target_link_libraries(${prog} PRIVATE SDL2::SDL2main)
        endif()
        target_link_libraries(${prog} PRIVATE ${sdl2_target_name})
    endforeach()
endif()

add_library(SDL2::gfx INTERFACE IMPORTED GLOBAL)
set_target_properties(SDL2::gfx PROPERTIES
    INTERFACE_LINK_LIBRARIES "SDL2_gfx"
)
