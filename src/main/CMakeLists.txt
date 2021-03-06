#=============================================================================
#  MuseScore
#  Music Composition & Notation
#
#  Copyright (C) 2019 MuseScore BVBA
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License version 2.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#  along with this program; if not, write to the Free Software
#  Foundation, Inc., 675 Mass Ave, Cambridge, MA 02139, USA.
#=============================================================================

# Setup main application
include(GetPlatformInfo)

# Common
set(EXECUTABLE_NAME mscore)

# Platform specific
if (OS_IS_WIN)

    set(MSCORE_OUTPUT_NAME ${MUSESCORE_NAME}${MUSESCORE_VERSION_MAJOR})

    include(GetCompilerInfo)
    if (CC_IS_MSVC)
        # MSVC recognizes a *.rc file and will invoke the resource compiler to link it
        set (ICON_RES_FILE ${CMAKE_CURRENT_LIST_DIR}/res/mscore.rc)

    endif(CC_IS_MSVC)

    if (CC_IS_MINGW)
        set(ICON_RES_FILE ${PROJECT_BINARY_DIR}/resfile.o)
        add_custom_command(
           OUTPUT ${PROJECT_BINARY_DIR}/resfile.o
           COMMAND ${CMAKE_RC_COMPILER} -i mscore.rc -o ${PROJECT_BINARY_DIR}/resfile.o
           DEPENDS ${PROJECT_SOURCE_DIR}/src/main/res/mscore.rc
           WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/src/main/res
           )
        set_source_files_properties(${PROJECT_BINARY_DIR}/resfile.o PROPERTIES generated true)

    endif(CC_IS_MINGW)

elseif(OS_IS_LIN)

    if (MSCORE_INSTALL_SUFFIX)
        set(MSCORE_OUTPUT_NAME "${EXECUTABLE_NAME}${MSCORE_INSTALL_SUFFIX}")
    endif(MSCORE_INSTALL_SUFFIX)

    set(CMAKE_INSTALL_RPATH "${QT_INSTALL_LIBS}")
    if (BUILD_SHARED_LIBS)
        set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_RPATH};${SHARED_LIBS_INSTALL_DESTINATION}")
    endif(BUILD_SHARED_LIBS)

elseif(OS_IS_MAC)

    set(EXECUTABLE_NAME mscore MACOSX_BUNDLE)
    set(MACOSX_BUNDLE_ICON_FILE mscore.icns)
    set(MACOSX_BUNDLE_GUI_IDENTIFIER org.musescore.${MUSESCORE_NAME})
    set(MACOSX_BUNDLE_BUNDLE_NAME ${MUSESCORE_NAME})
    set(MACOSX_BUNDLE_LONG_VERSION_STRING ${MUSESCORE_VERSION_FULL})
    set(MACOSX_BUNDLE_SHORT_VERSION_STRING ${MUSESCORE_VERSION})
    set(MACOSX_BUNDLE_BUNDLE_VERSION ${MUSESCORE_VERSION_FULL})
    set(MACOSX_BUNDLE_COPYRIGHT musescore.org)
    set(MACOSX_BUNDLE_SHORT_VERSION_STRING ${MUSESCORE_VERSION_FULL}.${CMAKE_BUILD_NUMBER})
    set(MACOSX_BUNDLE_BUNDLE_VERSION ${MUSESCORE_VERSION_FULL}.${CMAKE_BUILD_NUMBER})

    set(CMAKE_INSTALL_RPATH ${QT_INSTALL_LIBS})

else()
    message(FATAL_ERROR "Unsopported Platform: ${CMAKE_HOST_SYSTEM_NAME}")
endif()


add_executable(${EXECUTABLE_NAME}
      ${ICON_RES_FILE}
      main.cpp
      modulessetup.cpp
      modulessetup.h
      )

# If MSCORE_OUTPUT_NAME is set (e.g, when cmake is called by the user), the output executable will be
# called MSCORE_OUTPUT_NAME instead of 'mscore'. This can be used to have MuseScore stable and unstable
# both installed in the same prefix on a Linux system.
if (MSCORE_OUTPUT_NAME)
  set_target_properties(${EXECUTABLE_NAME} PROPERTIES OUTPUT_NAME ${MSCORE_OUTPUT_NAME})
endif (MSCORE_OUTPUT_NAME)

if (OS_IS_MAC)
    set_target_properties(mscore PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${PROJECT_SOURCE_DIR}/build/MacOSXBundleInfo.plist.in)
endif (OS_IS_MAC)


if (CC_IS_MINGW)
    set_target_properties(mscore PROPERTIES LINK_FLAGS "-Wl,-S -mwindows" )
