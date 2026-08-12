cmake_minimum_required(VERSION 3.15 FATAL_ERROR)

project(remc2
	VERSION 10.0
	LANGUAGES CXX C
	HOMEPAGE_URL https://github.com/turican0/remc2
	DESCRIPTION "Magic Carpet 2: Netherworld by Bullfrog remake in C++"
)

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED OFF)
set(CMAKE_C_EXTENSIONS ON)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

set(CMAKE_COLOR_DIAGNOSTICS ON)

include_directories(
	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src>
	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/findfirst>
	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/stb>
	#	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/inih>
	#	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/inih/cpp>
	#	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/libADLMIDI/include>
)
link_directories(
	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}>
	$<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/lib/findfirst>
)

if(NOT CMAKE_GENERATOR MATCHES "^VisualStudio")
	include(GNUInstallDirs)
endif()

################################################################################
# Global configuration types
################################################################################
set(CMAKE_CONFIGURATION_TYPES
	"Debug"
	"Release"
	"ReleaseStrip"
	CACHE STRING "" FORCE
)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "${CMAKE_HOST_SYSTEM_NAME} ${CMAKE_HOST_SYSTEM_PROCESSOR} ${CMAKE_CXX_COMPILER_ID}/${CMAKE_CXX_COMPILER_FRONTEND_VARIANT} => ${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_PROCESSOR} ${CMAKE_CXX_COMPILER_ID}/${CMAKE_CXX_COMPILER_FRONTEND_VARIANT}")

################################################################################
# Global compiler options
################################################################################
add_compile_definitions(
	"$<$<CONFIG:DEBUG>:_DEBUG>"
)

if(CMAKE_CXX_COMPILER_FRONTEND_VARIANT MATCHES "GNU|AppleClang")
	if(NOT CMAKE_CROSSCOMPILING)
		add_compile_options(-march=native)
	endif()
	if(CMAKE_SYSTEM_PROCESSOR MATCHES "x64|x86_64|AMD64")
		add_compile_definitions(COMPILE_FOR_64BIT)
		add_compile_options(-mfpmath=sse)
	endif()
	# Linux/ARM
	if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm|aarch64)")
		add_compile_options(-fsigned-char)
	endif ()
	if (${SYSTEM_PROCESSOR_UPPER} MATCHES "PPC|POWERPC|PPC64|PPC64LE")
		include(CheckCSourceCompiles)

		set(CMAKE_REQUIRED_FLAGS "-maltivec")
		check_cxx_source_compiles("#include <altivec.h> int main() { vector int v = (vector int){0}; return 0; }" HAVE_ALTIVEC)

		if(HAVE_ALTIVEC)
			add_compile_options(-maltivec)
		endif()
	endif()
	add_compile_options(-ftree-vectorize)
	add_compile_options(-fopenmp)
	if(NOT CMAKE_CXX_COMPILER_FRONTEND_VARIANT MATCHES "AppleClang")
		add_compile_options(-fopenmp-simd)
	else()
		# Reverse-engineered structs use #pragma pack(1), which places pointers
		# at non-8-aligned offsets. Apple Silicon's default fixup-chain
		# relocations reject these. Fall back to classic relocations.
		add_link_options(-Wl,-no_fixup_chains)
	endif()
	if (CMAKE_BUILD_TYPE MATCHES Debug)
		add_compile_options(-g)
	else()
		add_compile_options(-O3)
	endif()
	add_compile_options(-fpermissive)
	add_compile_options(-Wno-narrowing)
	add_compile_options(-Wall)
	add_compile_options(-Wextra)
	add_compile_options(-pedantic)
	add_compile_options(-Wno-unknown-pragmas)
	add_compile_options(-Wno-write-strings)
	add_compile_options(-Wno-unused-result)
	add_compile_options(-Wno-unused-function)
	add_compile_options(-Wno-unused-variable)
	add_compile_options(-Wno-unused-but-set-variable)
	add_compile_options(-Wno-unused-parameter)
	add_compile_options(-Wno-missing-field-initializers)
	add_compile_options(-Wunreachable-code)		# Warn about unreachable code paths
	add_compile_options(-fno-asynchronous-unwind-tables)
	add_compile_options(-ffunction-sections)	# Put each function in its own section
	add_compile_options(-fdata-sections)		# Put each variable in its own section
	if (CMAKE_BUILD_TYPE MATCHES ReleaseStrip)
		add_link_options(-Wl,--gc-sections)
		add_link_options(-Wl,-s)
	endif()
	link_libraries(m)
	if (WIN32 AND MINGW)
		link_libraries(mingw32)
	endif()
