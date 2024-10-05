cmake_minimum_required (VERSION 3.0.2)

project (far2l)

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules)

set(APP_NAME "far2l")
set(APP_IDENTIFIER "com.far2l")
set(CPACK_PACKAGE_NAME "far2l")

string(ASCII 27 Esc)
set(ColorNormal "${Esc}[m")
set(ColorRed "${Esc}[1;31m")

include(GNUInstallDirs)

# just to force reconfigure if packaging/version changed
configure_file(${PROJECT_SOURCE_DIR}/packaging/version ${CMAKE_BINARY_DIR}/packaging/version @ONLY)

# reconfigure in case .git directory changed to update version correctly
IF(EXISTS ${PROJECT_SOURCE_DIR}/.git)
    set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS ${PROJECT_SOURCE_DIR}/.git)
endif()

file(READ "${PROJECT_SOURCE_DIR}/packaging/version" VERSION)
string(REGEX REPLACE "[\r\n\t ]$" "" VERSION "${VERSION}")
string(REGEX REPLACE "^([0-9]+)\\..*" "\\1" VERSION_MAJOR "${VERSION}")
string(REGEX REPLACE "^[0-9]+\\.([0-9]+).*" "\\1" VERSION_MINOR "${VERSION}")
string(REGEX REPLACE "^[0-9]+\\.[0-9]+\\.(.*)" "\\1" VERSION_PATCH "${VERSION}")

if("${VERSION_MAJOR}" STREQUAL "" OR "${VERSION_MAJOR}" STREQUAL "${VERSION}"
    OR "${VERSION_MINOR}" STREQUAL "" OR "${VERSION_MINOR}" STREQUAL "${VERSION}"
    OR "${VERSION_PATCH}" STREQUAL "" OR "${VERSION_PATCH}" STREQUAL "${VERSION}")
        message(FATAL_ERROR "${ColorRed}packaging/version is corrupted, please write actual X.Y.Z version into it.${ColorNormal}")
endif()

find_package(Git)

if(GIT_FOUND)
    execute_process(COMMAND "${GIT_EXECUTABLE}" describe --tag
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
        OUTPUT_VARIABLE GIT_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()

if(NOT "${GIT_TAG}" STREQUAL "v_${VERSION}")
    if(GIT_FOUND)
        if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/.git)
            execute_process(COMMAND "${GIT_EXECUTABLE}" show --no-patch --format=%ci HEAD
                WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
                OUTPUT_VARIABLE GIT_DATE
                OUTPUT_STRIP_TRAILING_WHITESPACE)
            string(SUBSTRING ${GIT_DATE} 0 10 GIT_DATE)
            execute_process(COMMAND "${GIT_EXECUTABLE}" rev-parse --short HEAD
                WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
                OUTPUT_VARIABLE GIT_HASH
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        endif()
    endif()
    if(NOT "${GIT_DATE}" STREQUAL "")
        message(STATUS "git date: ${GIT_DATE}")
        set(VERSION_PATCH "${VERSION_PATCH}-${GIT_DATE}")
    endif()
    if(NOT "${GIT_HASH}" STREQUAL "")
        message(STATUS "git hash: ${GIT_HASH}")
        set(VERSION_PATCH "${VERSION_PATCH}-${GIT_HASH}")
    else()
        message(WARNING "git information unavailable, assuming its a build from v_${VERSION}")
    endif()
endif()

set(VERSION_PATCH "${VERSION_PATCH}-beta")

set(VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release")
endif()

message(STATUS "Build Type: ${CMAKE_BUILD_TYPE} Version: ${VERSION}")

if(NOT DEFINED CMAKE_INSTALL_PREFIX)
   if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
      set(CMAKE_INSTALL_PREFIX "/usr/local")
   elseif (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
      set(CMAKE_INSTALL_PREFIX "/usr/local")
   else()
      set(CMAKE_INSTALL_PREFIX "/usr")
   endif()
endif()

##############################

if (NOT DEFINED TESTING)
    set(TESTING "NO")
endif()

if (NOT DEFINED USEWX)
    set(USEWX "YES")
endif()

if (NOT DEFINED UNRAR)
    set(UNRAR "bundled")
endif()

# EACP disabled by default for now as FAR's GetCodePageInfo rejects all multibyte charsets due to BUGBUG.
# Uncomment EACP-stuff here when that will be fixed. See https://github.com/elfmz/far2l/issues/1179
#if (NOT DEFINED EACP)
#   set(EACP "YES")
#endif()

if (NOT DEFINED LEGACY)
    # TODO: default it to NO  somewhere in 2022, after all obsolete settings
    # registry-upgrade-to-ini stuff will be safe to remove
    set(LEGACY "YES")
elseif(FARFTP)
    set(LEGACY "YES")
    message(STATUS "Force-enable LEGACY due to enabled FARFTP")
endif()

#if (NOT EACP)
#   message(STATUS "${ColorRed}Building without East Asian codepages support due to EACP=${EACP}${ColorNormal}")
#endif()

# set(RM rm) actually not needed: use $(CMAKE_COMMAND) -E remove

if(MUSL)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__MUSL__")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D__MUSL__")
endif()

if(TAR_LIMITED_ARGS)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__TAR_LIMITED_ARGS__")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D__TAR_LIMITED_ARGS__")
endif()

if(${CMAKE_VERSION} VERSION_LESS "3.1.0")
    message("Please consider to switch to newer CMake")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fPIC -Wno-unused-function -D_FILE_OFFSET_BITS=64") #  -fsanitize=address
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -std=c99 -fPIC -Wno-unused-function -D_FILE_OFFSET_BITS=64") #  -fsanitize=address
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O2")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS} -O2")

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(APP_DIR ${CMAKE_BINARY_DIR}/install)
    set(INSTALL_DIR ${APP_DIR}/${APP_NAME}.app/Contents/MacOS)
    set(EXECUTABLE_NAME ${APP_NAME})