endif(CC_IS_MINGW)

if (CC_IS_MSVC)
    target_link_options(mscore PRIVATE /SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup)
endif(CC_IS_MSVC)



set(FREETYPE_LIB )
if (USE_SYSTEM_FREETYPE)
     set(FREETYPE_LIB ${FREETYPE_LIBRARIES})
else (USE_SYSTEM_FREETYPE)
     set(FREETYPE_LIB mscore_freetype)
endif (USE_SYSTEM_FREETYPE)

set(LINK_LIB
    ${FREETYPE_LIB}
    ${QT_LIBRARIES}
    global
    ui
    uicomponents
    fonts
    system
    network
    )

if (BUILD_TELEMETRY_MODULE)
    set(LINK_LIB ${LINK_LIB} telemetry)
endif(BUILD_TELEMETRY_MODULE)

set(LINK_LIB ${LINK_LIB}
    libmscore
    actions
    appshell
    cloud
    context
    shortcuts
    workspace
    audio
    midi
    midi_old
    userscores
    extensions
    languages
    importexport
    notation
    commonscene
    palette
    inspector
    playback
    instruments
    plugins
    )

if (BUILD_VST)
    set(LINK_LIB ${LINK_LIB} vst)
endif(BUILD_VST)

add_dependencies(mscore workspaces)

target_link_libraries(mscore ${LINK_LIB} )

if (APPLE)
  target_link_libraries(mscore ${OsxFrameworks})
else (APPLE)
#    target_link_libraries(mscore rt) # TODO What is it?
endif (APPLE)


if (MSCORE_OUTPUT_NAME)
    set(MSCORE_EXECUTABLE_NAME ${MSCORE_OUTPUT_NAME})
else (MSCORE_OUTPUT_NAME)
    set(MSCORE_EXECUTABLE_NAME ${EXECUTABLE_NAME})
endif (MSCORE_OUTPUT_NAME)
set(MSCORE_EXECUTABLE_NAME "${MSCORE_EXECUTABLE_NAME}" PARENT_SCOPE)


# === INSTALL ===

