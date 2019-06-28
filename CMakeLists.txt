cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)
project(harfbuzz)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# Some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# Set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR "${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers" CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(SKIP_INSTALL_HEADERS ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)
    set(CMAKE_MACOSX_RPATH ON)
    set(SKIP_INSTALL_DOCS ON)
    set(BUILD_FRAMEWORK ON)
    add_definitions(-DMAC_OS_X_VERSION_MIN_REQUIRED=101100)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Share directory for documents" FORCE)
    set(INSTALL_MAN_DIR ${CMAKE_INSTALL_MANDIR} CACHE INTERNAL "Share directory for manuals" FORCE)
    set(INSTALL_PKGCONFIG_DIR "${INSTALL_LIB_DIR}/pkgconfig" CACHE INTERNAL "Installation directory for pkgconfig (.pc) files" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(INSTALL_SHARE_DIR ${CMAKE_INSTALL_DATADIR} CACHE INTERNAL "Installation directory for share data" FORCE)
endif()

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

## Limit framework build to Xcode generator
# if (BUILD_FRAMEWORK)
#   # for a framework on macOS, use `cmake .. -DBUILD_FRAMEWORK:BOOL=true -G Xcode`
#   if (NOT "${CMAKE_GENERATOR}" STREQUAL "Xcode")
#     message(FATAL_ERROR
#       "You should use Xcode generator with BUILD_FRAMEWORK enabled")
#   endif ()
#   set(CMAKE_OSX_ARCHITECTURES "$(ARCHS_STANDARD_32_64_BIT)")
#   set(CMAKE_MACOSX_RPATH ON)
#   set(BUILD_SHARED_LIBS ON)
# endif ()


## Disallow in-source builds, as CMake generated make files can collide with autotools ones
if (NOT MSVC AND "${PROJECT_BINARY_DIR}" STREQUAL "${PROJECT_SOURCE_DIR}")
  message(FATAL_ERROR
    "
In-source builds are not permitted!  Make a separate folder for"
    " building, e.g.,"
    "
  mkdir build; cd build; cmake .."
    "
Before that, remove the files created by this failed run with"
    "
  rm -rf CMakeCache.txt CMakeFiles")
endif ()


## HarfBuzz build configurations
option(HB_HAVE_FREETYPE "Enable freetype interop helpers" OFF)
option(HB_HAVE_GRAPHITE2 "Enable Graphite2 complementary shaper" OFF)
option(HB_BUILTIN_UCDN "Use HarfBuzz provided UCDN" ON)
option(HB_HAVE_GLIB "Enable glib unicode functions" OFF)
option(HB_HAVE_ICU "Enable icu unicode functions" OFF)
if (APPLE)
  option(HB_HAVE_CORETEXT "Enable CoreText shaper backend on macOS" ON)
endif ()
if (WIN32)
  option(HB_HAVE_UNISCRIBE "Enable Uniscribe shaper backend on Windows" OFF)
  option(HB_HAVE_DIRECTWRITE "Enable DirectWrite shaper backend on Windows" OFF)
endif ()
option(HB_BUILD_UTILS "Build harfbuzz utils, needs cairo, freetype, and glib properly be installed" OFF)
if (HB_BUILD_UTILS)
  set(HB_HAVE_GLIB ON)
  set(HB_HAVE_FREETYPE ON)
endif ()

option(HB_HAVE_GOBJECT "Enable GObject Bindings" OFF)
if (HB_HAVE_GOBJECT)
  set(HB_HAVE_GLIB ON)
endif ()

option(HB_HAVE_INTROSPECTION "Enable building introspection (.gir/.typelib) files" OFF)
if (HB_HAVE_INTROSPECTION)
  set(HB_HAVE_GOBJECT ON)
  set(HB_HAVE_GLIB ON)
endif ()

include_directories(AFTER
  ${PROJECT_SOURCE_DIR}/src
  ${PROJECT_BINARY_DIR}/src
  )

add_definitions(-DHAVE_OT)
add_definitions(-DHAVE_FALLBACK)

if (BUILD_SHARED_LIBS OR OSX_FRAMEWORK)
  add_definitions(-DHAVE_ATEXIT)
endif ()

if (MSVC)
  add_definitions(-wd4244 -wd4267 -D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_WARNINGS)
endif ()

if (WIN32 AND NOT MINGW AND BUILD_SHARED_LIBS)
  add_definitions("-DHB_EXTERN=__declspec(dllexport) extern")
endif ()


## Detect if we are running inside a distribution or regular repository folder
set(IN_HB_DIST FALSE)
if (EXISTS "${PROJECT_SOURCE_DIR}/ChangeLog")
  # perhaps we are on dist directory
  set(IN_HB_DIST TRUE)
  set(HB_VERSION_H "${PROJECT_SOURCE_DIR}/src/hb-version.h")
endif ()


## Extract variables from Makefile files
# http://stackoverflow.com/a/27630120/1414809
function (prepend var prefix)
  set(listVar "")
  foreach (f ${ARGN})
    list(APPEND listVar "${prefix}${f}")
  endforeach ()
  set(${var} "${listVar}" PARENT_SCOPE)
endfunction ()

function (extract_make_variable variable file prefix)
  string(REGEX MATCH "${variable} = ([^$]+)\\$" temp ${file})
  string(REGEX MATCHALL "[^ \n\t\\]+" list ${CMAKE_MATCH_1})
  prepend(list ${prefix} ${list})
  set(${variable} ${list} PARENT_SCOPE)
endfunction ()

file(READ ${PROJECT_SOURCE_DIR}/src/Makefile.sources SRCSOURCES)
file(READ ${PROJECT_SOURCE_DIR}/util/Makefile.sources UTILSOURCES)
file(READ ${PROJECT_SOURCE_DIR}/src/hb-ucdn/Makefile.sources UCDNSOURCES)

extract_make_variable(HB_BASE_sources ${SRCSOURCES} "${PROJECT_SOURCE_DIR}/src/")
extract_make_variable(HB_BASE_headers ${SRCSOURCES} "${PROJECT_SOURCE_DIR}/src/")
extract_make_variable(HB_FALLBACK_sources ${SRCSOURCES} "${PROJECT_SOURCE_DIR}/src/")
extract_make_variable(HB_OT_sources ${SRCSOURCES} "${PROJECT_SOURCE_DIR}/src/")
extract_make_variable(HB_OT_headers ${SRCSOURCES} "${PROJECT_SOURCE_DIR}/src/")

if (IN_HB_DIST)
  set(RAGEL_GENERATED_DIR "${PROJECT_SOURCE_DIR}/src/")
else ()
  set(RAGEL_GENERATED_DIR "${PROJECT_BINARY_DIR}/src/")
endif ()
extract_make_variable(HB_BASE_RAGEL_GENERATED_sources ${SRCSOURCES} ${RAGEL_GENERATED_DIR})
extract_make_variable(HB_OT_RAGEL_GENERATED_sources ${SRCSOURCES} ${RAGEL_GENERATED_DIR})

extract_make_variable(HB_VIEW_sources ${UTILSOURCES} "${PROJECT_SOURCE_DIR}/util/")
extract_make_variable(HB_SHAPE_sources ${UTILSOURCES} "${PROJECT_SOURCE_DIR}/util/")
extract_make_variable(HB_OT_SHAPE_CLOSURE_sources ${UTILSOURCES} "${PROJECT_SOURCE_DIR}/util/")

extract_make_variable(LIBHB_UCDN_sources ${UCDNSOURCES} "${PROJECT_SOURCE_DIR}/src/hb-ucdn/")

# file(READ configure.ac CONFIGUREAC)
# string(REGEX MATCH "\\[(([0-9]+)\\.([0-9]+)\\.([0-9]+))\\]" HB_VERSION_MATCH ${CONFIGUREAC})
# set(HB_VERSION ${CMAKE_MATCH_1})
# set(HB_VERSION_MAJOR ${CMAKE_MATCH_2})
# set(HB_VERSION_MINOR ${CMAKE_MATCH_3})
# set(HB_VERSION_MICRO ${CMAKE_MATCH_4})
include(util)
check_version(VERSION_MAJOR VERSION_MINOR VERSION_PATCH)
set(VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})
set(HB_VERSION ${VERSION})
set(PROJECT_VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${VERSION_MAJOR}")
endif()
report_version(${PROJECT_NAME} ${VERSION})

## Define ragel tasks
if (NOT IN_HB_DIST)
  find_program(RAGEL "ragel" CMAKE_FIND_ROOT_PATH_BOTH)

  if (RAGEL)
    message(STATUS "ragel found at: ${RAGEL}")
  else ()
    message(FATAL_ERROR "ragel not found, get it here -- http://www.complang.org/ragel/ or, use harfbuzz releases https://github.com/behdad/harfbuzz/releases")
  endif ()

  foreach (ragel_output IN ITEMS ${HB_BASE_RAGEL_GENERATED_sources} ${HB_OT_RAGEL_GENERATED_sources})
    string(REGEX MATCH "([^/]+)\\.hh" temp ${ragel_output})
    set(target_name ${CMAKE_MATCH_1})
    add_custom_command(OUTPUT ${ragel_output}
      COMMAND ${RAGEL} -G2 -o ${ragel_output} ${PROJECT_SOURCE_DIR}/src/${target_name}.rl -I ${PROJECT_SOURCE_DIR} ${ARGN}
      DEPENDS ${PROJECT_SOURCE_DIR}/src/${target_name}.rl
      )
    add_custom_target(harfbuzz_${target_name} DEPENDS ${PROJECT_BINARY_DIR}/src/${target_name})
  endforeach ()

  mark_as_advanced(RAGEL)
endif ()


## Generate hb-version.h
if (NOT IN_HB_DIST)
  set(HB_VERSION_H_IN "${PROJECT_SOURCE_DIR}/src/hb-version.h.in")
  set(HB_VERSION_H "${PROJECT_BINARY_DIR}/src/hb-version.h")
  set_source_files_properties("${HB_VERSION_H}" PROPERTIES GENERATED true)
  configure_file("${HB_VERSION_H_IN}" "${HB_VERSION_H}.tmp" @ONLY)
  execute_process(COMMAND "${CMAKE_COMMAND}" -E copy_if_different
    "${HB_VERSION_H}.tmp"
    "${HB_VERSION_H}"
    )
  file(REMOVE "${HB_VERSION_H}.tmp")
endif ()


## Define sources and headers of the project
set(project_sources
  ${HB_BASE_sources}
  ${HB_BASE_RAGEL_GENERATED_sources}

  ${HB_FALLBACK_sources}
  ${HB_OT_sources}
  ${HB_OT_RAGEL_GENERATED_sources}
  )

set(project_extra_sources)

set(project_headers
  ${HB_VERSION_H}

  ${HB_BASE_headers}
  ${HB_OT_headers}
  )


## Find and include needed header folders and libraries
if (HB_HAVE_FREETYPE)
    include(FindAnyProject)
    find_anyproject(Freetype DEFAULT ON)
  # include(FindFreetype)
  # if (NOT FREETYPE_FOUND)
  #   message(FATAL_ERROR "HB_HAVE_FREETYPE was set, but we failed to find it. Maybe add a CMAKE_PREFIX_PATH= to your Freetype2 install prefix")
  # endif()
    if(FREETYPE_FOUND)
      list(APPEND THIRD_PARTY_LIBS ${FREETYPE_LIBRARIES})
      include_directories(AFTER ${FREETYPE_INCLUDE_DIRS} ${FREETYPE_INCLUDE_DIRS}/freetype2)
      add_definitions(-DHAVE_FREETYPE=1 -DHAVE_FT_FACE_GETCHARVARIANTINDEX=1)

      list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-ft.cc)
      list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-ft.h)
    endif()