else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffunction-sections -fdata-sections")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffunction-sections -fdata-sections")
    set(INSTALL_DIR ${CMAKE_BINARY_DIR}/install)
    set(EXECUTABLE_NAME far2l)

    if (NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--gc-sections")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,--gc-sections")
    endif()
endif()

if (CMAKE_SYSTEM_NAME MATCHES "CYGWIN")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GNU_SOURCE -D_BSD_SOURCE -D_XOPEN_SOURCE=600")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_GNU_SOURCE -D_BSD_SOURCE -D_XOPEN_SOURCE=600")
    message(STATUS "Running under cygwin, wow!")
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Haiku")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GNU_SOURCE -D_DEFAULT_SOURCE -Wno-attributes")
endif()

set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_C_VISIBILITY_PRESET hidden)

if (LEGACY)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DWINPORT_REGISTRY")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWINPORT_REGISTRY")
    message(STATUS "Enabling registry support due to enabled LEGACY")
else()
    message(STATUS "LEGACY is not enabled, your old registry settings will be forgotten, if any")
endif()

if(CMAKE_VERSION VERSION_GREATER "3.3")
    cmake_policy(SET CMP0063 NEW)
    cmake_policy(SET CMP0057 NEW)
endif()

include_directories(utils/include)

add_subdirectory (WinPort)
add_subdirectory (utils)
add_subdirectory (far2l)

if (NOT ${USEWX})
    message(STATUS "Building without GUI/wxWidgets backend due to USEWX=${USEWX}")
