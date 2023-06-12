cmake_minimum_required(VERSION 3.14)

project(Wiimote
    LANGUAGES CXX C
    VERSION 0.2.0)

# Include custom cmake modules
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Config
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGNING_REQUIRED "NO")
set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "")
set(LSL_COMFY_DEFAULTS ON)

# Thirdparty Libraries

## LSL
# Find an installed liblsl in paths set by the user (LSL_INSTALL_ROOT)
# and some default paths
find_package(LSL REQUIRED
        HINTS ${LSL_INSTALL_ROOT}
        "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/install"
        "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/build/"
        "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/build/install"
        "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/out/build/x64-Release"
        "${CMAKE_CURRENT_LIST_DIR}/../../LSL/liblsl/out/install/x64-Release"
        PATH_SUFFIXES share/LSL)

## Qt
set(CMAKE_AUTOMOC ON)  # The later version of this in LSLCMake is somehow not enough.
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)
find_package(Qt6 REQUIRED COMPONENTS Core Widgets)

## Wiiuse
include(FetchContent)
set(BUILD_SHARED_LIBS OFF CACHE INTERNAL "")
FetchContent_Declare(wiiuse
        GIT_REPOSITORY https://github.com/wiiuse/wiiuse.git
        GIT_TAG master
        GIT_SHALLOW 1
        )
FetchContent_MakeAvailable(wiiuse)

# Target
add_executable(${PROJECT_NAME} MACOSX_BUNDLE WIN32
        src/main.cpp
        src/mainwindow.cpp
        src/mainwindow.h
        src/mainwindow.ui
)
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        Qt6::Widgets
        LSL::lsl
        wiiuse
)
target_include_directories(${PROJECT_NAME} PRIVATE ${WIIUSE_INCLUDE_DIRS})
target_compile_definitions(${PROJECT_NAME} PRIVATE WIIUSE_STATIC)

## Copy dlls to build so that you can debug in IDE
if (WIN32)
    get_target_property(QT_QMAKE_EXECUTABLE Qt::qmake IMPORTED_LOCATION)
    get_filename_component(QT_WINDEPLOYQT_EXECUTABLE ${QT_QMAKE_EXECUTABLE} PATH)
    set(QT_WINDEPLOYQT_EXECUTABLE "${QT_WINDEPLOYQT_EXECUTABLE}/windeployqt.exe")

    add_custom_command(
            TARGET ${PROJECT_NAME}
            POST_BUILD
            COMMAND
                ${QT_WINDEPLOYQT_EXECUTABLE}
                --no-translations --no-system-d3d-compiler --qmldir ${CMAKE_CURRENT_SOURCE_DIR}
            $<TARGET_FILE_DIR:${PROJECT_NAME}>
    )
    add_custom_command(
            TARGET ${PROJECT_NAME}
            POST_BUILD
            COMMAND
                ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_RUNTIME_DLLS:${PROJECT_NAME}>
                $<TARGET_FILE_DIR:${PROJECT_NAME}>
            COMMAND_EXPAND_LISTS
    )
endif()

# Deployment
installLSLApp(${PROJECT_NAME})
# LSLGenerateCPackConfig()