endif ()

if (HB_HAVE_GRAPHITE2)
  add_definitions(-DHAVE_GRAPHITE2)

  find_path(GRAPHITE2_INCLUDE_DIR graphite2/Font.h)
  find_library(GRAPHITE2_LIBRARY graphite2)

  include_directories(${GRAPHITE2_INCLUDE_DIR})

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-graphite2.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-graphite2.h)

  list(APPEND THIRD_PARTY_LIBS ${GRAPHITE2_LIBRARY})

  mark_as_advanced(GRAPHITE2_INCLUDE_DIR GRAPHITE2_LIBRARY)
endif ()

if (HB_BUILTIN_UCDN)
  include_directories(src/hb-ucdn)
  add_definitions(-DHAVE_UCDN)

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-ucdn.cc)
  list(APPEND project_extra_sources ${LIBHB_UCDN_sources})
endif ()

if (HB_HAVE_GLIB)
  add_definitions(-DHAVE_GLIB)

  # https://github.com/WebKit/webkit/blob/master/Source/cmake/FindGLIB.cmake
  find_package(PkgConfig)
  pkg_check_modules(PC_GLIB QUIET glib-2.0)

  find_library(GLIB_LIBRARIES NAMES glib-2.0 HINTS ${PC_GLIB_LIBDIR} ${PC_GLIB_LIBRARY_DIRS})
  find_path(GLIBCONFIG_INCLUDE_DIR NAMES glibconfig.h HINTS ${PC_LIBDIR} ${PC_LIBRARY_DIRS} ${PC_GLIB_INCLUDEDIR} ${PC_GLIB_INCLUDE_DIRS} PATH_SUFFIXES glib-2.0/include)
  find_path(GLIB_INCLUDE_DIR NAMES glib.h HINTS ${PC_GLIB_INCLUDEDIR} ${PC_GLIB_INCLUDE_DIRS} PATH_SUFFIXES glib-2.0)

  include_directories(${GLIBCONFIG_INCLUDE_DIR} ${GLIB_INCLUDE_DIR})

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-glib.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-glib.h)

  list(APPEND THIRD_PARTY_LIBS ${GLIB_LIBRARIES})

  mark_as_advanced(GLIB_LIBRARIES GLIBCONFIG_INCLUDE_DIR GLIB_INCLUDE_DIR)