else()
    message(STATUS "Building with GUI/wxWidgets backend")
    if (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
        if (DEFINED wxWidgets_CONFIG_EXECUTABLE)
            message(STATUS "Using pre-defined wx config: ${wxWidgets_CONFIG_EXECUTABLE}")

        elseif(EXISTS "/usr/local/bin/wxgtk3u-3.2-config")
            set(wxWidgets_CONFIG_EXECUTABLE "/usr/local/bin/wxgtk3u-3.2-config")
            set(wxWidgets_wxrc_EXECUTABLE "/usr/local/bin/wxrc-gtk3u-3.2")
            message(STATUS "Using known location of wx-config 3.2")

        elseif(EXISTS "/usr/local/bin/wxgtk3u-3.1-config")
            set(wxWidgets_CONFIG_EXECUTABLE "/usr/local/bin/wxgtk3u-3.1-config")
            set(wxWidgets_wxrc_EXECUTABLE "/usr/local/bin/wxrc-gtk3u-3.1")
            message(STATUS "Using known location of wx-config 3.1")

        elseif (EXISTS "/usr/local/bin/wxgtk3u-3.0-config")
            set(wxWidgets_CONFIG_EXECUTABLE "/usr/local/bin/wxgtk3u-3.0-config")
            set(wxWidgets_wxrc_EXECUTABLE "/usr/local/bin/wxrc-gtk3u-3.0")
            message(STATUS "Using known location of wx-config 3.0")

        else()
            message(STATUS "No wxgtk3u-3.*-config in expected locations")
        endif()

    elseif (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        if (EXISTS "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.2/bin/wx-config")
            # MacPorts
            set(wxWidgets_CONFIG_EXECUTABLE "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.2/bin/wx-config")
            set(wxWidgets_wxrc_EXECUTABLE "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.2/bin/wxrc")
            message(STATUS "Using known location of wx-config 3.2")

        elseif (EXISTS "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.1/bin/wx-config")
            # MacPorts
            set(wxWidgets_CONFIG_EXECUTABLE "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.1/bin/wx-config")
            set(wxWidgets_wxrc_EXECUTABLE "/opt/local/Library/Frameworks/wxWidgets.framework/Versions/wxWidgets/3.1/bin/wxrc")
            message(STATUS "Using known location of wx-config 3.1")

        else()
            message(STATUS "No wx-config in expected locations")
        endif()
    endif()

    add_subdirectory(WinPort/src/Backend/WX)
endif()

if (NOT DEFINED TTYX)
    find_package(X11)
    if(X11_FOUND)
        message(STATUS "Building with TTY X11 extensions due to X11 found")
        add_subdirectory(WinPort/src/Backend/TTY/TTYX)
    else()
        message(STATUS "Building without TTY X11 extensions due to X11 not found")
    endif()
elseif (TTYX)
    message(STATUS "Building with TTY X11 extensions due to TTYX=${TTYX}")
    add_subdirectory(WinPort/src/Backend/TTY/TTYX)
else()
    message(STATUS "Building without TTY X11 extensions due to TTYX=${TTYX}")
endif()

##############################
# common install directives

add_subdirectory (packaging)

add_subdirectory(man)
add_subdirectory(bash-completion)

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(FIXUP_SCRIPT packaging/osx/FixupBundle.cmake)
    configure_file(${FIXUP_SCRIPT}.in ${PROJECT_BINARY_DIR}/${FIXUP_SCRIPT} @ONLY)
    install(DIRECTORY ${APP_DIR}/${APP_NAME}.app DESTINATION . USE_SOURCE_PERMISSIONS COMPONENT app EXCLUDE_FROM_ALL)
    install(SCRIPT ${PROJECT_BINARY_DIR}/${FIXUP_SCRIPT} COMPONENT app EXCLUDE_FROM_ALL)
endif()

install(PROGRAMS "${INSTALL_DIR}/${EXECUTABLE_NAME}" DESTINATION "bin" RENAME far2l COMPONENT base)

install(DIRECTORY "${INSTALL_DIR}/" DESTINATION "lib/far2l" USE_SOURCE_PERMISSIONS COMPONENT base FILES_MATCHING
    PATTERN "colorer/base" EXCLUDE
    PATTERN "far2l_gui.so"
    PATTERN "far2l_ttyx.broker"
    PATTERN "plug/*.far-plug-*"
    PATTERN "plug/*.broker"
)

install(DIRECTORY "${INSTALL_DIR}/" DESTINATION "share/far2l" USE_SOURCE_PERMISSIONS COMPONENT base FILES_MATCHING
    PATTERN "${EXECUTABLE_NAME}" EXCLUDE
    PATTERN "far2ledit" EXCLUDE
    PATTERN "far2l_*" EXCLUDE
    PATTERN "*.far-plug-*" EXCLUDE
    PATTERN "*.broker" EXCLUDE
    PATTERN "python/plug/python" EXCLUDE
    PATTERN "*"
)

# setup some symlinks and remove deprecated stuff from previous installation
execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink ../../bin/far2l ${PROJECT_BINARY_DIR}/far2l/far2l_askpass)
execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink ../../bin/far2l ${PROJECT_BINARY_DIR}/far2l/far2l_sudoapp)
execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink far2l ${PROJECT_BINARY_DIR}/far2l/far2ledit)
install(FILES "${PROJECT_BINARY_DIR}/far2l/far2l_askpass" DESTINATION "lib/far2l" COMPONENT system)
install(FILES "${PROJECT_BINARY_DIR}/far2l/far2l_sudoapp" DESTINATION "lib/far2l" COMPONENT system)
install(FILES "${PROJECT_BINARY_DIR}/far2l/far2ledit" DESTINATION "bin" COMPONENT base)
install(CODE "
    execute_process(COMMAND ${CMAKE_COMMAND} -E remove -f \"\${CMAKE_INSTALL_PREFIX}/lib/far2l/Plugins/objinfo/plug/objinfo.far-plug-mb\")
    " COMPONENT system)
