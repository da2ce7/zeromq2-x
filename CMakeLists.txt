# CMake build script for ZeroMQ on Windows.
#

cmake_minimum_required (VERSION 2.8)

project (ZeroMQ)

# TODO: Extract from include/zmq.h
set(ZMQ_VERSION_MAJOR 2)
set(ZMQ_VERSION_MINOR 2)
set(ZMQ_VERSION_PATCH 1)

# WARNING: Windows Python will override Cygwin yet not work with Asciidoc.
find_package (PythonInterp REQUIRED)
# Workaround, manually set Python location
##set(PYTHON_EXECUTABLE c:/cygwin/bin/python2.6.exe)

set(OPENPGM_ROOT /libpgm/libpgm-5.1.118-1~dfsg/openpgm/pgm)

# TODO: Replace with FindAsciidoc.cmake
set(ASCIIDOC_EXECUTABLE c:/cygwin/bin/asciidoc)

option (WITH_DOC "Build Reference Guide documentation (requires DocBook)" OFF)

#-----------------------------------------------------------------------------
# force off-tree build

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
message(FATAL_ERROR "CMake generation is not allowed within the source directory!
Remove the CMakeCache.txt file and try again from another folder, e.g.:

   del CMakeCache.txt
   mkdir build
   cd build
   cmake ..
")
endif(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})

#-----------------------------------------------------------------------------
# default to Release build

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
endif(NOT CMAKE_BUILD_TYPE)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
set(LIBRARY_OUTPUT_PATH  ${CMAKE_BINARY_DIR}/lib)

#-----------------------------------------------------------------------------
# platform specifics

add_definitions(
        -DWIN32
	-DDLL_EXPORT
# NB: May require tweaking for highly connected applications.
	-DFD_SETSIZE=1024
)

#-----------------------------------------------------------------------------
# source files

set(cxx-sources
	clock.cpp
	command.cpp
	connect_session.cpp
	ctx.cpp
	decoder.cpp
	device.cpp
	devpoll.cpp
	dist.cpp
	encoder.cpp
	epoll.cpp
	err.cpp
	fq.cpp
	io_object.cpp
	io_thread.cpp
	ip.cpp
	kqueue.cpp
	lb.cpp
	mailbox.cpp
	named_session.cpp
	object.cpp
	options.cpp
	own.cpp
	pair.cpp
	pgm_receiver.cpp
	pgm_sender.cpp
	pgm_socket.cpp
	pipe.cpp
	poll.cpp
	poller_base.cpp
	pub.cpp
	pull.cpp
	push.cpp
	reaper.cpp
	rep.cpp
	req.cpp
	select.cpp
	session.cpp
	signaler.cpp
	socket_base.cpp
	sub.cpp
	swap.cpp
	tcp_connecter.cpp
	tcp_listener.cpp
	tcp_socket.cpp
	thread.cpp
	transient_session.cpp
	trie.cpp
	uuid.cpp
	xpub.cpp
	xreq.cpp
	xrep.cpp
	xsub.cpp
	zmq.cpp
	zmq_connecter.cpp
	zmq_engine.cpp
	zmq_init.cpp
	zmq_listener.cpp
)
set(rc-sources
	version.rc
)

include_directories(
        include
	${CMAKE_BINARY_DIR}
)
set(headers
	include/zmq.h
	include/zmq.hpp
	include/zmq_utils.h
)
set(readme-docs
	AUTHORS
	COPYING
	COPYING.LESSER
	MAINTAINERS
	NEWS
	README
)

#-----------------------------------------------------------------------------
# optional modules

add_definitions(
	-DZMQ_HAVE_OPENPGM
)

include_directories(
	${OPENPGM_ROOT}/include
)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
# Win64
	if (CMAKE_BUILD_TYPE STREQUAL "Debug")
		set(OPENPGM_BUILD_TYPE "debug64")
	else (CMAKE_BUILD_TYPE STREQUAL "Debug")
		set(OPENPGM_BUILD_TYPE "build64")
	endif (CMAKE_BUILD_TYPE STREQUAL "Debug")
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
# Win32
	if (CMAKE_BUILD_TYPE STREQUAL "Debug")
		set(OPENPGM_BUILD_TYPE "debug")
	else (CMAKE_BUILD_TYPE STREQUAL "Debug")
		set(OPENPGM_BUILD_TYPE "build")
	endif (CMAKE_BUILD_TYPE STREQUAL "Debug")
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)

