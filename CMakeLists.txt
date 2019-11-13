cmake_minimum_required(VERSION 3.5)

project(Wiimote
	LANGUAGES CXX
	VERSION 0.1.0)
include(Findliblsl.cmake)

# Include cmake modules
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)
find_package(Qt5 REQUIRED COMPONENTS Widgets)

#Wiiuse
include(ExternalProject)
set(WIIUSE_INSTALL_DIR ${CMAKE_CURRENT_LIST_DIR}/deps/local)
ExternalProject_Add(wiiuse
    PREFIX ${CMAKE_CURRENT_LIST_DIR}/deps/wiiuse
    INSTALL_DIR ${WIIUSE_INSTALL_DIR}
    GIT_REPOSITORY https://github.com/rpavlik/wiiuse.git
    GIT_TAG updates
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
IF(APPLE)
    set(WIIUSE_LIB ${WIIUSE_INSTALL_DIR}/lib/libwiiuse.a)
ELSEIF(WIN32)
    add_definitions(-DWIIUSE_STATIC)
    set(WIIUSE_LIB
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
    list(APPEND PLATFORM_LIBS ${IOBLUETOOTH_FRAMEWORK} ${COREFOUNDATION_FRAMEWORK} ${FOUNDATION_FRAMEWORK})
else(WIN32)
    find_package(WinHID REQUIRED)
    include_directories(${WINHID_INCLUDE_DIRS})
    list(APPEND PLATFORM_LIBS ws2_32 setupapi ${WINHID_LIBRARIES})
endif(APPLE)

add_executable(${PROJECT_NAME} MACOSX_BUNDLE WIN32
	main.cpp
	mainwindow.cpp
	mainwindow.h
	mainwindow.ui
)

add_dependencies(${PROJECT_NAME} wiiuse)

target_include_directories(${PROJECT_NAME}
    PRIVATE
    ${WIIUSE_INSTALL_DIR}/include)

target_link_libraries(${PROJECT_NAME}
	PRIVATE
	Qt5::Widgets
	LSL::lsl
    ${WIIUSE_LIB}
    ${PLATFORM_LIBS}
)


# Deployment #
installLSLApp(${PROJECT_NAME})
LSLGenerateCPackConfig()