endif ()

if (HB_HAVE_ICU)
  add_definitions(-DHAVE_ICU)

  # https://github.com/WebKit/webkit/blob/master/Source/cmake/FindICU.cmake
  find_package(PkgConfig)
  pkg_check_modules(PC_ICU QUIET icu-uc)

  find_path(ICU_INCLUDE_DIR NAMES unicode/utypes.h HINTS ${PC_ICU_INCLUDE_DIRS} ${PC_ICU_INCLUDEDIR})
  find_library(ICU_LIBRARY NAMES libicuuc cygicuuc cygicuuc32 icuuc HINTS ${PC_ICU_LIBRARY_DIRS} ${PC_ICU_LIBDIR})

  include_directories(${ICU_INCLUDE_DIR})

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-icu.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-icu.h)

  list(APPEND THIRD_PARTY_LIBS ${ICU_LIBRARY})

  mark_as_advanced(ICU_INCLUDE_DIR ICU_LIBRARY)
endif ()

if (APPLE AND HB_HAVE_CORETEXT)
  # Apple Advanced Typography
  add_definitions(-DHAVE_CORETEXT)

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-coretext.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-coretext.h)

  find_library(APPLICATION_SERVICES_FRAMEWORK ApplicationServices)
  if (APPLICATION_SERVICES_FRAMEWORK)
    list(APPEND THIRD_PARTY_LIBS ${APPLICATION_SERVICES_FRAMEWORK})
  endif (APPLICATION_SERVICES_FRAMEWORK)

  mark_as_advanced(APPLICATION_SERVICES_FRAMEWORK)
