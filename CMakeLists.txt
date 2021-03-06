
cmake_minimum_required( VERSION 3.11.4 )

# CMP0046 -- We need this policy because this project adds
#            dependencies on mpool, but has no rule to build mpool
#            (because it's not part of this project).  We should
#            fix that and eliminate this policy setting.
cmake_policy( SET CMP0046 OLD )

project( hse )

set( PROJECT_VERSION            "${BUILD_PKG_VERSION}" )
set( PROJECT_HOMEPAGE_URL       "https://github.com/hse-project/hse" )

set( CMAKE_PROJECT_VERSION      "${PROJECT_VERSION}" )
set( CMAKE_PROJECT_HOMEPAGE_URL "${PROJECT_HOMEPAGE_URL}" )

message(STATUS "Configuring ${CMAKE_PROJECT_NAME}")

set (CMAKE_C_STANDARD 99)

set( HSE_SUB_REPOS      ${PROJECT_SOURCE_DIR}/sub)
set( HSE_SCRIPTS_DIR    ${PROJECT_SOURCE_DIR}/scripts)
set( HSE_STAGE_DIR      ${PROJECT_BINARY_DIR}/stage)

# CPACK_PACKAGING_INSTALL_PREFIX is used by CPack.
# CMAKE_INSTALL_PREFIX is used by:
#   - make install
#   - Cmake's "install()" function when its DESTINATION parameter
#     is a relative path.
set( CPACK_PACKAGING_INSTALL_PREFIX  /usr)
set( CMAKE_INSTALL_PREFIX            /usr)
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY  ${HSE_STAGE_DIR}/bin)

include(GNUInstallDirs)

# Install destinations
set( HSE_DIAG_BIN  /opt/hse/bin)
set( HSE_DOCS_DIR  share/doc/hse/)
set( HSE_UDOC_DIR  /opt/hse/doc)

set( HSE_UTIL_INCLUDE_DIRS
  ${PROJECT_SOURCE_DIR}/src/util/include
  ${CMAKE_BINARY_DIR}/stage/gensrc)

set( HSE_INCLUDE_DIRS ${PROJECT_SOURCE_DIR}/include)

set( HSE_UT_FMWK_INCLUDE_DIRS   ${PROJECT_SOURCE_DIR}/test/framework/include)
set( HSE_TEST_LIB_INCLUDE_DIRS    ${PROJECT_SOURCE_DIR}/test/library/include)
set( HSE_3RD_PARTY_INCLUDE_DIRS  ${PROJECT_SOURCE_DIR}/3rdparty/include)

include_directories(${PROJECT_SOURCE_DIR}/3rdparty/include)
include_directories(${HSE_STAGE_DIR}/include)

# Translate "-DFOO -DBAR" to "-DFOO;-DBAR" for add_definitions()
#
separate_arguments( BUILD_CDEFS_SEPARATED UNIX_COMMAND ${BUILD_CDEFS} )
add_definitions( ${BUILD_CDEFS_SEPARATED} )

add_compile_options(
    ${BUILD_CFLAGS} -g3 -std=gnu99 -Wall -Werror
    -Wlogical-op -Wno-missing-field-initializers -Wuninitialized -Wmaybe-uninitialized
    -Wextra -Wno-conversion -Wno-sign-conversion -Wno-sign-compare -Wno-unused-parameter
    )

# TODO: Remove and fix compile errors in various rest functions...
#
if( "${BUILD_PKG_TYPE}" STREQUAL "deb" )
    add_compile_options( -Wno-unused-result )
endif()

if ( ${UBSAN} )
    message(STATUS "Enabling ubsan support")
    add_compile_options( -fno-omit-frame-pointer )
    add_compile_options( -fsanitize=undefined -fno-sanitize=alignment )
    link_libraries( -fsanitize=undefined -fno-sanitize=alignment )
endif()

if ( ${ASAN} )
    message(STATUS "Enabling asan/lsan support")
    add_compile_options( -fno-omit-frame-pointer )
    add_compile_options( -fsanitize=address )
    link_libraries( -fsanitize=address )
endif()

# Define valgrind wrapper functions if valgrind-devel package
# is installed (sudo dnf install valgrind-devel).
find_path(ValgrindIncludes valgrind.h PATHS /usr/include/valgrind)
if(ValgrindIncludes)
    message(STATUS "Enabling valgrind support")
