project(BIOS-DESKTOPSTREAMER)

cmake_minimum_required(VERSION 2.4.0)

# version information
set(VERSION_MAJOR 0)
set(VERSION_MINOR 1)
set(VERSION_PATCH 0)

# required for mixing libraries with full paths and not...
if(COMMAND cmake_policy)
    cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

# show all warnings
set(CMAKE_CXX_FLAGS "-Wall")

# keep the library path in installed executables
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# build options
option(BUILD_DESKTOPSTREAMER "Build BIOS-DesktopStreamer application" OFF)

# path for additional modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/modules/")

if(BUILD_DESKTOPSTREAMER)
    # find and setup Qt4
    # see http://cmake.org/cmake/help/cmake2.6docs.html#module:FindQt4 for details
    set(QT_USE_QTOPENGL TRUE)
    set(QT_USE_QTSVG TRUE)
    set(QT_USE_QTXMLPATTERNS TRUE)
    set(QT_USE_QTXML TRUE)
    set(QT_USE_QTNETWORK TRUE)

    if(WIN32)
        #set(QT_USE_QTMAIN TRUE)
    endif()

    find_package(Qt4 REQUIRED)

    include(${QT_USE_FILE})
    set(LIBS ${LIBS} ${QT_LIBRARIES})

    # libjpeg-turbo
    find_package(LibJpegTurbo REQUIRED)
    include_directories(${LibJpegTurbo_INCLUDE_DIRS})
    set(LIBS ${LIBS} ${LibJpegTurbo_LIBRARIES})
endif()

# DesktopStreamer app
if(BUILD_DESKTOPSTREAMER)
    find_package( Boost COMPONENTS serialization thread date_time program_options filesystem system REQUIRED )
	include_directories( ${Boost_INCLUDE_DIRS} )
	link_directories(${BOOST_LIBRARYDIR})
	
	set(DESKTOP_STREAMER_LIBS ${QT_QTCORE_LIBRARY} ${QT_QTGUI_LIBRARY} ${QT_QTNETWORK_LIBRARY} ${Boost_LIBRARIES})

    if(WIN32)
        set(DESKTOP_STREAMER_LIBS ${DESKTOP_STREAMER_LIBS} ${QT_QTMAIN_LIBRARY})
    endif()

    # libjpeg-turbo
    set(DESKTOP_STREAMER_LIBS ${DESKTOP_STREAMER_LIBS} ${LibJpegTurbo_LIBRARIES})

    set(DESKTOP_STREAMER_SRCS ${DESKTOP_STREAMER_SRCS}
        apps/DesktopStreamer/src/log.cpp
        apps/DesktopStreamer/src/DesktopSelectionRectangle.cpp
        apps/DesktopStreamer/src/DesktopSelectionWindow.cpp
        apps/DesktopStreamer/src/DesktopSelectionView.cpp
        apps/DesktopStreamer/src/main.cpp
        apps/DesktopStreamer/src/MainWindow.cpp
		apps/DesktopStreamer/src/iconDesktopStreamer.rc
    )

    set(DESKTOP_STREAMER_MOC_HEADERS
        apps/DesktopStreamer/src/MainWindow.h
    )

    qt4_wrap_cpp(DESKTOP_STREAMER_MOC_OUTFILES ${DESKTOP_STREAMER_MOC_HEADERS})

    # on Apple platform set the icon
    if(APPLE)
        # set how it shows up in the Info.plist file
        SET(MACOSX_BUNDLE_ICON_FILE desktopstreamer.icns)

        # set where in the bundle to put the icns file
        SET_SOURCE_FILES_PROPERTIES(apps/DesktopStreamer/desktopstreamer.icns PROPERTIES MACOSX_PACKAGE_LOCATION Resources)

        # include the icns file in the target
        SET(DESKTOP_STREAMER_SRCS ${DESKTOP_STREAMER_SRCS} apps/DesktopStreamer/desktopstreamer.icns)
    endif(APPLE)

    add_executable(desktopstreamer MACOSX_BUNDLE WIN32
        ${DESKTOP_STREAMER_SRCS} ${DESKTOP_STREAMER_MOC_OUTFILES})

    target_link_libraries(desktopstreamer ${DESKTOP_STREAMER_LIBS})

    # install executable
    INSTALL(TARGETS desktopstreamer
        RUNTIME DESTINATION bin COMPONENT Runtime
        BUNDLE DESTINATION . COMPONENT Runtime
    )

    # on Apple platform make the .app bundle; on Windows make the installer
    if(APPLE OR WIN32)
        set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "DesktopStreamer is an application that lets you stream your desktop to a running DisplayCluster instance.")
        set(CPACK_PACKAGE_EXECUTABLES "desktopstreamer" "DesktopStreamer")
        set(CPACK_PACKAGE_NAME "DesktopStreamer")
        set(CPACK_PACKAGE_VENDOR "The University of Texas at Austin")
        set(CPACK_PACKAGE_VERSION_MAJOR ${VERSION_MAJOR})
        set(CPACK_PACKAGE_VERSION_MINOR ${VERSION_MINOR})
        set(CPACK_PACKAGE_VERSION_PATCH ${VERSION_PATCH})

        if(APPLE)
            set(APPS "\${CMAKE_INSTALL_PREFIX}/desktopstreamer.app")

            # Qt plugins and qt.conf required so we don't accidentally dynamically link against system-installed Qt libraries
            SET(PLUGIN_DEST_DIR desktopstreamer.app/Contents/MacOS)
            SET(QTCONF_DEST_DIR desktopstreamer.app/Contents/Resources)
        elseif(WIN32)
            set(APPS "\${CMAKE_INSTALL_PREFIX}/bin/desktopstreamer.exe")

            # Qt plugins and qt.conf required so we don't accidentally dynamically link against system-installed Qt libraries
            SET(PLUGIN_DEST_DIR bin)
            SET(QTCONF_DEST_DIR bin)
        endif()

        INSTALL(DIRECTORY "${QT_PLUGINS_DIR}/imageformats" DESTINATION ${PLUGIN_DEST_DIR}/plugins COMPONENT Runtime)

        INSTALL(CODE "
            file(WRITE \"\${CMAKE_INSTALL_PREFIX}/${QTCONF_DEST_DIR}/qt.conf\" \"\")
            " COMPONENT Runtime)

        if(WIN32)
            # directories to look for dependencies
            set(DEPENDENCY_DIRS ${QT_BINARY_DIR})

            include(InstallRequiredSystemLibraries)
        endif()

        INSTALL(CODE "
            file(GLOB_RECURSE QTPLUGINS \"\${CMAKE_INSTALL_PREFIX}/${PLUGIN_DEST_DIR}/plugins/*${CMAKE_SHARED_LIBRARY_SUFFIX}\")
            include(BundleUtilities)
            fixup_bundle(\"${APPS}\" \"\${QTPLUGINS}\" \"${DEPENDENCY_DIRS}\")
            " COMPONENT Runtime)

        # to create a package, one can run "cpack -G DragNDrop CPackConfig.cmake" for example
        # where CPackConfig.cmake is created by including CPack
        if(APPLE)
            set(CPACK_BINARY_DRAGNDROP ON)
        elseif(WIN32)
            set(CPACK_BINARY_NSIS ON)
        endif()

        include(CPack)
    endif()

endif()
