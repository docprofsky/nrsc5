cmake_minimum_required (VERSION 2.8)
include (CheckLibraryExists)
include (CheckSymbolExists)
include (ExternalProject)
project (nrsc5 C)

option (USE_COLOR "Colorize log output")
option (USE_NEON "Use NEON instructions")
option (USE_SSE "Use SSE3 instructions")
option (USE_THREADS "Enable multithreading" ON)
option (USE_FAST_MATH "Use unsafe math optimizations")
option (USE_FAAD2 "AAC decoding with FAAD2" ON)
option (USE_ID3V2LIB "ID3V2 decoding from PSD" ON)

find_program (AUTOCONF autoconf)
if (NOT AUTOCONF)
    message (FATAL_ERROR "Missing autoconf. Install autoconf package and try again.")
endif ()

find_program (AUTOMAKE automake)
if (NOT AUTOMAKE)
    message (FATAL_ERROR "Missing automake. Install autoconf package and try again.")
endif ()

find_program (LIBTOOLIZE NAMES libtoolize glibtoolize)
if (NOT LIBTOOLIZE)
    message (FATAL_ERROR "Missing libtoolize. Install libtool package and try again.")
endif ()

find_program (PATCH patch)
if (NOT PATCH)
    message (FATAL_ERROR "Missing patch. Install patch package and try again.")
endif()

find_library (FFTW3F_LIBRARY fftw3f)
find_library (RTL_SDR_LIBRARY rtlsdr)

if (CMAKE_SYSTEM_PROCESSOR MATCHES "arm.*")
    if (USE_NEON)
        set (CMAKE_C_FLAGS "-mcpu=cortex-a7 -mfloat-abi=hard -mfpu=neon-vfpv4")
        add_definitions (-DHAVE_NEON)
    endif()
endif()

if (CMAKE_SYSTEM_PROCESSOR MATCHES "(i[456]|x)86.*")
    if (USE_SSE)
        set (CMAKE_C_FLAGS "-msse2 -msse3 -mssse3")
        add_definitions (-DHAVE_SSE2 -DHAVE_SSE3)
    endif()
endif()

set (CMAKE_REQUIRED_FLAGS --std=gnu11)
check_symbol_exists (strndup string.h HAVE_STRNDUP)
check_symbol_exists (CMPLXF complex.h HAVE_CMPLXF)
check_symbol_exists (_Imaginary_I complex.h HAVE_IMAGINARY_I)
check_symbol_exists (_Complex_I complex.h HAVE_COMPLEX_I)

if (USE_FAAD2)
    # libao only used if we have FAAD2
    find_library (AO_LIBRARY ao)

    set (FAAD2_PREFIX "${CMAKE_BINARY_DIR}/faad2-prefix")
    ExternalProject_Add (
        faad2_external
        GIT_REPOSITORY "https://github.com/dsvensson/faad2.git"
        GIT_TAG b7aa099fd3220b71180ed2b0bc19dc6209a1b418
        PREFIX ${FAAD2_PREFIX}

        UPDATE_COMMAND ""
        PATCH_COMMAND patch -p1 -Ni "${CMAKE_SOURCE_DIR}/support/faad2-hdc-support.patch" || exit 0
        COMMAND sh ./bootstrap

        CONFIGURE_COMMAND ${FAAD2_PREFIX}/src/faad2_external/configure --prefix=${FAAD2_PREFIX} --with-hdc "CFLAGS=-O3 ${CMAKE_C_FLAGS}"

        BUILD_COMMAND make
    )

    add_library (faad2 STATIC IMPORTED)
    set_property (TARGET faad2 PROPERTY IMPORTED_LOCATION "${FAAD2_PREFIX}/lib/libfaad.a")
    add_dependencies (faad2 faad2_external)
    include_directories ("${FAAD2_PREFIX}/include")

    set (FAAD2_LIBRARY faad2)
    add_definitions (-DHAVE_FAAD2)
endif()

if (USE_ID3V2LIB)
    set (ID3V2LIB_PREFIX "${CMAKE_BINARY_DIR}/id3v2lib-prefix")
    ExternalProject_Add (
        id3v2lib_external
        GIT_REPOSITORY "https://github.com/larsbs/id3v2lib.git"
        GIT_TAG 9bca1c33b8cd8037ee59de927bda057c9ac7043d
        PREFIX ${ID3V2LIB_PREFIX}

        INSTALL_COMMAND ""
    )

    add_library (id3v2lib STATIC IMPORTED)
    set_property (TARGET id3v2lib PROPERTY IMPORTED_LOCATION "${ID3V2LIB_PREFIX}/src/id3v2lib_external-build/src/libid3v2.a")
    add_dependencies (id3v2lib id3v2lib_external)
    include_directories ("${ID3V2LIB_PREFIX}/src/id3v2lib_external/include")

    set (ID3V2LIB_LIBRARY id3v2lib)
    add_definitions (-DHAVE_ID3V2LIB)
endif()

add_subdirectory (src)