else()
    message(STATUS "Disabling valgrind support")
    add_definitions( -DNVALGRIND )
endif()

if( EXISTS /usr/libexec/platform-python )
    set( HSE_PYTHON /usr/libexec/platform-python )
elseif( EXISTS /etc/fedora-release )
    set( HSE_PYTHON /usr/bin/python3 )
else()
    set( HSE_PYTHON /usr/bin/python )
endif()

execute_process( COMMAND ${HSE_PYTHON} -c "from distutils.sysconfig import get_python_lib; print(get_python_lib())" OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE HSE_PYTHON_PKG_DIR )

get_filename_component( HSE_PYTHON_BASE_DIR ${HSE_PYTHON_PKG_DIR}/.. ABSOLUTE )
set ( HSE_PYTHON_SOS_DIR ${HSE_PYTHON_PKG_DIR}/sos )
set ( HSE_PYTHON_SOS_PLUGINS_DIR ${HSE_PYTHON_PKG_DIR}/sos/plugins )


include(CMakeLists.macros)


# Enable the following flags to try and catch "use before initialization"
# problems that gcc doesn't catch with default build flags.
#
#set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O1 -fno-tree-ccp")


#
# Flags to enable the use of gprof
#
set( PROFILING false )

if( ${PROFILING} )
    add_compile_options( -pg )
    link_directories( /usr/local/lib )
    link_libraries( profiler )
endif( ${PROFILING} )

if( HSE_UNIT_TEST_FLAG )
  message( STATUS "Compiling with unit test flag" )

  # Tool to pre-process files for unit test mocking
  set( UTPP ${HSE_SCRIPTS_DIR}/build/utpp )

  # defines the include directories needed by all tests using mocking
  set( HSE_UT_BASE_INCLUDE_DIRS ${HSE_UT_FMWK_INCLUDE_DIRS}
                                ${HSE_TEST_LIB_INCLUDE_DIRS}
                                ${HSE_TEST_SUPPORT_UT_GS_INCLUDE_DIRS} )

  set( HSE_TEST_SUPPORT_LIBS hse_test_support-lib )

else()

  # defines the include directories needed by all tests using mocking
  set( HSE_UT_BASE_INCLUDE_DIRS ${HSE_UT_FMWK_INCLUDE_DIRS}
                                ${HSE_TEST_LIB_INCLUDE_DIRS} )

endif()


# Needed for the JNI bridge.
#    make -C ~/hse config CMAKE_FLAGS=-DYCSB:BOOL=TRUE
#
set( YCSB TRUE CACHE BOOL "Build JNI bridge artifacts" )
if( ${YCSB} )
    add_compile_options( -fPIC )
endif()


################################################################
#
# Collate the input files, and recreate the mapi output file
#
################################################################

file(WRITE ${CMAKE_BINARY_DIR}/mapi_idx.cmake
"
set( ENV{HSE_BUILD_TYPE_FLAG} ${HSE_BUILD_TYPE_FLAG} )
set( ENV{CMAKE_BINARY_DIR} ${CMAKE_BINARY_DIR} )
set( ENV{CMAKE_STAGE_DIR} ${HSE_STAGE_DIR} )
set( ENV{PUBLIC_HEADERS} ${MPOOL_INCLUDE_DIR} )
execute_process(
    COMMAND ${PROJECT_SOURCE_DIR}/scripts/build/mapi_idx_generate
    OUTPUT_VARIABLE CFL_OUTPUT
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_VARIABLE CFL_ERROR
    ERROR_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE CMD_RES
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
)
if( NOT \${CMD_RES} STREQUAL 0 )
  message( FATAL_ERROR \"Error running mapi_idx_generate.\" \"\${CFL_ERROR}\")
endif()
"
)

add_custom_target(
  mapi_idx
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/mapi_idx.cmake
)

################################################################
#
# Establish prerequisites for all compiled targets
#
################################################################

# All compiled binaries should depend upon hse_version and mapi_idx
macro(add_library _target)
    _add_library(${_target} ${ARGN})
    add_dependencies(${_target} hse_version mapi_idx)
endmacro()

macro(add_executable _target)
    _add_executable(${_target} ${ARGN})
    add_dependencies(${_target} hse_version mapi_idx)
endmacro()

################################################################
#
# Standard sets of libraries for applications to link against
#
################################################################

set(HSE_USER_MPOOL_LINK_LIBS
  mpool
  mpool-blkid
)


#
# Needed to handle the hard-coded link line for hse_platform_init
#
set( HSE_UTIL_BIN_DIR "${PROJECT_BINARY_DIR}/src/util/src" )
set( CMAKE_EXE_LINKER_FLAGS
  "${CMAKE_EXE_LINKER_FLAGS} -L ${HSE_UTIL_BIN_DIR}" )

#
# Needed to handle the hard-coded link line for hse_kvdb_init
#
set( HSE_KVDB_BIN_DIR "${PROJECT_BINARY_DIR}/src" )
set( CMAKE_EXE_LINKER_FLAGS
  "${CMAKE_EXE_LINKER_FLAGS} -L ${HSE_KVDB_BIN_DIR}" )


# Common package options...
#
set( CPACK_PACKAGE_NAME                   "${PROJECT_NAME}" )
set( CPACK_PACKAGE_LICENSE                "ASL 2.0" )
set( CPACK_PACKAGE_VENDOR                 "${BUILD_PKG_VENDOR}" )
set( CPACK_PACKAGE_VERSION                "${PROJECT_VERSION}${BUILD_PKG_VQUAL}" )

set( CPACK_PACKAGE_DESCRIPTION            "Heterogeneous-memory Storage Engine (HSE)" )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY    "hse ${BUILD_PKG_TAG}-${BUILD_STYPE}${BUILD_NUMBER}" )