endif ()

if (WIN32 AND HB_HAVE_UNISCRIBE)
  add_definitions(-DHAVE_UNISCRIBE)

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-uniscribe.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-uniscribe.h)

  list(APPEND THIRD_PARTY_LIBS usp10 gdi32 rpcrt4)
endif ()

if (WIN32 AND HB_HAVE_DIRECTWRITE)
  add_definitions(-DHAVE_DIRECTWRITE)

  list(APPEND project_sources ${PROJECT_SOURCE_DIR}/src/hb-directwrite.cc)
  list(APPEND project_headers ${PROJECT_SOURCE_DIR}/src/hb-directwrite.h)

  list(APPEND THIRD_PARTY_LIBS dwrite rpcrt4)
endif ()

if (HB_HAVE_GOBJECT)
  include(FindPythonInterp)
  include(FindPerl)

  # Use the hints from glib-2.0.pc to find glib-mkenums
  find_package(PkgConfig)
  pkg_check_modules(PC_GLIB QUIET glib-2.0)
  find_program(GLIB_MKENUMS glib-mkenums
    HINTS ${PC_glib_mkenums}
    )
  set(GLIB_MKENUMS_CMD)

  if (WIN32 AND NOT MINGW)
    # In Visual Studio builds, shebang lines are not supported
    # in the standard cmd.exe shell that we use, so we need to
    # first determine whether glib-mkenums is a Python or PERL
    # script
    execute_process(COMMAND "${PYTHON_EXECUTABLE}" "${GLIB_MKENUMS}" --version
      RESULT_VARIABLE GLIB_MKENUMS_PYTHON
      OUTPUT_QUIET ERROR_QUIET
      )
    if (GLIB_MKENUMS_PYTHON EQUAL 0)
      message("${GLIB_MKENUMS} is a Python script.")
      set(GLIB_MKENUMS_CMD "${PYTHON_EXECUTABLE}" "${GLIB_MKENUMS}")
    else ()
      execute_process(COMMAND "${PERL_EXECUTABLE}" "${GLIB_MKENUMS}" --version
        RESULT_VARIABLE GLIB_MKENUMS_PERL
        OUTPUT_QUIET ERROR_QUIET
        )
      if (GLIB_MKENUMS_PERL EQUAL 0)
        message("${GLIB_MKENUMS} is a PERL script.")
        set(GLIB_MKENUMS_CMD "${PERL_EXECUTABLE}" "${GLIB_MKENUMS}")
      endif ()
      if (NOT GLIB_MKENUMS_PERL EQUAL 0 AND NOT GLIB_MKENUMS_PYTHON EQUAL 0)
        message(FATAL_ERROR "Unable to determine type of glib-mkenums script")
      endif ()
	endif ()
  else ()
    set(GLIB_MKENUMS_CMD "${GLIB_MKENUMS}")
  endif ()
  if (NOT GLIB_MKENUMS_CMD)
    message(FATAL_ERROR "HB_HAVE_GOBJECT was set, but we failed to find glib-mkenums, which is required")
  endif()

  pkg_check_modules(PC_GOBJECT QUIET gobject-2.0)

  find_library(GOBJECT_LIBRARIES NAMES gobject-2.0 HINTS ${PC_GLIB_LIBDIR} ${PC_GLIB_LIBRARY_DIRS})
  find_path(GOBJECT_INCLUDE_DIR NAMES glib-object.h HINTS ${PC_GLIB_INCLUDEDIR} ${PC_GLIB_INCLUDE_DIRS} PATH_SUFFIXES glib-2.0)

  include_directories(${GOBJECTCONFIG_INCLUDE_DIR} ${GOBJECT_INCLUDE_DIR})
  mark_as_advanced(GOBJECT_LIBRARIES GOBJECT_INCLUDE_DIR)

  list(APPEND hb_gobject_sources ${PROJECT_SOURCE_DIR}/src/hb-gobject-structs.cc)
  list(APPEND hb_gobject_gen_sources
    ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.cc
    )
  list(APPEND hb_gobject_structs_headers
    ${PROJECT_SOURCE_DIR}/src/hb-gobject-structs.h
    )
  list(APPEND hb_gobject_headers
    ${PROJECT_SOURCE_DIR}/src/hb-gobject.h
    ${hb_gobject_structs_headers}
    )
  list(APPEND hb_gobject_gen_headers
    ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h
    )

  add_custom_command (
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h
    COMMAND ${GLIB_MKENUMS_CMD}
      --template=${PROJECT_SOURCE_DIR}/src/hb-gobject-enums.h.tmpl
      --identifier-prefix hb_
      --symbol-prefix hb_gobject
      ${hb_gobject_structs_headers}
      ${project_headers}
      > ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h.tmp
    COMMAND "${CMAKE_COMMAND}"
      "-DENUM_INPUT_SRC=${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h.tmp"
      "-DENUM_OUTPUT_SRC=${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h"
      -P ${PROJECT_SOURCE_DIR}/replace-enum-strings.cmake
    DEPENDS ${PROJECT_SOURCE_DIR}/src/hb-gobject-enums.h.tmpl
      ${hb_gobject_header}
      ${project_headers}
    )

  add_custom_command (
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.cc
    COMMAND ${GLIB_MKENUMS_CMD}
      --template=${PROJECT_SOURCE_DIR}/src/hb-gobject-enums.cc.tmpl
      --identifier-prefix hb_
      --symbol-prefix hb_gobject
      ${hb_gobject_header}
      ${project_headers}
      > ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.cc.tmp
    COMMAND "${CMAKE_COMMAND}"
      "-DENUM_INPUT_SRC=${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.cc.tmp"
      "-DENUM_OUTPUT_SRC=${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.cc"
      -P ${PROJECT_SOURCE_DIR}/replace-enum-strings.cmake
    DEPENDS ${PROJECT_SOURCE_DIR}/src/hb-gobject-enums.cc.tmpl
      ${CMAKE_CURRENT_BINARY_DIR}/src/hb-gobject-enums.h
      ${hb_gobject_header}
      ${project_headers}
    )
