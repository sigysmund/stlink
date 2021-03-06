cmake_minimum_required(VERSION 2.8.7)
project(stlink C)
set(PROJECT_DESCRIPTION "Open source version of the STMicroelectronics Stlink Tools")

include(CheckCCompilerFlag)
include(CheckIncludeFile)
find_package(PkgConfig)

if (POLICY CMP0042)
	# Newer cmake on MacOS should use @rpath
	cmake_policy (SET CMP0042 NEW)
endif ()

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${CMAKE_SOURCE_DIR}/cmake/modules")

message(STATUS "Building on ${CMAKE_SYSTEM_NAME}")

# Determine package version.
find_package (Git QUIET)
if (DEFINED ENV{TRAVIS_TAG} AND NOT "$ENV{TRAVIS_TAG}" STREQUAL "")
	set (STLINK_PACKAGE_VERSION "$ENV{TRAVIS_TAG}")
elseif (GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
	# Working off a git repo, using git versioning
	# Check if HEAD is pointing to a tag
	execute_process (
		COMMAND             "${GIT_EXECUTABLE}" describe --always
		WORKING_DIRECTORY   "${PROJECT_SOURCE_DIR}"
		OUTPUT_VARIABLE     STLINK_PACKAGE_VERSION
		OUTPUT_STRIP_TRAILING_WHITESPACE)

	# If the sources have been changed locally, add -dirty to the version.
	execute_process (
	COMMAND             "${GIT_EXECUTABLE}" diff --quiet
	WORKING_DIRECTORY   "${PROJECT_SOURCE_DIR}"
	RESULT_VARIABLE     res)

	if (res EQUAL 1)
		set (STLINK_PACKAGE_VERSION "${STLINK_PACKAGE_VERSION}-dirty")
	endif()
elseif (EXISTS ${PROJECT_SOURCE_DIR}/.version)
	#  If git is not available (e.g. when building from source package)
	#  we can extract the package version from .version file.
	file (STRINGS .version STLINK_PACKAGE_VERSION)
else ()
	set (STLINK_PACKAGE_VERSION "999.99.99-unknown")
endif()

# We shall use STLINK_PACKAGE_VERSION* variables to set soversion and other stuff
# Since these numbers will appear in .so names, e.g. libstlink.so.1.2.0
# We can't just use STLINK_PACKAGE_VERSION here
# (Well, we can, but that breaks all the existing conventions)
# We can't as well use PROJECT_VERSION* variables here, since they are managed
# by project() directive and the whole version detection relies on PROJECT_SOURCE_DIR

string(REPLACE "." ";" STLINK_PACKAGE_VERSION_ARRAY ${STLINK_PACKAGE_VERSION})
string(REPLACE "-" ";" STLINK_PACKAGE_VERSION_ARRAY "${STLINK_PACKAGE_VERSION_ARRAY}")
list(GET STLINK_PACKAGE_VERSION_ARRAY 0 STLINK_PACKAGE_VERSION_MAJOR)
list(GET STLINK_PACKAGE_VERSION_ARRAY 1 STLINK_PACKAGE_VERSION_MINOR)
list(GET STLINK_PACKAGE_VERSION_ARRAY 2 STLINK_PACKAGE_VERSION_PATCH)
list(GET STLINK_PACKAGE_VERSION_ARRAY 3 STLINK_PACKAGE_VERSION_TAG)

if(WIN32)
	find_package(7Zip REQUIRED)
	message(STATUS "7Zip Location: " ${ZIP_LOCATION})
endif()

find_package(LibUSB REQUIRED)
if (NOT APPLE AND NOT WIN32)
	find_package(PkgConfig)
	pkg_check_modules(gtk gtk+-3.0)
endif ()

function(add_cflag_if_supported flag)
    string(REPLACE "-" "_" flagclean ${flag})
    string(REPLACE "=" "_" flagclean ${flagclean})
    string(REPLACE "+" "_" flagclean ${flagclean})
    string(REPLACE "," "_" flagclean ${flagclean})
    string(TOUPPER ${flagclean} flagclean)

    check_c_compiler_flag(${flag} C_SUPPORTS${flagclean})

    if (C_SUPPORTS${flagclean})
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}" PARENT_SCOPE)
    endif()
endfunction()