set( CPACK_COMPONENT_RUNTIME_DESCRIPTION  "${CPACK_PACKAGE_DESCRIPTION} runtime" )
set( CPACK_COMPONENT_DEVEL_DESCRIPTION    "${CPACK_PACKAGE_DESCRIPTION} SDK" )
set( CPACK_COMPONENT_TEST_DESCRIPTION     "${CPACK_PACKAGE_DESCRIPTION} test tools" )
set( CPACK_COMPONENT_TEST_DEPENDS         "runtime" )

if( "${BUILD_NUMBER}" )
    set( BUILD_RTYPE "${BUILD_STYPE}${BUILD_NUMBER}" )
else()
    set( BUILD_RTYPE "" )
endif()

set( CPACK_PACKAGE_RELEASE  "${BUILD_PKG_REL}${BUILD_RTYPE}${BUILD_PKG_DIST}" )


# Create the version header file template:
#
file( WRITE ${CMAKE_BINARY_DIR}/hse_version.h.in
    "static const char hse_version[] = \"@CPACK_PACKAGE_VERSION@-@CPACK_PACKAGE_RELEASE@\";\n"
    "static const char hse_tag[] = \"@BUILD_PKG_TAG@\";\n"
    "static const char hse_sha[] = \"@BUILD_PKG_SHA@\";\n"
)

configure_file(
    ${CMAKE_BINARY_DIR}/hse_version.h.in
    ${HSE_STAGE_DIR}/include/hse_version.h
    @ONLY
    )

################################################################
#
# Subdirectories
#
################################################################

enable_testing()

add_subdirectory( sub )
# we need to compile the test support before any component
add_subdirectory( test/library )
add_subdirectory( src )
add_subdirectory( include )
add_subdirectory( cli )
add_subdirectory( test )
add_subdirectory( scripts/rpm/sos )
add_subdirectory( samples )
add_subdirectory( config )


# Configure cpack now that we've seen all components...
#
include( ${HSE_SCRIPTS_DIR}/${BUILD_PKG_TYPE}/CMakeLists.txt )
include( CPack )


get_directory_property( DirIncs   INCLUDE_DIRECTORIES )
get_directory_property( DirCDefs  COMPILE_DEFINITIONS )
get_directory_property( DirCOpts  COMPILE_OPTIONS )
get_directory_property( DirLDirs  LINK_DIRECTORIES )
get_directory_property( DirLLibs  LINK_LIBRARIES )
get_directory_property( DirLOpts  LINK_OPTIONS )

message( STATUS "Compiler flags:" )
message( STATUS "  INCLUDE_DIRECTORIES     ${DirIncs}" )
message( STATUS "  COMPILE_DEFINITIONS     ${DirCDefs}" )
message( STATUS "  COMPILE_OPTIONS         ${DirCOpts}" )
message( STATUS "  LINK_DIRECTORIES        ${DirLDirs}" )
message( STATUS "  LINK_LIBRARIES          ${DirLLibs}" )
message( STATUS "  LINK_OPTIONS            ${DirLOpts}" )