endif ()

## Atomic ops availability detection
file(WRITE "${PROJECT_BINARY_DIR}/try_compile_intel_atomic_primitives.c"
"		void memory_barrier (void) { __sync_synchronize (); }
		int atomic_add (int *i) { return __sync_fetch_and_add (i, 1); }
		int mutex_trylock (int *m) { return __sync_lock_test_and_set (m, 1); }
		void mutex_unlock (int *m) { __sync_lock_release (m); }
		int main () { return 0; }
")
try_compile(HB_HAVE_INTEL_ATOMIC_PRIMITIVES
  ${PROJECT_BINARY_DIR}/try_compile_intel_atomic_primitives
  SOURCES ${PROJECT_BINARY_DIR}/try_compile_intel_atomic_primitives.c)
if (HB_HAVE_INTEL_ATOMIC_PRIMITIVES)
  add_definitions(-DHAVE_INTEL_ATOMIC_PRIMITIVES)
endif ()

file(WRITE "${PROJECT_BINARY_DIR}/try_compile_solaris_atomic_ops.c"
"		#include <atomic.h>
		/* This requires Solaris Studio 12.2 or newer: */
		#include <mbarrier.h>
		void memory_barrier (void) { __machine_rw_barrier (); }
		int atomic_add (volatile unsigned *i) { return atomic_add_int_nv (i, 1); }
		void *atomic_ptr_cmpxchg (volatile void **target, void *cmp, void *newval) { return atomic_cas_ptr (target, cmp, newval); }
		int main () { return 0; }
")
try_compile(HB_HAVE_SOLARIS_ATOMIC_OPS
  ${PROJECT_BINARY_DIR}/try_compile_solaris_atomic_ops
  SOURCES ${PROJECT_BINARY_DIR}/try_compile_solaris_atomic_ops.c)
if (HB_HAVE_SOLARIS_ATOMIC_OPS)
  add_definitions(-DHAVE_SOLARIS_ATOMIC_OPS)
endif ()

set(INSTALL_TARGETS harfbuzz)

## Define harfbuzz library
if(OSX_FRAMEWORK)
    add_library(harfbuzz SHARED ${project_sources} ${project_extra_sources} ${project_headers})
else()
    add_library(harfbuzz ${project_sources} ${project_extra_sources} ${project_headers})
endif()

target_link_libraries(harfbuzz ${THIRD_PARTY_LIBS})

## Define harfbuzz-gobject library
if (HB_HAVE_GOBJECT)
    set(INSTALL_TARGETS ${INSTALL_TARGETS} harfbuzz-gobject)
  add_library(harfbuzz-gobject
    ${hb_gobject_sources}
    ${hb_gobject_gen_sources}
    ${hb_gobject_headers}
    ${hb_gobject_gen_headers}
    )
  include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR}/src)
  add_dependencies(harfbuzz-gobject harfbuzz)
  target_link_libraries(harfbuzz-gobject harfbuzz ${GOBJECT_LIBRARIES} ${THIRD_PARTY_LIBS})