elseif(CMAKE_CXX_COMPILER_FRONTEND_VARIANT MATCHES "MSVC")
	string(REPLACE "/RTC1" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
	string(REPLACE "/RTC1" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	add_compile_options(/openmp:experimental)
	add_compile_options(/Zc:__cplusplus)
	add_compile_options(/std:c++latest)
	add_compile_options(/Zc:preprocessor)
	add_compile_options(/Z7)
	add_compile_options(/Gy)					# Function-level linking (puts each function in its own section)
	add_compile_options(/Gw)					# Optimize global data (puts each variable in its own section)
	if (NOT CMAKE_CROSSCOMPILING)
		add_compile_definitions(_CRT_SECURE_NO_WARNINGS _WIN32)
		if (CMAKE_VS_PLATFORM_NAME MATCHES "x64|Win32")
			#add_compile_options(/Gv)
		endif()
	endif()
	if (CMAKE_BUILD_TYPE MATCHES ReleaseStrip)
		add_link_options(/DEBUG:NONE)
		add_link_options(/OPT:REF)				# Eliminate unreferenced functions and data
		add_link_options(/VERBOSE:REF)			# Print out everything the linker throws away
	endif()
	if (CMAKE_BUILD_TYPE MATCHES Debug)
		add_compile_options(/Zi)
		add_link_options(/DEBUG:FULL)
	else()
		add_compile_options(/O2)
	endif()
	add_link_options(/INCREMENTAL:NO)
else()
	message(FATAL_ERROR "Compiler frontend variant not supported!")
endif()



################################################################################
# Use solution folders feature
################################################################################
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

################################################################################
# Sub-projects
################################################################################

set(APP_ICON_RESOURCE ${CMAKE_SOURCE_DIR}/assets/icons/app.rc)
source_group("Source Files" FILES ${PROJECT_NAME})
source_group("Header Files" FILES ${HEADER_FILES})

set(SOURCE_FILES_ENGINE
	"src/engine/MenuState.h"
	"src/engine/Animation.cpp"
	"src/engine/Animation.h"
	"src/engine/Basic.cpp"
	"src/engine/Basic.h"
	"src/engine/CommandLineParser.cpp"
	"src/engine/CommandLineParser.h"
	"src/engine/Compare.cpp"
	"src/engine/Compare.h"
	"src/engine/Events.cpp"
	"src/engine/Events.h"
	"src/engine/Event.h"
	"src/engine/EventDispatcher.cpp"
	"src/engine/EventDispatcher.h"
	"src/engine/GameBitmap.cpp"
	"src/engine/GameBitmap.h"
	"src/engine/GameBitmapIndexes.h"
	"src/engine/GameRenderGL.cpp"
	"src/engine/GameRenderGL.h"
	"src/engine/GameRenderHD.cpp"
	"src/engine/GameRenderHD.h"
	"src/engine/GameRenderInterface.h"
	"src/engine/GameRenderNG.cpp"
	"src/engine/GameRenderNG.h"
	"src/engine/GameRenderOriginal.cpp"
	"src/engine/GameRenderOriginal.h"
	"src/engine/GameUI.cpp"
	"src/engine/GameUI.h"
	"src/engine/GameUiConstants.h"
	"src/engine/Graphics.cpp"
	"src/engine/Graphics.h"
	"src/engine/LangTextIndexes.h"
	"src/engine/Level.cpp"
	"src/engine/Level.h"
	"src/engine/LevelStructs.cpp"
	"src/engine/LevelStructs.h"
	"src/engine/Map.h"
	"src/engine/MapColourIndexs.h"
	"src/engine/MenusAndIntros.cpp"
	"src/engine/MenusAndIntros.h"
	"src/engine/Network.cpp"
	"src/engine/Network.h"
	"src/engine/Player.cpp"
	"src/engine/Player.h"
	"src/engine/PlayerInput.cpp"
	"src/engine/PlayerInput.h"
	"src/engine/ProjectionPolygon.cpp"
	"src/engine/ProjectionPolygon.h"
	"src/engine/RenderThread.cpp"
	"src/engine/RenderThread.h"
	"src/engine/Sound.cpp"
	"src/engine/Sound.h"
	"src/engine/XmiInfo.cpp"
	"src/engine/XmiInfo.h"
	"src/engine/WavIndexes.h"
	"src/engine/Terrain.cpp"
	"src/engine/Terrain.h"
	"src/engine/TextureMaps.cpp"
	"src/engine/TextureMaps.h"
	"src/engine/Type_D404C.cpp"
	"src/engine/Type_D404C.h"
	"src/engine/Type_D93C0_Bldgprmbuffer.cpp"
	"src/engine/Type_D93C0_Bldgprmbuffer.h"
	"src/engine/Type_D94F0_Bldgprmbuffer.cpp"
	"src/engine/Type_D94F0_Bldgprmbuffer.h"
	"src/engine/Type_MapScreenPortals_E17CC.cpp"
	"src/engine/Type_MapScreenPortals_E17CC.h"
	"src/engine/Type_SecretMapScreenPortals_E2970.cpp"
	"src/engine/Type_SecretMapScreenPortals_E2970.h"
	"src/engine/Type_WORD_D951C.cpp"
	"src/engine/Type_WORD_D951C.h"
	"src/engine/Unk_D47E0.cpp"
	"src/engine/Unk_D47E0.h"
	"src/engine/Unk_D4A30.cpp"
	"src/engine/Unk_D4A30.h"
	"src/engine/ViewPort.cpp"
	"src/engine/ViewPort.h"
	"src/engine/ail_sound.h"
	"src/engine/defs.h"
	"src/engine/engine_support.cpp"
	"src/engine/engine_support.h"
	"src/engine/engine_support_converts.cpp"
	"src/engine/engine_support_converts.h"
	"src/engine/read_config.cpp"
	"src/engine/read_config.h"
	"src/engine/sub_main_mouse.h"
	"src/engine/RecordedEvent.h"
	"src/sub_main.cpp"
	"src/sub_main.h"
	"src/utilities/BitmapIO.cpp"
	"src/utilities/BitmapIO.h"
	"src/utilities/DataFileIO.cpp"
	"src/utilities/DataFileIO.h"
	"src/utilities/Maths.cpp"
	"src/utilities/Maths.h"
	"src/utilities/RendererTests.cpp"
	"src/utilities/RendererTests.h"
	"src/utilities/StateMonitor.cpp"
	"src/utilities/StateMonitor.h"
	"src/build_config.h"
	"src/engine/BasicMini.cpp"
	"src/engine/BasicMini.h"
	"src/engine/BasicTerrain.cpp"
	"src/engine/BasicTerrain.h"
	"src/engine/CompareMaps.cpp"
	"src/engine/CompareMaps.h"
	"src/engine/ConvertMapInfo.cpp"
	"src/engine/ConvertMapInfo.h"
	"src/engine/DatTabIndexes.cpp"
	"src/engine/DatTabIndexes.h"
	"src/engine/EventsFunctions.cpp"
	"src/engine/EventsFunctions.h"
	"src/engine/LevelInit.cpp"
	"src/engine/LevelInit.h"
	"src/engine/MouseOnOff.cpp"
	"src/engine/MouseOnOff.h"
	"src/engine/PrintMessages.cpp"
	"src/engine/PrintMessages.h"
	"src/engine/ReadAndDecompress.cpp"
	"src/engine/ReadAndDecompress.h"
	"src/engine/ScreenBuff.cpp"
	"src/engine/ScreenBuff.h"
	"src/engine/Spells.cpp"
	"src/engine/Spells.h"
	"src/engine/UVTable_D4350.cpp"
	"src/engine/UVTable_D4350.h"
	"src/utilities/DataFileRNC.cpp"
	"src/utilities/DataFileRNC.h"
	"src/utilities/stb_impl.cpp"
	"lib/stb/stb_image.h"
	"lib/stb/stb_image_write.h"
)
source_group("Source Files\\engine" FILES ${SOURCE_FILES_ENGINE})

set(SOURCE_FILES__libs__findfirst
	"lib/findfirst/findfirst.c"
	"lib/findfirst/findfirst.h"
	"lib/findfirst/spec.c"
	"lib/findfirst/spec.h"
)
source_group("Source Files\\libs\\findfirst" FILES ${SOURCE_FILES__libs__findfirst})
add_library(findfirst ${SOURCE_FILES__libs__findfirst})

set(SOURCE_FILES__portability
	"src/portability/bitmap_pos_struct.h"
	"src/portability/Config.cpp"
	"src/portability/Config.h"
	"src/portability/fcaseopen.cpp"
	"src/portability/fcaseopen.h"
	"src/portability/GameKey.h"
	"src/portability/KeyboardInputMapping.h"
	"src/portability/MouseInputMapping.h"
	"src/portability/port_net.cpp"
	"src/portability/port_net.h"
	"src/portability/port_filesystem.cpp"
	"src/portability/port_filesystem.h"
	"src/portability/InputRecorder.cpp"
	"src/portability/InputRecorder.h"
	"src/portability/port_outputs.cpp"
	"src/portability/port_outputs.h"
	"src/portability/port_sdl_sound.cpp"
	"src/portability/port_sdl_sound.h"
	"src/portability/port_sdl_vga_mouse.cpp"
	"src/portability/port_sdl_vga_mouse.h"
	"src/portability/port_sdl_joystick.cpp"
	"src/portability/port_sdl_joystick.h"
	"src/portability/port_show_perifery.cpp"
	"src/portability/port_show_perifery.h"
	"src/portability/port_time.cpp"
	"src/portability/port_time.h"
	"src/portability/xmi2mid.cpp"
	"src/portability/xmi2mid.h"
)
source_group("Source Files\\portability" FILES ${SOURCE_FILES__portability})
set(ALL_FILES
	${HEADER_FILES}
	${SOURCE_FILES}
	${SOURCE_FILES_ENGINE}
	${SOURCE_FILES__portability}
	${SOURCE_FILES__networklib}
)

################################################################################
# Dependencies
################################################################################

cmake_policy(SET CMP0072 NEW)   #OpenGL_GL_PREFERENCE to GLVND
find_package(OpenGL)
find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(SDL2_ttf REQUIRED)
find_package(libADLMIDI REQUIRED)
find_package(SDL2_mixer_ext REQUIRED)
#find_package(PNG REQUIRED)
#find_package(fmt REQUIRED)
#find_package(nlohmann_json REQUIRED)
#find_package(spdlog REQUIRED)

################################################################################
# targets
################################################################################
add_executable(${PROJECT_NAME} src/remc2.cpp ${ALL_FILES})
target_link_libraries(${PROJECT_NAME} PUBLIC 
	SDL2::SDL2
	SDL2::SDL2main
	libADLMIDI::ADLMIDI_static
	SDL2_mixer_ext::SDL2_mixer_ext_Static
	SDL2_ttf::SDL2_ttf
	findfirst
	#${PNG_LIBRARY}
	#fmt::fmt
	#nlohmann_json::nlohmann_json
	#spdlog::spdlog
)


################################################################################
# Linux settings
################################################################################

install(
	TARGETS ${PROJECT_NAME}
	CONFIGURATIONS Debug;Release;RelWithDebInfo;ReleaseStrip
)

install(TARGETS ${PROJECT_NAME} DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/games/${PROJECT_NAME})

install(DIRECTORY ${CMAKE_SOURCE_DIR}/assets/ DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/games/${PROJECT_NAME} USE_SOURCE_PERMISSIONS)

install(
	FILES 
	${CMAKE_SOURCE_DIR}/config.json
	DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/games/${PROJECT_NAME}
)

set(ROOT_NAMESPACE remc2)