add_cflag_if_supported("-std=gnu99")
add_cflag_if_supported("-Wall")
add_cflag_if_supported("-Wextra")
add_cflag_if_supported("-Wshadow")
add_cflag_if_supported("-D_FORTIFY_SOURCE=2")
add_cflag_if_supported("-fstrict-aliasing")
add_cflag_if_supported("-Wformat")
add_cflag_if_supported("-Wformat-security")
add_cflag_if_supported("-Wmaybe-uninitialized")
add_cflag_if_supported("-Wmissing-variable-declarations")
add_cflag_if_supported("-Wshorten-64-to-32")
add_cflag_if_supported("-Wimplicit-function-declaration")

##
# On OpenBSD the system headers suck so we need to disable redundant declaration check
# /usr/include/unistd.h:429: warning: redundant redeclaration of 'truncate'
# /usr/include/sys/types.h:218: warning: previous declaration of 'truncate' was here
##
if (NOT CMAKE_SYSTEM_NAME STREQUAL "OpenBSD")
	add_cflag_if_supported("-Wredundant-decls")
endif ()

add_cflag_if_supported("-Wundef")
if (NOT WIN32)
	add_cflag_if_supported("-fPIC")
endif ()

if (CMAKE_BUILD_TYPE STREQUAL "")
	set (CMAKE_BUILD_TYPE "Debug")
endif ()

if(${CMAKE_BUILD_TYPE} MATCHES "Debug")
	add_cflag_if_supported("-ggdb")
	add_cflag_if_supported("-O0")
	include(CTest)
elseif()
	add_cflag_if_supported("-O2")
endif()

CHECK_INCLUDE_FILE(sys/mman.h STLINK_HAVE_SYS_MMAN_H)
if (STLINK_HAVE_SYS_MMAN_H)
	add_definitions(-DSTLINK_HAVE_SYS_MMAN_H)
endif()

message("Building ${PROJECT_NAME} ${STLINK_PACKAGE_VERSION}")

set(STLINK_HEADERS
	include/stlink.h
	include/stlink/usb.h
	include/stlink/sg.h
	include/stlink/logging.h
	include/stlink/mmap.h
	include/stlink/chipid.h
	include/stlink/flash_loader.h
)

set(STLINK_SOURCE
	src/chipid.c
	src/common.c
	src/usb.c
	src/sg.c
	src/logging.c
	src/flash_loader.c
)

if (WIN32 OR MSYS OR MINGW)
	set (STLINK_SOURCE "${STLINK_SOURCE};src/mmap.c;src/mingw/mingw.c") # TODO
endif ()

include_directories(${LIBUSB_INCLUDE_DIR})
include_directories(include)
include_directories(src/mingw)

set(STLINK_LIB_STATIC ${PROJECT_NAME}-static)

add_library(${PROJECT_NAME} SHARED
            ${STLINK_HEADERS} # header files for ide projects generated by cmake
            ${STLINK_SOURCE})
target_link_libraries(${PROJECT_NAME} ${LIBUSB_LIBRARY})

if (WIN32 OR MSYS OR MINGW)
	set(STLINK_SHARED_VERSION
		${STLINK_PACKAGE_VERSION_MAJOR}.${STLINK_PACKAGE_VERSION_MINOR})
else()
	set(STLINK_SHARED_VERSION
		${STLINK_PACKAGE_VERSION_MAJOR}.${STLINK_PACKAGE_VERSION_MINOR}.${STLINK_PACKAGE_VERSION_PATCH})
endif()

set_target_properties(${PROJECT_NAME} PROPERTIES SOVERSION ${STLINK_PACKAGE_VERSION_MAJOR}
VERSION ${STLINK_SHARED_VERSION})

add_library(${STLINK_LIB_STATIC} STATIC
            ${STLINK_HEADERS} # header files for ide projects generated by cmake
            ${STLINK_SOURCE})
target_link_libraries(${STLINK_LIB_STATIC} ${LIBUSB_LIBRARY})
set_target_properties(${STLINK_LIB_STATIC} PROPERTIES OUTPUT_NAME ${PROJECT_NAME})

if (APPLE)
	find_library(ObjC objc)
	find_library(CoreFoundation CoreFoundation)
	find_library(IOKit IOKit)
	target_link_libraries(${PROJECT_NAME} ${CoreFoundation} ${IOKit} ${ObjC})
endif ()

if(WIN32 OR MSYS OR MINGW)
	target_link_libraries(${PROJECT_NAME} ${LIBUSB_LIBRARY} wsock32 ws2_32)