endif ()

# On Windows, g-ir-scanner requires a DLL build in order for it to work
if (WIN32)
  if (NOT BUILD_SHARED_LIBS)
    message("Building introspection files on Windows requires BUILD_SHARED_LIBS to be enabled.")
    set(HB_HAVE_INTROSPECTION OFF)
  endif ()
endif ()

if (HB_HAVE_INTROSPECTION)

  find_package(PkgConfig)
  pkg_check_modules(PC_GI QUIET gobject-introspection-1.0)

  find_program(G_IR_SCANNER g-ir-scanner
    HINTS ${PC_g_ir_scanner}
    )

  find_program(G_IR_COMPILER g-ir-compiler
    HINTS ${PC_g_ir_compiler}
    )

  if (WIN32 AND NOT MINGW)
    # Note that since we already enable HB_HAVE_GOBJECT
    # we would already have PYTHON_EXECUTABLE handy
    set(G_IR_SCANNER_CMD "${PYTHON_EXECUTABLE}" "${G_IR_SCANNER}")
  else ()
    set(G_IR_SCANNER_CMD "${G_IR_SCANNER}")
  endif ()

  # We need to account for the varying output directories
  # when we build using Visual Studio projects
  if("${CMAKE_GENERATOR}" MATCHES "Visual Studio*")
    set (hb_libpath "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>")
  else ()
    set (hb_libpath "$<TARGET_FILE_DIR:harfbuzz-gobject>")
  endif ()

  # Get the CFlags that we used to build HarfBuzz/HarfBuzz-GObject
  set (hb_defines_cflags "")
  foreach(hb_cflag ${hb_cflags})
    list(APPEND hb_defines_cflags "-D${hb_cflag}")
  endforeach(hb_cflag)

  # Get the other dependent libraries we used to build HarfBuzz/HarfBuzz-GObject
  set (extra_libs "")
  foreach (extra_lib ${THIRD_PARTY_LIBS})
    # We don't want the .lib extension here...
    string(REPLACE ".lib" "" extra_lib_stripped "${extra_lib}")
    list(APPEND extra_libs "--extra-library=${extra_lib_stripped}")
  endforeach ()

  set(introspected_sources)
  foreach (f
    ${project_headers}
    ${project_sources}
    ${hb_gobject_gen_sources}
    ${hb_gobject_gen_headers}
    ${hb_gobject_sources}
    ${hb_gobject_headers}
    )
    if (WIN32)
      # Nasty issue: We need to make drive letters lower case,
      # otherwise g-ir-scanner won't like it and give us a bunch
      # of invalid items and unresolved types...
      STRING(SUBSTRING "${f}" 0 1 drive)
      STRING(SUBSTRING "${f}" 1 -1 path)
      if (drive MATCHES "[A-Z]")
        STRING(TOLOWER ${drive} drive_lower)
        list(APPEND introspected_sources "${drive_lower}${path}")
      else ()
        list(APPEND introspected_sources "${f}")
      endif ()
    else ()
      list(APPEND introspected_sources "${f}")
    endif ()
  endforeach ()

  # Finally, build the introspection files...
  add_custom_command (
    TARGET harfbuzz-gobject
    POST_BUILD
    COMMAND ${G_IR_SCANNER_CMD}
      --warn-all --no-libtool --verbose
      -n hb
      --namespace=HarfBuzz
      --nsversion=0.0
      --identifier-prefix=hb_
      --include GObject-2.0
      --pkg-export=harfbuzz
      --cflags-begin
      -I${PROJECT_SOURCE_DIR}/src
      -I${PROJECT_BINARY_DIR}/src
      ${hb_includedir_cflags}
      ${hb_defines_cflags}
      -DHB_H
      -DHB_H_IN
      -DHB_OT_H
      -DHB_OT_H_IN
      -DHB_GOBJECT_H
      -DHB_GOBJECT_H_IN
      -DHB_EXTERN=
      --cflags-end
      --library=harfbuzz-gobject
      --library=harfbuzz
      -L${hb_libpath}
      ${extra_libs}
      ${introspected_sources}
      -o ${hb_libpath}/HarfBuzz-0.0.gir
    DEPENDS harfbuzz-gobject harfbuzz
    )

  add_custom_command (
    TARGET harfbuzz-gobject
    POST_BUILD
    COMMAND "${G_IR_COMPILER}"
      --verbose --debug
      --includedir ${CMAKE_CURRENT_BINARY_DIR}
      ${hb_libpath}/HarfBuzz-0.0.gir
      -o ${hb_libpath}/HarfBuzz-0.0.typelib
    DEPENDS ${hb_libpath}/HarfBuzz-0.0.gir harfbuzz-gobject
    )
