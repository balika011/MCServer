
cmake_minimum_required (VERSION 2.8.2)
project (MCServer)

include_directories (SYSTEM "${PROJECT_SOURCE_DIR}/../lib/")
include_directories (SYSTEM "${PROJECT_SOURCE_DIR}/../lib/jsoncpp/include")

set(FOLDERS OSSupport HTTPServer Bindings Items Blocks Protocol Generating) 
set(FOLDERS ${FOLDERS} WorldStorage Mobs Entities Simulator UI BlockEntities)



if (NOT MSVC)
	foreach(folder ${FOLDERS})
		add_subdirectory(${folder})
	endforeach(folder)

	file(GLOB SOURCE
		"*.cpp"
	)

	list(REMOVE_ITEM SOURCE "${PROJECT_SOURCE_DIR}/StackWalker.cpp" "${PROJECT_SOURCE_DIR}/LeakFinder.cpp")

	# If building a windows version, but not using MSVC, add the resources directly to the makefile:
	if (WIN32)
		FILE(GLOB ResourceFiles
			"Resources/*.rc"
		)
		list(APPEND SOURCE "${ResourceFiles}")
	endif()


else ()

	# Generate the Bindings if they don't exist:
	if (NOT EXISTS "${PROJECT_SOURCE_DIR}/Bindings/Bindings.cpp")
		message("Bindings.cpp not found, generating now")
		set(tolua_executable ${PROJECT_SOURCE_DIR}/Bindings/tolua++.exe)
		execute_process(
			COMMAND ${tolua_executable} -L virtual_method_hooks.lua -o Bindings.cpp -H Bindings.h AllToLua.pkg
			WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/Bindings
		)
	endif()

	# Add all subfolders as solution-folders:
	list(APPEND FOLDERS "Resources")
	function(includefolder PATH)
		FILE(GLOB FOLDER_FILES
			"${PATH}/*.cpp"
			"${PATH}/*.h"
			"${PATH}/*.rc"
		)
		source_group("${PATH}" FILES ${FOLDER_FILES})
	endfunction(includefolder)

	foreach(folder ${FOLDERS})
		includefolder(${folder})
	endforeach(folder)

	file(GLOB_RECURSE SOURCE
		"*.cpp"
		"*.h"
	)

	include_directories("${PROJECT_SOURCE_DIR}")

	source_group("" FILES ${SOURCE})

	# Precompiled headers (1st part)
	SET_SOURCE_FILES_PROPERTIES(
		Globals.cpp PROPERTIES COMPILE_FLAGS "/Yc\"Globals.h\""
	)
	# CMake cannot "remove" the precompiled header flags, so we use a dummy precompiled header compatible with just this one file:
	SET_SOURCE_FILES_PROPERTIES(
		Bindings/Bindings.cpp PROPERTIES COMPILE_FLAGS "/Yc\"string.h\" /Fp\"$(IntDir)/Bindings.pch\""
	)
	SET_SOURCE_FILES_PROPERTIES(
		"StackWalker.cpp LeakFinder.h" PROPERTIES COMPILE_FLAGS "/Yc\"Globals.h\""
	)
	list(APPEND SOURCE "Resources/MCServer.rc")
endif()

set(EXECUTABLE MCServer)

add_executable(${EXECUTABLE} ${SOURCE})


# Output the executable into the $/MCServer folder, so that it has access to external resources:
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_SOURCE_DIR}/MCServer)
SET_TARGET_PROPERTIES(${EXECUTABLE} PROPERTIES
	RUNTIME_OUTPUT_DIRECTORY_DEBUG          ${CMAKE_SOURCE_DIR}/MCServer
	RUNTIME_OUTPUT_DIRECTORY_RELEASE        ${CMAKE_SOURCE_DIR}/MCServer
	RUNTIME_OUTPUT_DIRECTORY_DEBUGPROFILE   ${CMAKE_SOURCE_DIR}/MCServer
	RUNTIME_OUTPUT_DIRECTORY_RELEASEPROFILE ${CMAKE_SOURCE_DIR}/MCServer
)


# Make the debug executable have a "_debug" suffix
SET_TARGET_PROPERTIES(${EXECUTABLE} PROPERTIES DEBUG_POSTFIX "_debug")

# Make the profiled executables have a "_profile" postfix
SET_TARGET_PROPERTIES(${EXECUTABLE} PROPERTIES DEBUGPROFILE_POSTFIX "_debug_profile")
SET_TARGET_PROPERTIES(${EXECUTABLE} PROPERTIES RELEASEPROFILE_POSTFIX "_profile")


# Precompiled headers (2nd part)
if (MSVC)
	SET_TARGET_PROPERTIES(
		${EXECUTABLE} PROPERTIES COMPILE_FLAGS "/Yu\"Globals.h\""
		OBJECT_DEPENDS "$(IntDir)/$(TargetName.pch)"
	)
endif ()


if (NOT MSVC)
	target_link_libraries(${EXECUTABLE} OSSupport HTTPServer Bindings Items Blocks)
	target_link_libraries(${EXECUTABLE} Protocol Generating WorldStorage)
	target_link_libraries(${EXECUTABLE} Mobs Entities Simulator UI BlockEntities)
endif ()
if (WIN32)
	target_link_libraries(${EXECUTABLE} expat tolualib ws2_32.lib Psapi.lib)
endif()
target_link_libraries(${EXECUTABLE} md5 luaexpat iniFile jsoncpp cryptopp zlib lua sqlite)