# ===================
# ===== Windows =====
# ===================
if (OS_IS_WIN)

    install(TARGETS ${EXECUTABLE_NAME} RUNTIME DESTINATION bin )

    include(GetCompilerInfo)

    if (CC_IS_MINGW)
        get_filename_component(COMPILER_DIR ${CMAKE_CXX_COMPILER} DIRECTORY)
        get_filename_component (MINGW_ROOT ${COMPILER_DIR} DIRECTORY)

        install( FILES
               ${MINGW_ROOT}/bin/libgcc_s_seh-1.dll
               ${MINGW_ROOT}/bin/libstdc++-6.dll
               ${MINGW_ROOT}/bin/libwinpthread-1.dll
               DESTINATION bin)

    endif(CC_IS_MINGW)

    # Install Qt
    foreach (QtLibrary ${QT_LIBRARIES})
        #message(STATUS "Library ${QtLibrary}")
        # always use release libs
        set_target_properties(${QtLibrary} PROPERTIES MAP_IMPORTED_CONFIG_DEBUG "RELEASE")
        get_target_property(QtSharedLibrary ${QtLibrary} LOCATION_RELEASE)
        if (EXISTS ${QtSharedLibrary})
            list (APPEND QtInstallLibraries ${QtSharedLibrary})
        endif (EXISTS ${QtSharedLibrary})
    endforeach (QtLibrary ${QT_LIBRARIES})
    list(REMOVE_DUPLICATES QtInstallLibraries)

    install(FILES
            ${PROJECT_SOURCE_DIR}/build/qt.conf
            ${QtInstallLibraries}
            ${QT_INSTALL_BINS}/libEGL.dll
            ${QT_INSTALL_BINS}/libGLESv2.dll
            ${QT_INSTALL_BINS}/opengl32sw.dll
            ${QT_INSTALL_BINS}/d3dcompiler_47.dll
            ${QT_INSTALL_BINS}/Qt5QmlModels.dll
            ${QT_INSTALL_BINS}/Qt5QmlWorkerScript.dll
            DESTINATION bin)

    install(FILES
            ${QT_INSTALL_PLUGINS}/iconengines/qsvgicon.dll
            DESTINATION bin/iconengines)

    install(FILES
            ${QT_INSTALL_PLUGINS}/imageformats/qjpeg.dll
            ${QT_INSTALL_PLUGINS}/imageformats/qsvg.dll
            ${QT_INSTALL_PLUGINS}/imageformats/qtiff.dll
            DESTINATION bin/imageformats)


    install(FILES
            ${QT_INSTALL_PLUGINS}/platforms/qwindows.dll
            DESTINATION bin/platforms)

    install(FILES
            ${QT_INSTALL_PLUGINS}/printsupport/windowsprintersupport.dll
            DESTINATION bin/printsupport)

    install(DIRECTORY
            ${QT_INSTALL_QML}
            DESTINATION .
            REGEX ".*d\\.dll" EXCLUDE
            REGEX ".pdb" EXCLUDE
            REGEX ".*QtMultimedia.*" EXCLUDE
            REGEX ".*QtSensors.*" EXCLUDE
            REGEX ".*QtTest.*" EXCLUDE
            REGEX ".*QtWebkit.*" EXCLUDE
            )

    if (USE_WEBENGINE)
        install(FILES
                ${QT_INSTALL_BINS}/Qt5WebEngineWidgets.dll
                ${QT_INSTALL_BINS}/Qt5WebEngineCore.dll
                DESTINATION bin)

        install(FILES
               ${QT_INSTALL_LIBEXECS}/QtWebEngineProcess.exe
               DESTINATION bin)

        install(DIRECTORY
               ${QT_INSTALL_DATA}/resources
               DESTINATION bin/webengineresources)

        install(DIRECTORY
               ${QT_INSTALL_TRANSLATIONS}/qtwebengine_locales
               DESTINATION bin/webengineresources/translations)

    endif(USE_WEBENGINE)

    if (SNDFILE_LIB)
        install(FILES ${DEPENDENCIES_DIR}/libsndfile-1.dll DESTINATION bin)
    endif()

    # Install ssl
    install(FILES
            ${DEPENDENCIES_DIR}/libcrypto-1_1-x64.dll
            ${DEPENDENCIES_DIR}/libssl-1_1-x64.dll
            DESTINATION bin)

    if (WIN_PORTABLE)
        # deploy the files and directory structure needed for the PortableApps.com format
        install(DIRECTORY ${PROJECT_SOURCE_DIR}/build/PortableApps/App DESTINATION ${CMAKE_INSTALL_PREFIX}/../..)
        install(DIRECTORY ${PROJECT_SOURCE_DIR}/build/PortableApps/Other DESTINATION ${CMAKE_INSTALL_PREFIX}/../..)
        install(FILES ${PROJECT_SOURCE_DIR}/build/PortableApps/help.html DESTINATION ${CMAKE_INSTALL_PREFIX}/../..)
        set(MSCORE_ICON_BASE ${PROJECT_SOURCE_DIR}/assets/musescore-icon-round)
        configure_file(${PROJECT_SOURCE_DIR}/build/PortableApps/appinfo.ini.in     ${CMAKE_INSTALL_PREFIX}/../../App/AppInfo/appinfo.ini   @ONLY)
        install(FILES ${PROJECT_SOURCE_DIR}/src/main/res/mscore.ico RENAME appicon.ico DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo)
        install(FILES ${MSCORE_ICON_BASE}-16.png RENAME appicon_16.png DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo)
        install(FILES ${MSCORE_ICON_BASE}-32.png RENAME appicon_32.png DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo)
        install(FILES ${MSCORE_ICON_BASE}-128.png RENAME appicon_128.png DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo)
        install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscz-icon.ico RENAME mscz.ico DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # icon for .MSCZ files
        install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscz-icon-48.png RENAME mscz_48.png DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # PNG icon for .MSCZ files
        install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscx-icon.ico RENAME mscx.ico DESTINATION ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # icon for .MSCX files
        install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscx-icon-48.png RENAME mscx_48.png DESTINATION  ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # PNG icon for .MSCX files
        # use a custom icon for MusicXML files (there isn't a standard icon for MusicXML files)
        # install( FILES   ${PROJECT_SOURCE_DIR}/assets/mxl-icon.svg RENAME mxl.svg DESTINATION  ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # SVG icon for .MXL (compressed MusicXML) files
        # install( FILES   ${PROJECT_SOURCE_DIR}/assets/mxl-icon-48.png RENAME mxl_48.png DESTINATION  ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # PNG icon for .MXL (compressed MusicXML) files
        # install( FILES   ${PROJECT_SOURCE_DIR}/assets/xml-icon.svg RENAME xml.svg DESTINATION  ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # SVG icon for .XML (MusicXML) files
        # install( FILES   ${PROJECT_SOURCE_DIR}/assets/xml-icon-48.png RENAME xml_48.png DESTINATION  ${CMAKE_INSTALL_PREFIX}/../AppInfo/FileTypeIcons) # PNG icon for .XML (MusicXML) files
    endif (WIN_PORTABLE)

