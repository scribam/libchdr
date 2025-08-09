cmake_minimum_required(VERSION 3.10)

project(chdr VERSION 0.2 LANGUAGES C)

if(CMAKE_PROJECT_NAME STREQUAL "chdr")
  option(BUILD_SHARED_LIBS "Build libchdr also as a shared library" ON)
endif()
option(INSTALL_STATIC_LIBS "Install static libraries" OFF)
option(WITH_SYSTEM_ZLIB "Use system provided zlib library" OFF)
option(WITH_SYSTEM_ZSTD "Use system provided zstd library" OFF)

option(BUILD_LTO "Compile libchdr with link-time optimization if supported" OFF)
if(BUILD_LTO)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT HAVE_IPO)
  if(HAVE_IPO)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  endif()
endif()

option(BUILD_FUZZER "Build instrumented binary for fuzzing with libfuzzer, requires clang")
if(BUILD_FUZZER)
  # Override CFLAGS early for instrumentation. Disable shared libs for instrumentation.
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address,fuzzer-no-link")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,fuzzer-no-link")
  set(BUILD_SHARED_LIBS OFF)
endif()

include(GNUInstallDirs)

#--------------------------------------------------
# dependencies
#--------------------------------------------------

if(NOT TARGET lzma)
  add_library(lzma STATIC
    external/7zip/C/Alloc.c
    external/7zip/C/Bra.c
    external/7zip/C/Bra86.c
    external/7zip/C/BraIA64.c
    external/7zip/C/CpuArch.c
    external/7zip/C/Delta.c
    external/7zip/C/LzFind.c
    external/7zip/C/Lzma86Dec.c
    external/7zip/C/LzmaDec.c
    external/7zip/C/LzmaEnc.c
    external/7zip/C/Sort.c
  )
  target_compile_definitions(lzma PRIVATE Z7_ST)
  target_include_directories(lzma PUBLIC external/7zip/C)
  set_target_properties(lzma PROPERTIES POSITION_INDEPENDENT_CODE ON)

  option(WITH_LZMA_ASM "Use lzma asm" ON)
  if(WITH_LZMA_ASM)
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
      include(CheckSymbolExists)
      check_symbol_exists("__aarch64__" "" CPU_ARM64)
      if(CPU_ARM64)
        enable_language(ASM)
        set_source_files_properties(external/7zip/C/LzmaDec.c PROPERTIES COMPILE_DEFINITIONS Z7_LZMA_DEC_OPT)
        target_sources(lzma PRIVATE external/7zip/Asm/arm64/LzmaDecOpt.S)
        set_source_files_properties(external/external/7zip/Asm/arm64/LzmaDecOpt.S PROPERTIES LANGUAGE ASM)
      endif()
    elseif(WIN32)
      include(CheckSymbolExists)
      check_symbol_exists("_M_AMD64" "" CPU_X64)
      if(CPU_X64)
        enable_language(ASM_MASM)
        set_source_files_properties(external/7zip/C/LzmaDec.c PROPERTIES COMPILE_DEFINITIONS Z7_LZMA_DEC_OPT)
        target_sources(lzma PRIVATE external/7zip/Asm/x86/LzmaDecOpt.asm)
        set_source_files_properties(external/7zip/Asm/x86/LzmaDecOpt.asm PROPERTIES LANGUAGE ASM_MASM)
      endif()
    endif()
  endif()
endif()
list(APPEND CHDR_LIBS lzma)

# zlib
if(WITH_SYSTEM_ZLIB)
  # From 3.24 there is a cmake option to find zlib static
  # but before, we need to do it in a more manual way
  if(CMAKE_VERSION VERSION_LESS 3.24.0)
    set(_CMAKE_FIND_LIBRARY_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES})
    set(CMAKE_FIND_LIBRARY_SUFFIXES "static.lib" ".a")
  else()
    set(ZLIB_USE_STATIC_LIBS ON)
  endif()

  find_package(ZLIB REQUIRED)

  # Revert library suffix to what they were
  if(CMAKE_VERSION VERSION_LESS 3.24.0)
    set(CMAKE_FIND_LIBRARY_SUFFIXES ${_CMAKE_FIND_LIBRARY_SUFFIXES})
    unset(_CMAKE_FIND_LIBRARY_SUFFIXES)
  endif()

  list(APPEND PLATFORM_LIBS ZLIB::ZLIB)