else()
	target_link_libraries(${PROJECT_NAME} ${LIBUSB_LIBRARY})
endif()

add_executable(st-flash src/tools/flash.c src/tools/flash_opts.c)
target_link_libraries(st-flash ${PROJECT_NAME})

add_executable(st-info src/tools/info.c)
target_link_libraries(st-info ${PROJECT_NAME})

add_executable(st-util src/gdbserver/gdb-remote.c
                       src/gdbserver/gdb-remote.h
                       src/gdbserver/gdb-server.c
                       src/gdbserver/gdb-server.h
                       src/gdbserver/semihosting.c
                       src/gdbserver/semihosting.h)

target_link_libraries(st-util ${PROJECT_NAME})

install(TARGETS ${PROJECT_NAME} ${STLINK_LIB_STATIC} st-flash st-util st-info
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib/${CMAKE_LIBRARY_PATH}
		LIBRARY DESTINATION lib/${CMAKE_LIBRARY_PATH}
)

if (NOT APPLE AND NOT WIN32)
	set(PKG_CONFIG_LIBDIR
    	"\${prefix}/lib/\${deb_host_multiarch}"
	)
	set(PKG_CONFIG_INCLUDEDIR
    	"\${prefix}/include/\${deb_host_multiarch}/${PROJECT_NAME}-${STLINK_PACKAGE_VERSION}"
	)
	set(PKG_CONFIG_LIBS
    	"-L\${libdir} -l:libstlink.so.${STLINK_PACKAGE_VERSION_MAJOR}"
	)
	set(PKG_CONFIG_CFLAGS
    	"-I\${includedir}"
	)

	set(PKG_CONFIG_REQUIRES
		"libusb-1.0"
	)

	configure_file(
  	"${CMAKE_CURRENT_SOURCE_DIR}/pkg-config.pc.cmake"
  	"${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
	)

	install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
       	DESTINATION lib/${CMAKE_LIBRARY_PATH}/pkgconfig/)
endif()

if (NOT APPLE AND gtk_FOUND)
  include_directories(SYSTEM ${gtk_INCLUDE_DIRS})
  set(GUI_SOURCES src/tools/gui/stlink-gui.c
                  src/tools/gui/stlink-gui.h)

  # TODO REMOVE whole stlink-gui-local target, fixup auto-search of ui file in implementation
  add_executable(stlink-gui-local ${GUI_SOURCES})
  set_target_properties(stlink-gui-local PROPERTIES
                        COMPILE_FLAGS -DSTLINK_UI_DIR=\\"${CMAKE_CURRENT_SOURCE_DIR}/gui\\")
  target_link_libraries(stlink-gui-local stlink ${gtk_LDFLAGS})

  set(INSTALLED_UI_DIR share/stlink)
  add_executable(stlink-gui ${GUI_SOURCES})
  # TODO REMOVE, fixup auto-search of ui file in implementation
  set_target_properties(stlink-gui PROPERTIES
                        COMPILE_FLAGS -DSTLINK_UI_DIR=\\"${CMAKE_INSTALL_PREFIX}/${INSTALLED_UI_DIR}\\")
  target_link_libraries(stlink-gui stlink ${gtk_LDFLAGS})

  install(TARGETS stlink-gui
          RUNTIME DESTINATION bin)
  install(FILES src/tools/gui/stlink-gui.ui
          DESTINATION ${INSTALLED_UI_DIR})
endif()

add_subdirectory(include)
add_subdirectory(doc/man)
add_subdirectory(tests)

set (CPACK_PACKAGE_NAME ${PROJECT_NAME})
set (CPACK_PACKAGE_VERSION ${STLINK_PACKAGE_VERSION})
set (CPACK_SOURCE_GENERATOR "TBZ2;ZIP")
set (CPACK_SOURCE_IGNORE_FILES "/build/;/.git/;~$;${CPACK_SOURCE_IGNORE_FILES}")
set (CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${STLINK_PACKAGE_VERSION}")
if (APPLE)
	set(CPACK_GENERATOR "ZIP")
	file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/dist/osx")
	set(CPACK_OUTPUT_FILE_PREFIX "${CMAKE_BINARY_DIR}/dist/osx")
endif()
add_custom_target(dist COMMAND ${CMAKE_MAKE_PROGRAM} package_source)
include (CPack)