# =================
# ===== Linux =====
# =================
elseif(OS_IS_LIN)

    install(TARGETS ${EXECUTABLE_NAME} RUNTIME DESTINATION bin )
    # Install mscore executable (package maintainers may add "MuseScore" and/or "musescore" aliases that symlink to mscore)
    if (LN_EXECUTABLE)
       add_custom_target(mscore_alias ALL
           COMMAND echo "Creating symlink alias for mscore executable."
           COMMAND ${LN_EXECUTABLE} -sf "mscore${MSCORE_INSTALL_SUFFIX}" "musescore${MSCORE_INSTALL_SUFFIX}"
           COMMAND echo 'Symlink alias: musescore${MSCORE_INSTALL_SUFFIX} -> mscore${MSCORE_INSTALL_SUFFIX}'
           )
       install(FILES ${PROJECT_BINARY_DIR}/src/main/musescore${MSCORE_INSTALL_SUFFIX} DESTINATION bin)
    else (LN_EXECUTABLE)
       add_custom_target(mscore_alias ALL COMMAND echo "No symlink aliases will be created." VERBATIM )
    endif (LN_EXECUTABLE)

    # Install MuseScore icons (use SVGs where possible, but install PNGs as backup for systems that don't support SVG)
    set(MSCORE_ICON_BASE ${PROJECT_SOURCE_DIR}/assets/musescore-icon-round)
    install(FILES ${MSCORE_ICON_BASE}.svg RENAME mscore${MSCORE_INSTALL_SUFFIX}.svg DESTINATION share/icons/hicolor/scalable/apps)
    install(FILES ${MSCORE_ICON_BASE}-16.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/16x16/apps)
    install(FILES ${MSCORE_ICON_BASE}-24.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/24x24/apps)
    install(FILES ${MSCORE_ICON_BASE}-32.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/32x32/apps)
    install(FILES ${MSCORE_ICON_BASE}-48.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/48x48/apps)
    install(FILES ${MSCORE_ICON_BASE}-64.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/64x64/apps)
    install(FILES ${MSCORE_ICON_BASE}-96.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/96x96/apps)
    install(FILES ${MSCORE_ICON_BASE}-128.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/128x128/apps)
    install(FILES ${MSCORE_ICON_BASE}-512.png RENAME mscore${MSCORE_INSTALL_SUFFIX}.png DESTINATION share/icons/hicolor/512x512/apps)
    # Install MIME (filetype) icons for each mimetype on Linux
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscz-icon.svg RENAME application-x-musescore${MSCORE_INSTALL_SUFFIX}.svg
       DESTINATION share/icons/hicolor/scalable/mimetypes) # SVG icon for .MSCZ files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscz-icon-48.png RENAME application-x-musescore${MSCORE_INSTALL_SUFFIX}.png
       DESTINATION share/icons/hicolor/48x48/mimetypes) # PNG icon for .MSCZ files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscx-icon.svg RENAME application-x-musescore${MSCORE_INSTALL_SUFFIX}+xml.svg
       DESTINATION share/icons/hicolor/scalable/mimetypes) # SVG icon for .MSCX files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mscx-icon-48.png RENAME application-x-musescore${MSCORE_INSTALL_SUFFIX}+xml.png
       DESTINATION share/icons/hicolor/48x48/mimetypes) # PNG icon for .MSCX files
    # use a custom icon for MusicXML files (there isn't a standard icon for MusicXML files)
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mxl-icon.svg RENAME application-vnd.recordare.musicxml${MSCORE_INSTALL_SUFFIX}.svg
       DESTINATION share/icons/hicolor/scalable/mimetypes) # SVG icon for .MXL (compressed MusicXML) files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/mxl-icon-48.png RENAME application-vnd.recordare.musicxml${MSCORE_INSTALL_SUFFIX}.png
       DESTINATION share/icons/hicolor/48x48/mimetypes) # PNG icon for .MXL (compressed MusicXML) files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/xml-icon.svg RENAME application-vnd.recordare.musicxml${MSCORE_INSTALL_SUFFIX}+xml.svg
       DESTINATION share/icons/hicolor/scalable/mimetypes) # SVG icon for .XML (MusicXML) files
    install( FILES   ${PROJECT_SOURCE_DIR}/assets/xml-icon-48.png RENAME application-vnd.recordare.musicxml${MSCORE_INSTALL_SUFFIX}+xml.png
       DESTINATION share/icons/hicolor/48x48/mimetypes) # PNG icon for .XML (MusicXML) files
    # Note: Must now run "gtk-update-icon-cache" to set the new icons. This is done in the Makefile.

