PROJECT (QtTest)
cmake_minimum_required(VERSION 2.8)

set(CMAKE_AUTOMOC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

#set(CMAKE_INSTALL_RPATH "$ORIGIN")
#SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)

SET( QTTEST_SRCS
  main.cpp
  TestForm.h TestForm.cpp
)

SET( QTTEST_MOC_HDRS
)

SET( QTTEST_UIS
     TestForm.ui
)

SET( QTTEST_RCS
)

FIND_PACKAGE( Qt5Widgets REQUIRED )
find_package(Qt5Sql REQUIRED)

qt5_add_resources( QTTEST_GEN_RCS ${QTTEST_RCS} )
qt5_wrap_ui( QTTEST_GEN_UIS ${QTTEST_UIS} )
qt5_wrap_cpp( QTTEST_GEN_MOC_HDRS ${QTTEST_MOC_HDRS} )

# For Apple set the icns file containing icons
IF(APPLE)
  # set how it shows up in the Info.plist file
  SET(MACOSX_BUNDLE_ICON_FILE QtTest.icns) 
  # set where in the bundle to put the icns file
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/QtTest.icns PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
  # include the icns file in the target
  SET(QTTEST_SRCS ${QTTEST_SRCS} ${CMAKE_CURRENT_SOURCE_DIR}/QtTest.icns)
ENDIF(APPLE)

set(CMAKE_INSTALL_RPATH "$ORIGIN")
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)
#?
#LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/bin/plugins/platforms" isSystemDir)
#IF("${isSystemDir}" STREQUAL "-1")
#   SET(CMAKE_INSTALL_RPATH "$ORIGIN/../../")
#ENDIF("${isSystemDir}" STREQUAL "-1")
message("CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES: " ${CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES})
#


if(WIN32)
    ADD_EXECUTABLE( QtTest
        ${QTTEST_SRCS}
        ${QTTEST_GEN_MOC_HDRS} 
        ${QTTEST_GEN_UIS}
        ${QTTEST_GEN_RCS}
    )
else()
    ADD_EXECUTABLE( QtTest MACOSX_BUNDLE 
        ${QTTEST_SRCS}
        ${QTTEST_GEN_MOC_HDRS} 
        ${QTTEST_GEN_UIS}
        ${QTTEST_GEN_RCS}
    )
endif()

#qt5_use_modules(QtTest Widgets)

if(Win32)
    TARGET_LINK_LIBRARIES ( QtTest  Qt5::Widgets Qt5::WinMain )
else()
	target_link_libraries(QtTest Qt5::Widgets)
endif()


#--------------------------------------------------------------------------------
# Now the installation stuff below
#--------------------------------------------------------------------------------

##set(CMAKE_INSTALL_RPATH "$ORIGIN")
##SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)


SET(plugin_dest_dir bin)
SET(qtconf_dest_dir bin)
SET(APPS "\${CMAKE_INSTALL_PREFIX}/bin/QtTest")
IF(APPLE)
  SET(plugin_dest_dir QtTest.app/Contents/MacOS)
  SET(qtconf_dest_dir QtTest.app/Contents/Resources)
  SET(APPS "\${CMAKE_INSTALL_PREFIX}/QtTest.app")
ENDIF(APPLE)
IF(WIN32)
  SET(APPS "\${CMAKE_INSTALL_PREFIX}/bin/QtTest.exe")
ENDIF(WIN32)

#--------------------------------------------------------------------------------
# Install the QtTest application, on Apple, the bundle is at the root of the
# install tree, and on other platforms it'll go into the bin directory.
INSTALL(TARGETS QtTest 
    BUNDLE DESTINATION . COMPONENT Runtime
    RUNTIME DESTINATION bin COMPONENT Runtime
    )

#test-remove

message("Qt5Gui plugins: ")
foreach(plg ${Qt5Gui_PLUGINS})
	get_target_property(plg_loc ${plg} LOCATION)
	message(" - " ${plg} ": " ${plg_loc})
	##install(FILES ${plg_loc} DESTINATION ${plugin_dest_dir}/plugins COMPONENT Runtime)
endforeach()

