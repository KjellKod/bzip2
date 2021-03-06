# Copyright (c) 2015, Ruslan Baratov
# All rights reserved.

cmake_minimum_required(VERSION 3.0)

project(BZip2
    VERSION 1.0.6
    LANGUAGES C
)

set(local_lib_name bz2)

option(BUILD_TESTING "Build the tests." ON)

string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "WindowsStore" is_windows_store)

if (MSVC)
    if (NOT DEFINED ARCH)
        set (ARCH ${MSVC_C_ARCHITECTURE_ID})
    endif()

    # use boost convention - you cant' get the VC compiler without VS
    if (NOT DEFINED MSVCVER)
        math(EXPR MSVCVER "(${MSVC_VERSION} / 10) - 60")
    endif()

    # add the runtiume version and target architecture into the DLL name per MS
    # guidelines
    set(MSVC_SHARED_SUFFIX_BASE "-${PROJECT_VERSION_MAJOR}-vc${MSVCVER}-mt-${ARCH}")
    set(CMAKE_SHARED_LIBRARY_SUFFIX "${MSVC_SHARED_SUFFIX_BASE}.dll")
endif()

set(bzip2_lib_SRCS
    blocksort.c
    huffman.c
    crctable.c
    randtable.c
    compress.c
    decompress.c
    bzlib.c
)

set(bzip2_lib_HDRS
    bzlib.h
    bzlib_private.h
)
set(bzip2_scripts
    bzdiff
    bzgrep
    bzmore
)
set(bzip2_docs
    bzip2.txt
    manual.pdf
)
set(bzip2_html
    manual.html
)

if(NOT DEFINED CMAKE_DEBUG_POSTFIX)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

if(MSVC)
    add_definitions(-D_FILE_OFFSET_BITS=64)
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)

    set(VER_PRODUCTNAME_STR "bzip2 - a freely available, patent free, high-quality data compressor.")
    set(VER_LEGALCOPYRIGHT_STR "copyright (C) 1996-2010 Julian R Seward")
    set(VER_STR "${PROJECT_VERSION}")
    set(VER_NUM "${PROJECT_VERSION_MAJOR},${PROJECT_VERSION_MINOR},${PROJECT_VERSION_PATCH},0")


    set(VER_ORIGINALFILENAME_STR "${local_lib_name}${CMAKE_SHARED_LIBRARY_SUFFIX}")
    set(VER_INTERNALNAME_STR ${local_lib_name})
    set(VER_FILEDESCRIPTION_STR "${local_lib_name} Dynamic Link Library")
    configure_file(
         ${CMAKE_SOURCE_DIR}/version.rc.cmakein
         ${CMAKE_CURRENT_BINARY_DIR}/version.rc
         @ONLY
     )
    set(bzip2_DLLSRCS
        ${CMAKE_SOURCE_DIR}/libbz2.def
        ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    )
endif(MSVC)

add_library(${local_lib_name} ${bzip2_lib_SRCS} ${bzip2_lib_HDRS} ${bzip2_DLLSRCS})
set_target_properties(${local_lib_name} PROPERTIES
    SOVERSION ${PROJECT_VERSION}
    VERSION ${PROJECT_VERSION_MAJOR}
    PUBLIC_HEADER bzlib.h
)

if(MSVC)
    set_target_properties(${local_lib_name} PROPERTIES PDB_NAME "${local_lib_name}${MSVC_SHARED_SUFFIX_BASE}")
endif()

if(NOT is_windows_store)
  # bzip2recover
  add_executable(bzip2recover bzip2recover.c)
  target_link_libraries(bzip2recover bz2)

  # bzip2
  add_executable(bzip2_bin bzip2.c)
  target_link_libraries(bzip2_bin bz2)
  set_target_properties(bzip2_bin PROPERTIES OUTPUT_NAME bzip2)
endif()

file(GLOB man1 ${bzip2_SOURCE_DIR}/*.1)


if(NOT WIN32)
  install(PROGRAMS ${bzip2_scripts} DESTINATION bin)
endif(NOT WIN32)

install(FILES ${man1}          DESTINATION share/man/man1)
install(FILES ${bzip2_docs}    DESTINATION doc)
install(FILES ${bzip2_html}    DESTINATION doc/html)

if (BUILD_TESTING)
    enable_testing()

    add_executable(spew spewG)

add_test(NAME compress_block_size_1
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-1
    -DINPUT=${CMAKE_SOURCE_DIR}/sample1.ref
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample1.bz2
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample1.rb2
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(NAME compress_block_size_2
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-2
    -DINPUT=${CMAKE_SOURCE_DIR}/sample2.ref
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample2.bz2
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample2.rb2
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(NAME compress_block_size_3
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-3
    -DINPUT=${CMAKE_SOURCE_DIR}/sample3.ref
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample3.bz2
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample3.rb2
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(NAME decompress_block_size_1
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-d
    -DINPUT=${CMAKE_SOURCE_DIR}/sample1.bz2
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample1.ref
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample1.tst
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(NAME decompress_block_size_2
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-d
    -DINPUT=${CMAKE_SOURCE_DIR}/sample2.bz2
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample2.ref
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample2.tst
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(NAME decompress_block_size_3
    COMMAND ${CMAKE_COMMAND}
    -DTEST_PROG=$<TARGET_FILE:bzip2_bin>
    -DARGUMENTS=-ds
    -DINPUT=${CMAKE_SOURCE_DIR}/sample3.bz2
    -DEXPECTED=${CMAKE_SOURCE_DIR}/sample3.ref
    -DOUTPUT=${CMAKE_CURRENT_BINARY_DIR}/sample3.tst
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

endif()

####
# Installation (https://github.com/forexample/package-example)

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: variable 'targets_export_name' used
configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)

set(targets_to_install bz2)
if(NOT IOS AND NOT is_windows_store)
  list(APPEND targets_to_install bzip2recover bzip2_bin)
endif()

install(
    TARGETS ${targets_to_install}
    EXPORT "${targets_export_name}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
    PUBLIC_HEADER DESTINATION "${include_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
