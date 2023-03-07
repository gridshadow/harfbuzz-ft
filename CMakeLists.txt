cmake_minimum_required(VERSION 3.13.0)
project(harfbuzz-ft VERSION 0.1.0)

# Ensure freetype & harfbuzz are available by updating submodules using git
find_package(Git QUIET)
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    SET(USE_FREETYPE_TAG "" CACHE STRING "Specifies FreeType tag to checkout")
    SET(USE_HARFBUZZ_TAG "" CACHE STRING "Specifies HarfBuzz tag to checkout")
    option(GIT_SUBMODULE "Check submodules during build" ON)
    if(GIT_SUBMODULE)
        message(STATUS "Submodule update")
        execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --remote --init
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        RESULT_VARIABLE GIT_SUBMOD_RESULT)
        if(NOT GIT_SUBMOD_RESULT EQUAL "0")
            message(FATAL_ERROR "git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
        endif()
    endif()
    if (NOT "${USE_FREETYPE_TAG}" STREQUAL "")
        message(STATUS "Checking out FreeType tag: ${USE_FREETYPE_TAG}")
        execute_process(COMMAND ${GIT_EXECUTABLE} checkout "${USE_FREETYPE_TAG}"
                        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/freetype"
                        RESULT_VARIABLE GIT_CHECKOUT_RESULT
                        COMMAND_ECHO STDOUT)
        if(NOT GIT_CHECKOUT_RESULT EQUAL "0")
            message(FATAL_ERROR "git checkout ${USE_FREETYPE_TAG} failed - is it a valid tag?")
        endif()
    endif()
    if (NOT "${USE_HARFBUZZ_TAG}" STREQUAL "")
        message(STATUS "Checking out HarfBuzz tag: ${USE_HARFBUZZ_TAG}")
        execute_process(COMMAND ${GIT_EXECUTABLE} checkout "${USE_HARFBUZZ_TAG}"
                        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/harfbuzz"
                        RESULT_VARIABLE GIT_CHECKOUT_RESULT)
        if(NOT GIT_CHECKOUT_RESULT EQUAL "0")
            message(FATAL_ERROR "git checkout ${USE_HARFBUZZ_TAG} failed - is it a valid tag?")
        endif()
    endif()
endif()

# Options from FreeType for disabling various support libraries
option(DISABLE_ZLIB "Disable use of system zlib and use internal zlib library instead." OFF)
option(DISABLE_PNG "Disable support of PNG compressed OpenType embedded bitmaps." OFF)
option(DISABLE_BROTLI "Disable support of compressed WOFF2 fonts." OFF)

# FreeType doesn't detect if libraries are built as part of this compilation
# So check if zlib, libpng, or brotli are built (have targets) and then "fake" it
# for FreeType - disable automatic finding of these libraries by setting FT_DISABLE_*
# and then set the libraries find_package should set to the target
if(NOT DISABLE_ZLIB AND TARGET zlib)
    set(FT_DISABLE_ZLIB ON CACHE BOOL "")
    set(ZLIB_FOUND ON)
    set(ZLIB_LIBRARIES zlib)
endif()
if(NOT DISABLE_PNG)
    if(TARGET png)
        set(FT_DISABLE_PNG ON CACHE BOOL "")
        set(PNG_FOUND ON)
        set(PNG_LIBRARIES png)
    elseif(TARGET png_static)
        set(FT_DISABLE_PNG ON CACHE BOOL "")
        set(PNG_FOUND ON)
        set(PNG_LIBRARIES png_static)
    endif()
endif()
if(NOT DISABLE_BROTLI AND TARGET brotlidec)
    set(FT_DISABLE_BROTLI ON CACHE BOOL "")
    set(BROTLIDEC_FOUND ON)
    set(BROTLIDEC_LIBRARIES brotlidec)
endif()

# Read FreeType version number:

file(READ "freetype/CMakeLists.txt" FREETYPE_CMAKE)
string(REGEX MATCH "set\\(VERSION_MAJOR \"([0-9]+)\"\\)" _ ${FREETYPE_CMAKE})
set(FREETYPE_VERSION_MAJOR ${CMAKE_MATCH_1})
string(REGEX MATCH "set\\(VERSION_MINOR \"([0-9]+)\"\\)" _ ${FREETYPE_CMAKE})
set(FREETYPE_VERSION_MINOR ${CMAKE_MATCH_1})
string(REGEX MATCH "set\\(VERSION_PATCH \"([0-9]+)\"\\)" _ ${FREETYPE_CMAKE})
set(FREETYPE_VERSION_PATCH ${CMAKE_MATCH_1})
message(STATUS "Using FreeType version ${FREETYPE_VERSION_MAJOR}.${FREETYPE_VERSION_MINOR}.${FREETYPE_VERSION_PATCH}")

# Read HarfBuzz version number:

file(READ "harfbuzz/configure.ac" HARFBUZZ_CONFIGUREAC)
string(REGEX MATCH "\\[([0-9]+\\.[0-9]+\\.[0-9]+)\\]" _ ${HARFBUZZ_CONFIGUREAC})
message(STATUS "Using HarfBuzz version ${CMAKE_MATCH_1}")

# Solving the freetype->harfbuzz->freetype dependency chain is actually simpler with CMake than
# the usual build FreeType, then HarfBuzz, then recompile FreeType

# Disable HarfBuzz library search in FreeType (by setting FT_DISABLE_HARFBUZZ to ON)
set(FT_DISABLE_HARFBUZZ ON CACHE BOOL "")
# Set HARFBUZZ_FOUND to ON; hence, FreeType thinks that harfBuzz was found
set(HARFBUZZ_FOUND ON)
# Now add freetype library -> this will configure FreeType with HarfBuzz support,
# and create the ftconfig.h and ftoption.h header, which is the required dependency for HarfBuzz
add_subdirectory(freetype)
# Add the HarfBuzz library now, which will detect FreeType and setup to build with FreeType support
add_subdirectory(harfbuzz)
# Now we add the cyclic dependency of FreeType on HarfBuzz
target_link_libraries(freetype PRIVATE harfbuzz)