# =================
# ===== MacOS =====
# =================
elseif(OS_IS_MAC)

    install(TARGETS mscore BUNDLE DESTINATION ${CMAKE_INSTALL_PREFIX})
    install(FILES "${PROJECT_SOURCE_DIR}/src/main/res/mscore.icns" DESTINATION ${Mscore_SHARE_NAME}${Mscore_INSTALL_NAME})
    install(FILES "${PROJECT_SOURCE_DIR}/src/main/res/musescoreDocument.icns" DESTINATION ${Mscore_SHARE_NAME}${Mscore_INSTALL_NAME})

    install(FILES
            ${PROJECT_SOURCE_DIR}/fonts/gootville/GootvilleText.otf
            ${PROJECT_SOURCE_DIR}/fonts/mscore/MScoreText.ttf
            ${PROJECT_SOURCE_DIR}/fonts/mscore/MusescoreIcon.ttf
            ${PROJECT_SOURCE_DIR}/fonts/musejazz/MuseJazzText.otf
            ${PROJECT_SOURCE_DIR}/fonts/campania/Campania.otf
            ${PROJECT_SOURCE_DIR}/fonts/FreeSerif.ttf
            ${PROJECT_SOURCE_DIR}/fonts/FreeSerifBold.ttf
            ${PROJECT_SOURCE_DIR}/fonts/FreeSerifItalic.ttf
            ${PROJECT_SOURCE_DIR}/fonts/FreeSerifBoldItalic.ttf
            ${PROJECT_SOURCE_DIR}/fonts/FreeSans.ttf
            ${PROJECT_SOURCE_DIR}/fonts/mscoreTab.ttf
            ${PROJECT_SOURCE_DIR}/fonts/mscore-BC.ttf
            ${PROJECT_SOURCE_DIR}/fonts/bravura/BravuraText.otf
            ${PROJECT_SOURCE_DIR}/fonts/leland/Leland.otf
            ${PROJECT_SOURCE_DIR}/fonts/petaluma/PetalumaText.otf
            ${PROJECT_SOURCE_DIR}/fonts/petaluma/PetalumaScript.otf
            DESTINATION ${Mscore_SHARE_NAME}${Mscore_INSTALL_NAME}fonts
    )

    install(DIRECTORY
            ${QT_INSTALL_QML}
            DESTINATION ${Mscore_SHARE_NAME}${Mscore_INSTALL_NAME}
            REGEX ".*QtWebkit.*" EXCLUDE
            REGEX ".*QtTest.*" EXCLUDE
            REGEX ".*QtSensors.*" EXCLUDE
            REGEX ".*QtMultimedia.*" EXCLUDE
            REGEX ".*QtAudioEngine.*" EXCLUDE
            REGEX ".*_debug\\.dylib" EXCLUDE)

else()
    message(FATAL_ERROR "Unsopported Platform: ${CMAKE_HOST_SYSTEM_NAME}")
endif()



##
## Miscellaneous Microsoft Visual Studio settings
##
if (MSVC)

   # Force the "install" and "package" targets not to depend on the "all" target.
   set(CMAKE_SKIP_INSTALL_ALL_DEPENDENCY true)
   set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY true)

   # Set the startup project to "mscore".
   if (NOT ${CMAKE_VERSION} VERSION_LESS "3.6.0")
      set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT mscore)
   endif ()

   # Set the debugging properties for the "mscore" project.
   file(TO_NATIVE_PATH "${CMAKE_INSTALL_PREFIX}/bin" VS_DEBUGGER_WORKING_DIRECTORY)
   if (NOT ${CMAKE_VERSION} VERSION_LESS "3.12.0")
      set_target_properties(${EXECUTABLE_NAME} PROPERTIES VS_DEBUGGER_COMMAND "${VS_DEBUGGER_WORKING_DIRECTORY}\\${MSCORE_EXECUTABLE_NAME}${CMAKE_EXECUTABLE_SUFFIX}")
   endif ()
   if (NOT ${CMAKE_VERSION} VERSION_LESS "3.8.0")
      set_target_properties(${EXECUTABLE_NAME} PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "${VS_DEBUGGER_WORKING_DIRECTORY}")
   endif ()
   if (NOT ${CMAKE_VERSION} VERSION_LESS "3.13.0")
      set_target_properties(${EXECUTABLE_NAME} PROPERTIES VS_DEBUGGER_COMMAND_ARGUMENTS "--debug")
   endif ()

endif (MSVC)
