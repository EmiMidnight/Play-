cmake_minimum_required(VERSION 3.15)
set(PROJECT_Version 1.0.0.11)
string(REPLACE "." "," PROJECT_Version_RC ${PROJECT_Version})
# macOS deployment target needs to be set before 'project' to work
if(APPLE AND NOT TARGET_IOS)
	set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "Minimum OS X deployment version")
endif()

project(Play)

include(GNUInstallDirs)

set(CMAKE_MODULE_PATH
	${CMAKE_CURRENT_SOURCE_DIR}/deps/Dependencies/cmake-modules
	${CMAKE_MODULE_PATH}
)
include(Header)

set(BUILD_PLAY ON CACHE BOOL "Build Play! Emulator")
set(BUILD_PSFPLAYER OFF CACHE BOOL "Build PsfPlayer")
set(BUILD_TESTS ON CACHE BOOL "Build Tests")
set(USE_AOT_CACHE OFF CACHE BOOL "Use AOT block cache")
set(BUILD_AOT_CACHE OFF CACHE BOOL "Build AOT block cache (for PsfPlayer only)")
set(BUILD_LIBRETRO_CORE OFF CACHE BOOL "Build Libretro Core")

set(PROJECT_NAME "Play!")

if(BUILD_TESTS)
	enable_testing()
endif()

find_package(Git)
# if(GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git/")
# 	execute_process(COMMAND ${GIT_EXECUTABLE} describe --always
# 		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
# 		RESULT_VARIABLE GIT_DESCRIBE_RESULT
# 		OUTPUT_VARIABLE GIT_TAG
# 		OUTPUT_STRIP_TRAILING_WHITESPACE
# 	)
# 	if(GIT_DESCRIBE_RESULT EQUAL 0)
# 		message("Building for ${GIT_TAG}.")
# 		set(PROJECT_Version ${GIT_TAG})
# 	else()
# 		message("Warning: Failed to obtain Git tag.")
# 	endif()
# else()
# 	message("Warning: Not a git repo or git not found.")
# endif()

add_definitions(-DPLAY_VERSION="${PROJECT_Version}")
add_definitions(-DPROJECT_Version_RC=${PROJECT_Version_RC})
set(PROJECT_LIBS)

if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    # Enable Link-Time Optimization
    include(CheckIPOSupported)
    check_ipo_supported(RESULT supported OUTPUT error)
    
    if(supported)
				set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
        message(STATUS "IPO/LTO enabled")
    else()
        message(STATUS "IPO/LTO not supported: <${error}>")
    endif()

		if(MSVC)
        add_compile_options(
            $<$<CONFIG:Release>:/arch:SSE4.2>
            $<$<CONFIG:Release>:/O2>
            $<$<CONFIG:Release>:/GL>
            $<$<CONFIG:Release>:/fp:fast>
            $<$<CONFIG:RelWithDebInfo>:/arch:SSE4.2>
            $<$<CONFIG:RelWithDebInfo>:/O2>
            $<$<CONFIG:RelWithDebInfo>:/GL>
            $<$<CONFIG:RelWithDebInfo>:/fp:fast>
        )
        add_link_options(/LTCG:INCREMENTAL)
        message(STATUS "MSVC optimizations enabled globally")
    endif()
endif()


include(PrecompiledHeader)

if(BUILD_LIBRETRO_CORE)
	if(NOT MSVC)
		add_compile_options($<$<NOT:$<COMPILE_LANGUAGE:Swift>>:-fPIC>)
	endif()
	add_subdirectory(Source/ui_libretro)
endif()

if(BUILD_PLAY)
	if(TARGET_PLATFORM_UNIX OR TARGET_PLATFORM_MACOS OR TARGET_PLATFORM_WIN32 OR USE_QT)
		add_subdirectory(Source/ui_qt/)
		add_subdirectory(tools/ElfView)
	endif(TARGET_PLATFORM_UNIX OR TARGET_PLATFORM_MACOS OR TARGET_PLATFORM_WIN32 OR USE_QT)

	if(NOT USE_QT)
		if(TARGET_PLATFORM_IOS)
			add_subdirectory(Source/ui_ios/)
		endif(TARGET_PLATFORM_IOS)

		if(TARGET_PLATFORM_ANDROID)
			add_subdirectory(Source/ui_android/)
		endif(TARGET_PLATFORM_ANDROID)

		if(TARGET_PLATFORM_JS)
			add_subdirectory(Source/ui_js)
		endif(TARGET_PLATFORM_JS)
	endif()
endif(BUILD_PLAY)

if(BUILD_TESTS)
	add_subdirectory(tools/AutoTest/)
	add_subdirectory(tools/GsAreaTest/)
	add_subdirectory(tools/McServTest/)
	add_subdirectory(tools/SpuTest/)
	add_subdirectory(tools/VuTest/)
	add_subdirectory(deps/Framework/build_cmake/Tests)
endif()

add_subdirectory(tools/NamcoSys147NANDTools)

if(BUILD_PSFPLAYER)
	add_subdirectory(tools/PsfPlayer)
endif(BUILD_PSFPLAYER)