# FreeType/HarfBuzz CMake (harfbuzz-ft)

This project is designed as a simplified way to add both FreeType and HarfBuzz
as dependent libraries of a CMake project, with automatic building, without the
usual FreeType $\rightarrow$ HarfBuzz $\rightarrow$ FreeType cyclic dependency issue.

**Warning**: HarfBuzz apparently has deprecated the CMake build and may soon remove the CMake
build in favour of Meson.

## Table of Contents

 * [Basic Usage](#basic-usage)
 * [Versions](#versions)
 * [Options](#options)
 * [Libraries](#libraries)
 * [Issues](#issues)
 * [Implementation Details](#implementation-details)

## Basic Usage

Either checkout the project, or use `git submodule` to add this project as
a subfolder of the main CMake project. Then, in the `CMakeLists.txt` for the
main project add
```cmake
add_subdirectory(path_to_harfbuzz-ft)
```
where `path_to_harfbuzz-ft` is the path to the folder where this project is
checked out.

The FreeType and HarfBuzz git repositories are automatically checked out, or
can be checked out manually be calling
```
git submodule update --init --recursive
```
inside this directory.

## Versions

Each harfbuzz-ft release is pinned to a FreeType and HarfBuzz versions

| harfbuzz-ft | FreeType | HarfBuzz |
|:-----------:|:--------:|:--------:|
| 1.0.2       | 2.13.3   | 11.2.1   |

The used version can be changed with the following CMake options during CMake
configuration:

  * `USE_FREETYPE_TAG`
  * `USE_HARFBUZZ_TAG`

where these options should be set to a git tag from the relevant FreeType
or HarfBuzz repositories:

  * <https://gitlab.freedesktop.org/freetype/freetype>
  * <https://github.com/harfbuzz/harfbuzz>

These options require git.

**Warning:** Running cmake configuration without these options will re-use the
same options used during the last configuration. To revert to the default use
the version number for `USE_HARFBUZZ_TAG` and `VER-{MAJOR}-{MINOR}-{REVISION}`
for `USE_FREETYPE_TAG` for the current harfbuzz-ft version.

**TODO:** Improving this to automatically revert (or allow revert to be
specified by arguments).

## Options

The following options are supported:

  * `GIT_SUBMODULE` (Boolean; Default=ON) - Enable update of git submodules;
    i.e., automatically checkout FreeType and HarfBuzz.
  * `BUILD_SHARED_LIBS` (Boolean; Default=ON) - Build FreeType and HarfBuzz
    as shared libraries. To build as static libraries only set to `OFF`.
  * `DISABLE_ZLIB` (Boolean; Default=OFF) Require system zlib instead of internal zlib library
  * `DISABLE_PNG` (Boolean; Default=OFF) Require support of PNG compressed OpenType embedded bitmaps
  * `DISABLE_BROTLI` (Boolean; Default=OFF) Require support of compressed WOFF2 fonts

The following options are for FreeType. FreeType by default
automatically detects optional libraries, and disables them if not found. It
is possible to disable the libraries completely, or require them using the
following options:

  * `FT_REQUIRE_ZLIB` (Boolean; Default=OFF) Require system zlib instead of internal zlib library
  * `FT_REQUIRE_BZIP2` (Boolean; Default=OFF) Require support of bzip2 compressed fonts
  * `FT_REQUIRE_PNG` (Boolean; Default=OFF) Require support of PNG compressed OpenType embedded bitmaps
  * `FT_REQUIRE_BROTLI` (Boolean; Default=OFF) Require support of compressed WOFF2 fonts
  * `FT_DISABLE_BZIP2` (Boolean; Default=OFF) Disable support of bzip2 compressed fonts

The following options are inherited from HarfBuzz:

  * `HB_HAVE_GRAPHITE2` (Boolean; Default=OFF) Enable Graphite2 complementary shaper
  * `HB_HAVE_GLIB` (Boolean; Default=OFF) Enable glib unicode functions
  * `HB_HAVE_ICU` (Boolean; Default=OFF) Enable icu unicode functions
  * `HB_HAVE_GOBJECT` (Boolean; Default=OFF) Enable GObject Bindings

The following options are inherited from HarfBuzz on macOS only:

  * `HB_HAVE_CORETEXT` (Boolean; Default=ON) Enable CoreText shaper backend on macOS

The following options are inherited from HarfBuzz on Windows only:

  * `HB_HAVE_UNISCRIBE` (Boolean; Default=OFF) Enable Uniscribe shaper backend on Windows
  * `HB_HAVE_GDI` (Boolean; Default=OFF) Enable GDI integration helpers on Windows
  * `HB_HAVE_DIRECTWRITE` (Boolean; Default=OFF) Enable DirectWrite shaper backend on Windows

## Libraries

FreeType automatically detects the presence of the following libraries (if not
disabled via `FT_DISABLE_*` options):

  * zlib (<https://zlib.net/>)
  * bzip2 (<https://sourceware.org/bzip2/>)
  * libpng (<http://www.libpng.org/pub/png/libpng.html>)
  * brotli (<https://github.com/google/brotli>)

HarfBuzz builds with no libraries by default; however, automatic detection of
the following libraries can enabled with `HB_HAVE_*` or if the CMake targets
for these libraries are available:

  * Graphite2 (<https://graphite.sil.org/>)
  * GLib (<https://docs.gtk.org/glib/>)
  * International Components for Unicode (ICU) (<https://icu.unicode.org/>)
  * GObject (<https://docs.gtk.org/gobject/>)

If any of these libraries are made as part of the project, make sure the
targets are defined BEFORE the
```cmake
add_subdirectory(path_to_harfbuzz-ft)
```
line (by adding subdirectories for their sources for example).

## Implementation Details

Solving the FreeType $\rightarrow$ HarfBuzz $\rightarrow$ FreeType dependency
chain is actually simpler with CMake than the usual build FreeType, then
HarfBuzz, then recompile FreeType.

The required steps are as follows:

  * Disable HarfBuzz library search in FreeType (by setting
    `FT_DISABLE_HARFBUZZ` to `ON`):
    ```cmake
    set(FT_DISABLE_HARFBUZZ ON CACHE BOOL "")
    ```
  * Set `HARFBUZZ_FOUND` to `ON`; hence, FreeType thinks that HarfBuzz was
    found:
    ```cmake
    set(HARFBUZZ_FOUND ON)
    ```
  * Now add FreeType library - this will configure FreeType with HarfBuzz
    support, and create the `ftconfig.h` and `ftoption.h` headers, which is the
    required dependency for HarfBuzz
    ```cmake
    add_subdirectory(freetype)
    ```
  * Add the HarfBuzz library now, which will detect FreeType and setup to build
    with FreeType support
    ```cmake
    add_subdirectory(harfbuzz)
    ```
  * Now we add the cyclic dependency of FreeType on HarfBuzz
    ```cmake
    target_link_libraries(freetype PRIVATE harfbuzz)
    ```