get_target_property(QT_QMAKE_EXECUTABLE ${Qt5Core_QMAKE_EXECUTABLE} IMPORTED_LOCATION)
if(NOT QT_QMAKE_EXECUTABLE)
        message(FATAL_ERROR "qmake is not found.")
endif()
message("QT_QMAKE_EXECUTABLE: " ${QT_QMAKE_EXECUTABLE})
execute_process(COMMAND ${QT_QMAKE_EXECUTABLE} -query QT_INSTALL_PLUGINS
        OUTPUT_VARIABLE QT_PLUGINS_DIR
        OUTPUT_STRIP_TRAILING_WHITESPACE
        )
message("QT_PLUGINS_DIR: " ${QT_PLUGINS_DIR})

get_target_property(QT_LIBRARY_DIRS Qt5::Core LOCATION)
get_filename_component(QT_LIBRARY_DIRS ${QT_LIBRARY_DIRS} DIRECTORY)
message("QT_LIBRARY_DIRS: " ${QT_LIBRARY_DIRS})


#
    
#--------------------------------------------------------------------------------
# Install needed Qt plugins by copying directories from the qt installation
# One can cull what gets copied by using 'REGEX "..." EXCLUDE'
##INSTALL(DIRECTORY "${QT_PLUGINS_DIR}/imageformats" DESTINATION ${plugin_dest_dir}/plugins COMPONENT Runtime)
install(DIRECTORY "${QT_PLUGINS_DIR}/platforms" DESTINATION ${plugin_dest_dir}/plugins COMPONENT Runtime)
##install(DIRECTORY "${QT_PLUGINS_DIR}/sqldrivers" DESTINATION ${plugin_dest_dir}/plugins COMPONENT Runtime)

#--------------------------------------------------------------------------------
# install a qt.conf file
# this inserts some cmake code into the install script to write the file
INSTALL(CODE "
    file(WRITE \"\${CMAKE_INSTALL_PREFIX}/${qtconf_dest_dir}/qt.conf\" \"\")
    " COMPONENT Runtime)


#--------------------------------------------------------------------------------
# Use BundleUtilities to get all other dependencies for the application to work.
# It takes a bundle or executable along with possible plugins and inspects it
# for dependencies.  If they are not system dependencies, they are copied.

# directories to look for dependencies
SET(DIRS ${QT_LIBRARY_DIRS})

#test-remove
message("CMAKE_SHARED_LIBRARY_SUFFIX: " ${CMAKE_SHARED_LIBRARY_SUFFIX})
file(GLOB_RECURSE QTPLUGINS
      ${CMAKE_INSTALL_PREFIX}/${plugin_dest_dir}/plugins/*${CMAKE_SHARED_LIBRARY_SUFFIX})
message("QTPLUGINS: ")
foreach(plg ${QTPLUGINS})
    message("- " ${plg})
endforeach()

# Now the work of copying dependencies into the bundle/package
# The quotes are escaped and variables to use at install time have their $ escaped
# An alternative is the do a configure_file() on a script and use install(SCRIPT  ...).
# Note that the image plugins depend on QtSvg and QtXml, and it got those copied
# over.
INSTALL(CODE "
	file(GLOB_RECURSE QTPLUGINS
	  \"\${CMAKE_INSTALL_PREFIX}/${plugin_dest_dir}/plugins/*${CMAKE_SHARED_LIBRARY_SUFFIX}\")
	include(BundleUtilities)
	if(UNIX AND NOT APPLE)
		foreach(qtplg \${QTPLUGINS})
			file(RPATH_REMOVE FILE \"\${qtplg}\")
		endforeach()
	endif()
	fixup_bundle(\"${APPS}\" \"\${QTPLUGINS}\" \"${DIRS}\")
	" COMPONENT Runtime)
#install(CODE "
#    include(BundleUtilities)
#    fixup_bundle(\"${APPS}\" \"${QTPLUGINS}\" \"${DIRS}\")
#    " COMPONENT Runtime)


# To Create a package, one can run "cpack -G DragNDrop CPackConfig.cmake" on Mac OS X
# where CPackConfig.cmake is created by including CPack
# And then there's ways to customize this as well
set(CPACK_BINARY_DRAGNDROP OFF)
include(CPack)