endif ()

## Additional framework build configs
if (BUILD_FRAMEWORK)
  set_target_properties(harfbuzz PROPERTIES
    FRAMEWORK TRUE
    PUBLIC_HEADER "${project_headers}"
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
    MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
    MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
    MACOSX_FRAMEWORK_IDENTIFIER "harfbuzz"
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    VERSION ${VERSION}
    SOVERSION ${FRAMEWORK_VERSION}
  )
endif ()


## Additional harfbuzz build artifacts
if (HB_BUILD_UTILS)
  # https://github.com/WebKit/webkit/blob/master/Source/cmake/FindCairo.cmake
  find_package(PkgConfig)
  pkg_check_modules(PC_CAIRO QUIET cairo)

  find_path(CAIRO_INCLUDE_DIRS NAMES cairo.h HINTS ${PC_CAIRO_INCLUDEDIR} ${PC_CAIRO_INCLUDE_DIRS} PATH_SUFFIXES cairo)
  find_library(CAIRO_LIBRARIESNAMES cairo HINTS ${PC_CAIRO_LIBDIR} ${PC_CAIRO_LIBRARY_DIRS})

  add_definitions("-DPACKAGE_NAME=\"HarfBuzz\"")
  add_definitions("-DPACKAGE_VERSION=\"${HB_VERSION}\"")
  include_directories(${CAIRO_INCLUDE_DIRS})

  add_executable(hb-view ${HB_VIEW_sources})
  target_link_libraries(hb-view harfbuzz ${CAIRO_LIBRARIESNAMES})

  add_executable(hb-shape ${HB_SHAPE_sources})
  target_link_libraries(hb-shape harfbuzz)

  add_executable(hb-ot-shape-closure ${HB_OT_SHAPE_CLOSURE_sources})
  target_link_libraries(hb-ot-shape-closure harfbuzz)

  set(INSTALL_TARGETS ${INSTALL_TARGETS} hb-view hb-shape hb-ot-shape-closure)

  mark_as_advanced(CAIRO_INCLUDE_DIRS CAIRO_LIBRARIESNAMES)