else()
  if(NOT TARGET zlibstatic)
    set(ZLIB_BUILD_TESTING OFF)
    set(ZLIB_BUILD_SHARED OFF)
    set(ZLIB_INSTALL OFF)
    add_subdirectory(external/zlib EXCLUDE_FROM_ALL)
    set_target_properties(zlibstatic PROPERTIES POSITION_INDEPENDENT_CODE ON)
  endif()
  list(APPEND CHDR_LIBS zlibstatic)
endif()

if(WITH_SYSTEM_ZSTD)
  find_package(zstd REQUIRED)
  if(TARGET zstd::libzstd_shared)
    list(APPEND PLATFORM_LIBS zstd::libzstd_shared)
  else()
    list(APPEND PLATFORM_LIBS zstd::libzstd_static)
  endif()
else()
  if(NOT TARGET libzstd_static)
    set(ZSTD_BUILD_SHARED "BUILD SHARED LIBRARIES" OFF)
    set(ZSTD_BUILD_COMPRESSION "BUILD COMPRESSION MODULE" OFF)
    set(ZSTD_BUILD_DICTBUILDER "BUILD DICTBUILDER MODULE" ON)
    set(ZSTD_BUILD_PROGRAMS "BUILD PROGRAMS" OFF)
    set(ZSTD_LEGACY_SUPPORT "LEGACY SUPPORT" OFF)
    add_subdirectory(external/zstd/build/cmake EXCLUDE_FROM_ALL)
  endif()
  list(APPEND CHDR_LIBS libzstd_static)
endif()

#--------------------------------------------------
# chdr
#--------------------------------------------------

set(CHDR_SOURCES
  src/libchdr_bitstream.c
  src/libchdr_cdrom.c
  src/libchdr_chd.c
  src/libchdr_flac.c
  src/libchdr_huffman.c
)

add_library(chdr-static STATIC ${CHDR_SOURCES})
target_include_directories(chdr-static PUBLIC external include)
target_link_libraries(chdr-static PRIVATE ${CHDR_LIBS} ${PLATFORM_LIBS})

if(MSVC)
  target_compile_definitions(chdr-static PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

if(INSTALL_STATIC_LIBS)
  install(TARGETS chdr-static ${CHDR_LIBS}
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  )
endif()

if(BUILD_SHARED_LIBS)
  add_library(chdr SHARED ${CHDR_SOURCES})
  target_include_directories(chdr PUBLIC external include)
  target_link_libraries(chdr PRIVATE ${CHDR_LIBS} ${PLATFORM_LIBS})

  if(MSVC)
    target_compile_definitions(chdr PUBLIC "CHD_DLL")
    target_compile_definitions(chdr PRIVATE "CHD_DLL_EXPORTS")
    target_compile_definitions(chdr PRIVATE _CRT_SECURE_NO_WARNINGS)
  elseif(APPLE)
    target_link_libraries(chdr PRIVATE -Wl,-dead_strip -Wl,-exported_symbol,_chd_*)
  elseif(CMAKE_SYSTEM_NAME STREQUAL OpenBSD)
    target_link_libraries(chdr PRIVATE -Wl,--version-script ${CMAKE_CURRENT_SOURCE_DIR}/src/link.T)
  else()
    target_link_libraries(chdr PRIVATE -Wl,--version-script ${CMAKE_CURRENT_SOURCE_DIR}/src/link.T -Wl,--no-undefined)
  endif()

  set_target_properties(chdr PROPERTIES C_VISIBILITY_PRESET hidden)
  set_target_properties(chdr PROPERTIES VISIBILITY_INLINES_HIDDEN 1)
  set_target_properties(chdr PROPERTIES PUBLIC_HEADER "include/libchdr/bitstream.h;include/libchdr/cdrom.h;include/libchdr/chd.h;include/libchdr/chdconfig.h;include/libchdr/coretypes.h;include/libchdr/flac.h;include/libchdr/huffman.h")
  set_target_properties(chdr PROPERTIES VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}" SOVERSION ${PROJECT_VERSION_MAJOR})

  install(TARGETS chdr
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/libchdr"
  )

  configure_file(pkg-config.pc.in ${CMAKE_BINARY_DIR}/libchdr.pc @ONLY)
  install(FILES ${CMAKE_BINARY_DIR}/libchdr.pc DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
endif()

add_subdirectory(tests)