set(OPENPGM_LIBRARYDIR ${OPENPGM_ROOT}/${OPENPGM_BUILD_TYPE}/lib)

link_directories(
	${OPENPGM_LIBRARYDIR}
)

#-----------------------------------------------------------------------------
# source generators

foreach (source ${cxx-sources} ${rc-sources})
	list(APPEND sources ${CMAKE_SOURCE_DIR}/src/${source})
endforeach()

add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/platform.hpp
	COMMAND ${CMAKE_COMMAND}
	ARGS	-E
		copy
		${CMAKE_SOURCE_DIR}/builds/msvc/platform.hpp
		${CMAKE_BINARY_DIR}/platform.hpp
	DEPENDS ${CMAKE_SOURCE_DIR}/builds/msvc/platform.hpp
)
list(APPEND sources ${CMAKE_BINARY_DIR}/platform.hpp)

configure_file(${CMAKE_SOURCE_DIR}/src/version.rc.in
	${CMAKE_SOURCE_DIR}/src/version.rc
	@ONLY)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
	set (nsis-template ${CMAKE_SOURCE_DIR}/NSIS.template64.in)
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
	set (nsis-template ${CMAKE_SOURCE_DIR}/NSIS.template32.in)
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)
add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/NSIS.template.in
	COMMAND ${CMAKE_COMMAND}
	ARGS	-E
		copy
		${nsis-template}
		${CMAKE_BINARY_DIR}/NSIS.template.in
	DEPENDS ${nsis-template}
)

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/doc)
file(GLOB docs RELATIVE ${CMAKE_BINARY_DIR}/ "${CMAKE_SOURCE_DIR}/doc/*.txt")
set (html-docs)
foreach (txt ${docs})
	string (REGEX REPLACE ".*/(.*)\\.txt" "\\1.html" html ${txt})
	set (src ${txt})
	set (dst doc/${html})
	add_custom_command(
		OUTPUT	${dst}
		COMMAND ${PYTHON_EXECUTABLE}
		ARGS	-x
			${ASCIIDOC_EXECUTABLE}
			-d manpage
			-b xhtml11
			-f ${CMAKE_SOURCE_DIR}/doc/asciidoc.conf
			-azmq_version=${ZMQ_VERSION_MAJOR}.${ZMQ_VERSION_MINOR}.${ZMQ_VERSION_PATCH}
			-o ${dst}
			${src}
		DEPENDS ${CMAKE_BINARY_DIR}/${src}
		WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
		COMMENT "Generating ${html}"
	)
	if (WITH_DOC)
		list(APPEND html-docs ${CMAKE_BINARY_DIR}/${dst})
	endif (WITH_DOC)
endforeach (txt ${docs})

#-----------------------------------------------------------------------------
# output

add_library(libzmq SHARED ${sources} ${html-docs} ${CMAKE_BINARY_DIR}/NSIS.template.in)
target_link_libraries(libzmq ws2_32.lib rpcrt4.lib)

set_target_properties(libzmq PROPERTIES RELEASE_POSTFIX "-v100-mt" DEBUG_POSTFIX "-v100-mt-gd")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
	install (FILES ${CMAKE_BINARY_DIR}/lib/libzmq-v100-mt-gd.dll DESTINATION bin COMPONENT SDK)
	install (FILES ${CMAKE_BINARY_DIR}/lib/libzmq-v100-mt-gd.lib DESTINATION lib COMPONENT SDK)
	install (FILES ${CMAKE_BINARY_DIR}/lib/libzmq-v100-mt-gd.pdb DESTINATION lib COMPONENT SDK)
else (CMAKE_BUILD_TYPE STREQUAL "Debug")
	install (TARGETS libzmq RUNTIME DESTINATION bin COMPONENT Runtime)
	install (FILES ${CMAKE_BINARY_DIR}/lib/libzmq-v100-mt.lib DESTINATION lib COMPONENT SDK)
endif (CMAKE_BUILD_TYPE STREQUAL "Debug")
install (FILES ${headers} DESTINATION include COMPONENT SDK)