endif ()


## Install
if (NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
  install(FILES ${project_headers} DESTINATION ${INSTALL_INC_DIR}/harfbuzz)
  if (HB_HAVE_GOBJECT)
    install(FILES ${hb_gobject_headers} ${hb_gobject_gen_headers} DESTINATION ${INSTALL_INC_DIR}/harfbuzz)
  endif ()
endif ()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

if (NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
  # install(TARGETS harfbuzz
  #   ARCHIVE DESTINATION lib
  #   LIBRARY DESTINATION lib
  #   RUNTIME DESTINATION bin
  #   FRAMEWORK DESTINATION Library/Frameworks
  #   )
  # if (HB_BUILD_UTILS)
  #   install(TARGETS hb-view
  #     RUNTIME DESTINATION bin
  #   )
  #   install(TARGETS hb-view
  #     RUNTIME DESTINATION bin
  #   )
  #
  #   install(TARGETS hb-shape
  #     RUNTIME DESTINATION bin
  #   )
  #
  #   install(TARGETS hb-ot-shape-closure
  #     RUNTIME DESTINATION bin
  #   )
  # endif ()
  # if (HB_HAVE_GOBJECT)
  #   install(TARGETS harfbuzz-gobject
  #     ARCHIVE DESTINATION lib
  #     LIBRARY DESTINATION lib
  #     RUNTIME DESTINATION bin
  #   )
  #   if (HB_HAVE_INTROSPECTION)
  #     if("${CMAKE_GENERATOR}" MATCHES "Visual Studio*")
  #       set (hb_libpath "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>")
  #     else ()
  #       set (hb_libpath "$<TARGET_FILE_DIR:harfbuzz-gobject>")
  #     endif ()
  #
  #     install(FILES "${hb_libpath}/HarfBuzz-0.0.gir"
  #       DESTINATION share/gir-1.0
  #       )
  #
  #     install(FILES "${hb_libpath}/HarfBuzz-0.0.typelib"
  #       DESTINATION lib/girepository-1.0
  #       )
  #   endif ()
  # endif ()

  install(TARGETS ${INSTALL_TARGETS}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
    )
endif ()

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(harfbuzz PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

# Add all targets to the build-tree export set
export(TARGETS harfbuzz
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the ZLIBConfig.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
