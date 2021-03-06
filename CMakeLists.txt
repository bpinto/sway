cmake_minimum_required(VERSION 3.1.0)

project(sway C)

add_compile_options(-g)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wno-unused-result -Werror)

# Add Address Sanitiezed build type
set(CMAKE_C_FLAGS_ASAN
    "${CMAKE_C_FLAGS_DEBUG} -fsanitize=address -fno-omit-frame-pointer"
    CACHE STRING "Flags used by the C compiler during address sanitizer builds."
    FORCE )
mark_as_advanced(
    CMAKE_C_FLAGS_ASAN
    CMAKE_EXE_LINKER_FLAGS_DEBUG
    CMAKE_SHARED_LINKER_FLAGS_DEBUG
    )

list(INSERT CMAKE_MODULE_PATH 0
	${CMAKE_CURRENT_SOURCE_DIR}/CMake
	)

if (VERSION)
	add_definitions(-DSWAY_VERSION=\"${VERSION}\")
else()
	execute_process(
		COMMAND git describe --always --tags
		OUTPUT_VARIABLE GIT_COMMIT_HASH
		OUTPUT_STRIP_TRAILING_WHITESPACE
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
		)
	execute_process(
		COMMAND git rev-parse --abbrev-ref HEAD
		OUTPUT_VARIABLE GIT_BRANCH
		OUTPUT_STRIP_TRAILING_WHITESPACE
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
		)

	string(TIMESTAMP CURRENT_DATE "%Y-%m-%d" UTC)
	add_definitions("-DSWAY_VERSION=\"${GIT_COMMIT_HASH} (${CURRENT_DATE}, branch \\\"${GIT_BRANCH}\\\")\"")
endif()

option(enable-swaylock "Enables the swaylock utility" YES)
option(enable-swaybg "Enables the wallpaper utility" YES)
option(enable-swaybar "Enables the swaybar utility" YES)
option(enable-swaygrab "Enables the swaygrab utility" YES)
option(enable-swaymsg "Enables the swaymsg utility" YES)
option(enable-gdk-pixbuf "Use Pixbuf to support more image formats" YES)
option(enable-tray "Enables the swaybar tray" YES)
option(zsh-completions "Zsh shell completions" NO)
option(default-wallpaper "Installs the default wallpaper" YES)
option(LD_LIBRARY_PATH "Configure sway's default LD_LIBRARY_PATH")

if (LD_LIBRARY_PATH)
	add_definitions(-D_LD_LIBRARY_PATH="${LD_LIBRARY_PATH}")
endif()

find_package(JsonC 0.12.1 REQUIRED)
find_package(PCRE REQUIRED)
find_package(WLC REQUIRED)
find_package(Wayland REQUIRED)
find_package(XKBCommon REQUIRED)
find_package(Cairo REQUIRED)
find_package(Pango REQUIRED)
find_package(GdkPixbuf)
find_package(PAM)
find_package(DBus 1.10)

find_package(LibInput REQUIRED)

if (CMAKE_SYSTEM_NAME STREQUAL Linux)
	find_package(Libcap REQUIRED)
endif (CMAKE_SYSTEM_NAME STREQUAL Linux)

if (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
	find_package(EpollShim REQUIRED)
endif (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)

include(FeatureSummary)
include(Manpage)
include(GNUInstallDirs)

if (enable-gdk-pixbuf)
	if (GDK_PIXBUF_FOUND)
		set(WITH_GDK_PIXBUF YES)
		add_definitions(-DWITH_GDK_PIXBUF)
	else()
		message(WARNING "gdk-pixbuf required but not found, only png images supported.")
	endif()
else()
	message(STATUS "Building without gdk-pixbuf, only png images supported.")
endif()

if (enable-tray)
	if (DBUS_FOUND)
		set(ENABLE_TRAY TRUE)
		add_definitions(-DENABLE_TRAY)
	else()
		message(WARNING "Tray required but DBus was not found. Tray will not be included")
	endif()
else()
	message(STATUS "Building without the tray.")
endif()

include_directories(include)

add_subdirectory(protocols)
add_subdirectory(common)
add_subdirectory(wayland)

add_subdirectory(sway)
if(enable-swaybg)
	if(CAIRO_FOUND AND PANGO_FOUND)
		add_subdirectory(swaybg)
	else()
		message(WARNING "Not building swaybg - cairo, and pango are required.")
	endif()
endif()
if(enable-swaymsg)
	add_subdirectory(swaymsg)
endif()
if(enable-swaygrab)
	add_subdirectory(swaygrab)
endif()
if(enable-swaybar)
	if(CAIRO_FOUND AND PANGO_FOUND)
		add_subdirectory(swaybar)
	else()
		message(WARNING "Not building swaybar - cairo, and pango  are required.")
	endif()
endif()
if(enable-swaylock)
	if(CAIRO_FOUND AND PANGO_FOUND AND PAM_FOUND)
		add_subdirectory(swaylock)
	else()
		message(WARNING "Not building swaylock - cairo, pango, and PAM are required.")
	endif()
endif()
if(zsh-completions)
	add_subdirectory(completions/zsh)
endif()
install(
	FILES ${CMAKE_CURRENT_SOURCE_DIR}/sway.desktop
	DESTINATION ${CMAKE_INSTALL_FULL_DATAROOTDIR}/wayland-sessions
	COMPONENT data
	)

if(default-wallpaper)
	install(
        DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/assets/
		DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/backgrounds/sway
		COMPONENT data
        FILES_MATCHING PATTERN "*Wallpaper*"
		)
endif()

feature_summary(WHAT ALL)
