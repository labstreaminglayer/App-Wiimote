cmake_minimum_required(VERSION 3.5)
project(Wiimote
	LANGUAGES CXX
	VERSION 0.1.0)

# set up LSL if not done already
if(NOT TARGET LSL::lsl)
	# when building out of tree LSL_ROOT needs to be specified on the cmd line
	file(TO_CMAKE_PATH "${LSL_INSTALL_ROOT}" LSL_INSTALL_ROOT)
	list(APPEND LSL_INSTALL_ROOT "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/build/install")
	find_package(LSL HINTS ${LSL_INSTALL_ROOT}/share/LSL/ ${LSL_INSTALL_ROOT}/LSL/share/LSL QUIET)
	if(NOT LSL_FOUND)
		message(FATAL_ERROR "Precompiled LSL was not found. See https://github.com/labstreaminglayer/labstreaminglayer/blob/master/doc/BUILD.md#lsl_install_root for more information.")
	endif()
	list(APPEND CMAKE_MODULE_PATH ${LSL_DIR})
	message(STATUS "Looking for LSLCMake in ${LSL_DIR}")
	include(LSLCMake)
endif()

# Meta information about the project
set(META_PROJECT_DESCRIPTION "Send Wii data streams (Wiimote, balanceboard, etc) to LSL Outlet.")
set(META_AUTHOR_ORGANIZATION "Ottawa Hospital Research Institute")
set(META_AUTHOR_DOMAIN       "https://github.com/sccn/labstreaminglayer/tree/master/Apps/Wiimote")
set(META_AUTHOR_MAINTAINER   "Chadwick Boulay -- chadwick<dot>boulay<at>gmail<dot>com")

# External Dependencies #

#Wiiuse
include(ExternalProject)
ExternalProject_Add(wiiuse
    GIT_REPOSITORY https://github.com/rpavlik/wiiuse.git
#    GIT_TAG updates
    GIT_SHALLOW 1
    CMAKE_GENERATOR ${gen}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
        -DCMAKE_INCLUDE_PATH=${WIIUSE_INSTALL_DIR}/include
        -DCMAKE_LIBRARY_PATH=${WIIUSE_INSTALL_DIR}/lib
        -DBUILD_WIIUSE_SHARED_LIB:BOOL=OFF
        -DBUILD_EXAMPLE:BOOL=OFF
        -DBUILD_EXAMPLE_SDL:BOOL=OFF
)
ExternalProject_Get_Property(wiiuse install_dir)
set(WIIUSE_INSTALL_DIR ${install_dir})

# Qt
find_package(Qt5 REQUIRED COMPONENTS Core Network Gui Widgets)

add_executable(${PROJECT_NAME} MACOSX_BUNDLE WIN32
	main.cpp
	mainwindow.cpp
	mainwindow.h
	mainwindow.ui
)

add_dependencies(${PROJECT_NAME} wiiuse)

target_include_directories(${PROJECT_NAME} PRIVATE
	${WIIUSE_INSTALL_DIR}/include
)

target_link_libraries(${PROJECT_NAME}
	PRIVATE
	${DEFAULT_LIBRARIES}
	Qt5::Widgets
	Qt5::Network
	LSL::lsl
	${WIIUSE_LIB}
	${PLATFORM_LIBS}
)

IF(UNIX)
	target_link_libraries(${PROJECT_NAME} PRIVATE
		${WIIUSE_INSTALL_DIR}/lib/libwiiuse.a)
elseif(WIN32)
	target_add_definitions(${PROJECT_NAME} PRIVATE -DWIIUSE_STATIC)
	target_link_libraries(${PROJECT_NAME}
		PRIVATE
        debug ${WIIUSE_INSTALL_DIR}/lib/wiiuse_debug.lib
        optimized ${WIIUSE_INSTALL_DIR}/lib/wiiuse.lib)
ENDIF()

# Platform libraries required by Wiiuse
set(PLATFORM_LIBS)
if(APPLE)
    find_library(IOBLUETOOTH_FRAMEWORK
        NAMES
        IOBluetooth)
    find_library(COREFOUNDATION_FRAMEWORK
        NAMES
        CoreFoundation)
    find_library(FOUNDATION_FRAMEWORK
        NAMES
        Foundation)
	target_link_libraries(${PROJECT_NAME}
		PRIVATE
		${IOBLUETOOTH_FRAMEWORK} ${COREFOUNDATION_FRAMEWORK} ${FOUNDATION_FRAMEWORK})
elseif(UNIX)
	include(FindPkgConfig)
	pkg_check_modules(BLUEZ REQUIRED IMPORTED_TARGET bluez)
	target_link_libraries(${PROJECT_NAME} PRIVATE PkgConfig::BLUEZ)
elseif(WIN32)
    find_package(WinHID REQUIRED)
	target_include_directories(${PROJECT_NAME} PRIVATE ${WINHID_INCLUDE_DIRS})
	target_link_libraries(${PROJECT_NAME} PRIVATE ws2_32 setupapi ${WINHID_LIBRARIES})
endif()


installLSLApp(${PROJECT_NAME})

LSLGenerateCPackConfig()