set (perf-tools
	local_lat
	remote_lat
	local_thr
	remote_thr
	inproc_lat
	inproc_thr
)
if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
	foreach (perf-tool ${perf-tools})
		add_executable (${perf-tool} perf/${perf-tool}.cpp)
		target_link_libraries (${perf-tool} libzmq)
		install (TARGETS ${perf-tool} RUNTIME DESTINATION bin COMPONENT PerfTools)
	endforeach (perf-tool ${perf-tools})
endif (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")

file(GLOB headers "${CMAKE_SOURCE_DIR}/src/*.hpp")
install (FILES ${sources} ${headers} DESTINATION src COMPONENT SourceCode)

foreach (readme ${readme-docs})
	configure_file (${CMAKE_SOURCE_DIR}/${readme} ${CMAKE_BINARY_DIR}/${readme}.txt)
	install (FILES ${CMAKE_BINARY_DIR}/${readme}.txt DESTINATION .)
endforeach (readme ${readme-docs})
if (WITH_DOC)
	install (FILES ${html-docs} DESTINATION doc COMPONENT RefGuide)
endif (WITH_DOC)

include (InstallRequiredSystemLibraries)

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
	set (CPACK_INSTALL_CMAKE_PROJECTS
		"${CMAKE_SOURCE_DIR}/build64;ZeroMQ;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug64;ZeroMQ;ALL;/"
	)
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
	set (CPACK_INSTALL_CMAKE_PROJECTS
		"${CMAKE_SOURCE_DIR}/build;ZeroMQ;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug;ZeroMQ;ALL;/"
	)
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)

set (CMAKE_MODULE_PATH "${CMAKE_BINARY_DIR}")
set (CPACK_PACKAGE_DESCRIPTION_SUMMARY "ZeroMQ lightweight messaging kernel")
set (CPACK_PACKAGE_VENDOR "iMatix")
set (CPACK_NSIS_CONTACT "iMatix <info@imatix.com>")
set (CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/COPYING.txt")
# There is a bug in NSI that does not handle full unix paths properly. Make
# sure there is at least one set of four (4) backlasshes.
set (CPACK_NSIS_MUI_ICON "${CMAKE_SOURCE_DIR}\\\\installer.ico")
set (CPACK_NSIS_MUI_UNIICON "${CMAKE_SOURCE_DIR}\\\\installer.ico")
set (CPACK_PACKAGE_ICON "${CMAKE_SOURCE_DIR}\\\\branding.bmp")
set (CPACK_NSIS_COMPRESSOR "/SOLID lzma")
set (CPACK_PACKAGE_VERSION_MAJOR "${ZMQ_VERSION_MAJOR}")
set (CPACK_PACKAGE_VERSION_MINOR "${ZMQ_VERSION_MINOR}")
set (CPACK_PACKAGE_VERSION_PATCH "${ZMQ_VERSION_PATCH}")

include (CPack)

cpack_add_component_group (Development
	DISPLAY_NAME "ZeroMQ software development kit"
	EXPANDED
)
cpack_add_component (PerfTools
	DISPLAY_NAME "ZeroMQ performance tools"
	INSTALL_TYPES FullInstall DevInstall
)
cpack_add_component (SourceCode
	DISPLAY_NAME "ZeroMQ source code"
	DISABLED
	INSTALL_TYPES FullInstall
)
cpack_add_component (SDK
	DISPLAY_NAME "ZeroMQ headers and libraries"
	INSTALL_TYPES FullInstall DevInstall
	GROUP Development
)
if (WITH_DOC)
	cpack_add_component (RefGuide
		DISPLAY_NAME "ZeroMQ reference guide"
		INSTALL_TYPES FullInstall DevInstall
		GROUP Development
	)
endif (WITH_DOC)
cpack_add_component (Runtime
	DISPLAY_NAME "ZeroMQ runtime files"
	REQUIRED
	INSTALL_TYPES FullInstall DevInstall MinInstall
)
cpack_add_install_type (FullInstall
	DISPLAY_NAME "Full install, including source code"
)
cpack_add_install_type (DevInstall
	DISPLAY_NAME "Developer install, headers and libraries"
)
cpack_add_install_type (MinInstall
	DISPLAY_NAME "Minimal install, runtime only"
)

# end of file
