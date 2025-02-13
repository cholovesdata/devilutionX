cmake_minimum_required(VERSION 3.13)

if(POLICY CMP0092)
  cmake_policy(SET CMP0092 NEW)
endif()
if(POLICY CMP0083)
  cmake_policy(SET CMP0083 NEW)
endif()
if(POLICY CMP0111)
  cmake_policy(SET CMP0111 NEW)
endif()

include(CMakeDependentOption)
include(CMake/out_of_tree.cmake)
include(CMake/genex.cmake)

DEBUG_OPTION(ASAN "Enable address sanitizer")
DEBUG_OPTION(UBSAN "Enable undefined behaviour sanitizer")
option(TSAN "Enable thread sanitizer (not compatible with ASAN=ON)" OFF)
DEBUG_OPTION(DEBUG "Enable debug mode in engine")
option(GPERF "Build with GPerfTools profiler" OFF)
cmake_dependent_option(GPERF_HEAP_FIRST_GAME_ITERATION "Save heap profile of the first game iteration" OFF "GPERF" OFF)
option(DISABLE_LTO "Disable link-time optimization (by default enabled in release mode)" OFF)
option(PIE "Generate position-independent code" OFF)
option(DIST "Dynamically link only glibc and SDL2" OFF)
option(BINARY_RELEASE "Enable options for binary release" OFF)
option(NIGHTLY_BUILD "Enable options for nightly build" OFF)
option(USE_SDL1 "Use SDL1.2 instead of SDL2" OFF)
option(NONET "Disable network support" OFF)
option(NOSOUND "Disable sound support" OFF)
option(RUN_TESTS "Build and run tests" OFF)
option(ENABLE_CODECOVERAGE "Instrument code for code coverage (only enabled with RUN_TESTS)" OFF)

if(NOT NONET)
  option(DISABLE_TCP "Disable TCP multiplayer option" OFF)
  option(DISABLE_ZERO_TIER "Disable ZeroTier multiplayer option" OFF)
endif()

option(DISABLE_STREAMING_MUSIC "Disable streaming music (to work around broken platform implementations)" OFF)
mark_as_advanced(DISABLE_STREAMING_MUSIC)
option(DISABLE_STREAMING_SOUNDS "Disable streaming sounds (to work around broken platform implementations)" OFF)
mark_as_advanced(DISABLE_STREAMING_SOUNDS)
option(STREAM_ALL_AUDIO "Stream all the audio. For extremely RAM-constrained platforms.")
mark_as_advanced(STREAM_ALL_AUDIO)

# The gettext[tools] package takes a very long time to install
if(CMAKE_TOOLCHAIN_FILE MATCHES "vcpkg.cmake$")
  option(USE_GETTEXT_FROM_VCPKG "Add vcpkg dependency for gettext[tools] for compiling translations" OFF)
endif()

RELEASE_OPTION(CPACK "Configure CPack")


if(BINARY_RELEASE OR CMAKE_BUILD_TYPE STREQUAL "Release")
  set(BINARY_RELEASE ON)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "")
  set(DIST ON)
  set(CPACK ON)
endif()

if(NIGHTLY_BUILD OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  set(NIGHTLY_BUILD ON)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "")
  set(DIST ON)
  set(CPACK ON)
endif()

if(USE_GETTEXT_FROM_VCPKG)
  list(APPEND VCPKG_MANIFEST_FEATURES "translations")
endif()
if(RUN_TESTS)
  list(APPEND VCPKG_MANIFEST_FEATURES "tests")
endif()

if(NOT NOSOUND)
  option(DEVILUTIONX_SYSTEM_SDL_AUDIOLIB "Use system-provided SDL_audiolib" OFF)
  cmake_dependent_option(DEVILUTIONX_STATIC_SDL_AUDIOLIB "Link static SDL_audiolib" OFF
                        "DEVILUTIONX_SYSTEM_SDL_AUDIOLIB AND NOT DIST" ON)
endif()

if(NOT NONET)
  option(DEVILUTIONX_SYSTEM_LIBSODIUM "Use system-provided libsodium" ON)
  cmake_dependent_option(DEVILUTIONX_STATIC_LIBSODIUM "Link static libsodium" OFF
                        "DEVILUTIONX_SYSTEM_LIBSODIUM AND NOT DIST" ON)
endif()

option(DEVILUTIONX_SYSTEM_LIBFMT "Use system-provided libfmt" ON)
cmake_dependent_option(DEVILUTIONX_STATIC_LIBFMT "Link static libfmt" OFF
                       "DEVILUTIONX_SYSTEM_LIBFMT AND NOT DIST" ON)

option(DEVILUTIONX_SYSTEM_LIBPNG "Use system-provided libpng" ON)
cmake_dependent_option(DEVILUTIONX_STATIC_LIBPNG "Link static libpng" OFF
                       "DEVILUTIONX_SYSTEM_LIBPNG AND NOT DIST" ON)

if(NOT VERSION_NUM)
  include(CMake/git.cmake)
  get_git_tag(VERSION_NUM)
  if (NOT "${VERSION_NUM}" STREQUAL "")
    string(REGEX MATCH "([0-9]+\\.[0-9]+\\.[0-9]+)" VERSION_NUM ${VERSION_NUM} )
  endif()
  get_git_commit_hash(GIT_COMMIT_HASH)
  if(NOT VERSION_SUFFIX)
    set(VERSION_SUFFIX "$<$<NOT:$<CONFIG:Release>>:-${GIT_COMMIT_HASH}>")
  endif()
endif()

find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

if(VERSION_NUM MATCHES untagged)
  project(DevilutionX
    LANGUAGES C CXX)
else()
  project(DevilutionX
    VERSION ${VERSION_NUM}
    LANGUAGES C CXX)
endif()

# Not a genexp because CMake doesn't support it
# https://gitlab.kitware.com/cmake/cmake/-/issues/20546
if(NOT DISABLE_LTO)
  # LTO if supported:
  include(CheckIPOSupported)
  check_ipo_supported(RESULT is_ipo_supported OUTPUT lto_error)
  if(is_ipo_supported)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELWITHDEBINFO ON)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_MINSIZEREL ON)
  endif()
endif()

list(APPEND CMAKE_MODULE_PATH "${DevilutionX_SOURCE_DIR}/CMake")

if(GPERF)
  if(GPERF_HEAP_FIRST_GAME_ITERATION)
    set(GPERF_HEAP_MAIN ON)
  endif()

  # Compile with information about file and line numbers for everything
  # even in non-Debug build types.
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    add_compile_options("$<$<NOT:$<CONFIG:Debug>>:-g2>")
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # Use the more size-efficient `-gmlt` option on clang.
    add_compile_options("$<$<NOT:$<CONFIG:Debug>>:-gmlt>")
  endif()
endif()

if(NINTENDO_SWITCH)
  list(APPEND CMAKE_MODULE_PATH "${DevilutionX_SOURCE_DIR}/CMake/switch")
  include(switch_defs)
endif()

if(VITA)
  # Work around a missing setting in the toolchain file.
  # Fix sent upstream: https://github.com/vitasdk/vita-toolchain/pull/182
  set(PKG_CONFIG_EXECUTABLE "$ENV{VITASDK}/bin/arm-vita-eabi-pkg-config")

  include("$ENV{VITASDK}/share/vita.cmake" REQUIRED)
  include(vita_defs)
endif()

set(TARGET_PLATFORM host CACHE STRING "Target platform")
set_property(CACHE TARGET_PLATFORM PROPERTY STRINGS host retrofw rg350 gkd350h cpigamesh)
if(TARGET_PLATFORM STREQUAL "retrofw")
  include(retrofw_defs)
elseif(TARGET_PLATFORM STREQUAL "rg350")
  include(rg350_defs)
elseif(TARGET_PLATFORM STREQUAL "gkd350h")
  include(gkd350h_defs)
elseif(TARGET_PLATFORM STREQUAL "cpigamesh")
  include(cpigamesh_defs)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD|NetBSD|OpenBSD|DragonFly")
  if(${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    add_definitions(-D_NETBSD_SOURCE)
  else()
    add_definitions(-D_BSD_SOURCE)
    set(UBSAN OFF)
  endif()
  set(ASAN OFF)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DO_LARGEFILE=0 -Dstat64=stat -Dlstat64=lstat -Dlseek64=lseek -Doff64_t=off_t -Dfstat64=fstat -Dftruncate64=ftruncate")
endif()

if(WIN32)
  set(ASAN OFF)
  set(UBSAN OFF)
  set(DIST ON)
endif()

if(HAIKU)
  set(ASAN OFF)
  set(UBSAN OFF)
endif()

if(AMIGA)
  include(amiga_defs)
endif()

if(NINTENDO_3DS)
  list(APPEND CMAKE_MODULE_PATH "${DevilutionX_SOURCE_DIR}/CMake/ctr")
  include(n3ds_defs)
endif()

if(ANDROID)
  list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/android-project/CMake")
  include(android_defs)
endif()

if(PIE)
  set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON) # for clang-tidy
set(CMAKE_THREAD_PREFER_PTHREAD ON)
set(THREADS_PREFER_PTHREAD_FLAG ON)

if(NOT NINTENDO_3DS)
  find_package(Threads REQUIRED)
endif()

if(NOT NONET)
  if(DEVILUTIONX_SYSTEM_LIBSODIUM)
    set(sodium_USE_STATIC_LIBS ${DEVILUTIONX_STATIC_LIBSODIUM})
    find_package(sodium REQUIRED)
  else()
    add_subdirectory(3rdParty/libsodium)
  endif()
endif()

if(DEVILUTIONX_SYSTEM_LIBFMT)
  find_package(fmt 7.0.0 QUIET)
  if(fmt_FOUND)
    message("-- Found fmt ${fmt_VERSION}")
  else()
    message("-- Suitable system fmt package not found, will use fmt from source")
  endif()
endif()
if(NOT fmt_FOUND)
  add_subdirectory(3rdParty/libfmt)
endif()

if(DEVILUTIONX_SYSTEM_LIBPNG)
  find_package(PNG QUIET)
  if(PNG_FOUND)
    message("-- Found png ${PNG_VERSION_STRING}")
  else()
    message("-- Suitable system png package not found, will use png from source")
  endif()
  if(NOT TARGET PNG::PNG AND TARGET 3ds::png)
    add_library(PNG::PNG ALIAS 3ds::png)
  endif()
endif()
if(NOT PNG_FOUND)
  add_subdirectory(3rdParty/libpng)
endif()

if(ANDROID)
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/android-project/3rdParty/SDL2)
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/android-project/3rdParty/SDL2_ttf)
elseif(USE_SDL1)
  find_package(SDL REQUIRED)
  find_package(SDL_ttf REQUIRED)
  include_directories(${SDL_INCLUDE_DIR})
else()
  find_package(SDL2 REQUIRED)
  if(TARGET SDL2::SDL2)
    set(SDL2_MAIN SDL2::SDL2main)
  elseif(TARGET SDL2::SDL2-static)
    # On some distros, such as vitasdk, only the SDL2::SDL2-static target is available.
    # Alias to SDL2::SDL2 because some finder scripts may refer to SDL2::SDL2.
    if(CMAKE_VERSION VERSION_LESS "3.18")
      # Aliasing local targets is not supported on CMake < 3.18, so make it global.
      set_target_properties(SDL2::SDL2-static PROPERTIES IMPORTED_GLOBAL TRUE)
    endif()
    add_library(SDL2::SDL2 ALIAS SDL2::SDL2-static)
    set(SDL2_MAIN SDL2::SDL2main)
  else()
    # Assume an older Debian derivate that comes with an sdl2-config.cmake
    # that only defines `SDL2_LIBRARIES` (as -lSDL2) and `SDL2_INCLUDE_DIRS`.
    add_library(SDL2_lib INTERFACE)
    target_link_libraries(SDL2_lib INTERFACE ${SDL2_LIBRARIES})
    target_include_directories(SDL2_lib INTERFACE ${SDL2_INCLUDE_DIRS})
    # Can't define an INTERFACE target with ::, so alias instead
    add_library(SDL2::SDL2 ALIAS SDL2_lib)
  endif()
  find_package(SDL2_ttf REQUIRED)
endif()

if(NOT NONET AND NOT DISABLE_TCP)
  add_subdirectory(3rdParty/asio)
  if(NINTENDO_3DS)
      target_compile_definitions(asio INTERFACE
        ASIO_DISABLE_THREADS=ON
        ASIO_HAS_UNISTD_H=ON)

      target_include_directories(asio BEFORE INTERFACE CMake/ctr/asio/include)
  elseif(NINTENDO_SWITCH)
    include(asio_defs)
  endif()
endif()

add_subdirectory(3rdParty/SDL_image)

if(NOT NOSOUND)
  if(DEVILUTIONX_SYSTEM_SDL_AUDIOLIB)
    find_package(SDL_audiolib REQUIRED)
  else()
    add_subdirectory(3rdParty/SDL_audiolib)
  endif()
endif()

add_library(smacker STATIC
  3rdParty/libsmacker/smacker.c)
target_include_directories(smacker PUBLIC 3rdParty/libsmacker)

if(WIN32)
add_subdirectory(3rdParty/find_steam_game)
endif()

add_subdirectory(3rdParty/simpleini)

add_library(StormLib STATIC
  3rdParty/StormLib/src/FileStream.cpp
  3rdParty/StormLib/src/SBaseCommon.cpp
  3rdParty/StormLib/src/SBaseFileTable.cpp
  3rdParty/StormLib/src/SBaseSubTypes.cpp
  3rdParty/StormLib/src/SCompression.cpp
  3rdParty/StormLib/src/SFileExtractFile.cpp
  3rdParty/StormLib/src/SFileFindFile.cpp
  3rdParty/StormLib/src/SFileGetFileInfo.cpp
  3rdParty/StormLib/src/SFileOpenArchive.cpp
  3rdParty/StormLib/src/SFileOpenFileEx.cpp
  3rdParty/StormLib/src/SFileReadFile.cpp)

if(WIN32)
# Enable Unicode for StormLib wchar_t* file APIs
target_compile_definitions(StormLib PRIVATE -DUNICODE -D_UNICODE)
endif()

add_library(PKWare STATIC
  3rdParty/PKWare/explode.cpp
  3rdParty/PKWare/implode.cpp)
target_include_directories(PKWare PUBLIC 3rdParty/PKWare)

set(libdevilutionx_SRCS
  Source/appfat.cpp
  Source/automap.cpp
  Source/capture.cpp
  Source/codec.cpp
  Source/control.cpp
  Source/cursor.cpp
  Source/dead.cpp
  Source/debug.cpp
  Source/diablo.cpp
  Source/doom.cpp
  Source/drlg_l1.cpp
  Source/drlg_l2.cpp
  Source/drlg_l3.cpp
  Source/drlg_l4.cpp
  Source/dthread.cpp
  Source/dx.cpp
  Source/encrypt.cpp
  Source/engine.cpp
  Source/error.cpp
  Source/gamemenu.cpp
  Source/gendung.cpp
  Source/gmenu.cpp
  Source/help.cpp
  Source/hwcursor.cpp
  Source/init.cpp
  Source/interfac.cpp
  Source/inv.cpp
  Source/itemdat.cpp
  Source/items.cpp
  Source/lighting.cpp
  Source/loadsave.cpp
  Source/menu.cpp
  Source/minitext.cpp
  Source/misdat.cpp
  Source/missiles.cpp
  Source/monstdat.cpp
  Source/monster.cpp
  Source/movie.cpp
  Source/mpqapi.cpp
  Source/msg.cpp
  Source/multi.cpp
  Source/nthread.cpp
  Source/objdat.cpp
  Source/objects.cpp
  Source/options.cpp
  Source/pack.cpp
  Source/palette.cpp
  Source/path.cpp
  Source/pfile.cpp
  Source/player.cpp
  Source/plrmsg.cpp
  Source/portal.cpp
  Source/quests.cpp
  Source/restrict.cpp
  Source/scrollrt.cpp
  Source/setmaps.cpp
  Source/sha.cpp
  Source/spelldat.cpp
  Source/spells.cpp
  Source/stores.cpp
  Source/sync.cpp
  Source/textdat.cpp
  Source/themes.cpp
  Source/tmsg.cpp
  Source/town.cpp
  Source/towners.cpp
  Source/track.cpp
  Source/trigs.cpp
  Source/controls/axis_direction.cpp
  Source/controls/controller.cpp
  Source/controls/controller_motion.cpp
  Source/controls/devices/game_controller.cpp
  Source/controls/devices/joystick.cpp
  Source/controls/devices/kbcontroller.cpp
  Source/controls/game_controls.cpp
  Source/controls/menu_controls.cpp
  Source/controls/modifier_hints.cpp
  Source/controls/plrctrls.cpp
  Source/controls/keymapper.cpp
  Source/engine/animationinfo.cpp
  Source/engine/demomode.cpp
  Source/engine/load_cel.cpp
  Source/engine/random.cpp
  Source/engine/render/automap_render.cpp
  Source/engine/render/cel_render.cpp
  Source/engine/render/cl2_render.cpp
  Source/engine/render/dun_render.cpp
  Source/engine/render/text_render.cpp
  Source/engine/surface.cpp
  Source/qol/autopickup.cpp
  Source/qol/common.cpp
  Source/qol/monhealthbar.cpp
  Source/qol/xpbar.cpp
  Source/qol/itemlabels.cpp
  Source/utils/console.cpp
  Source/utils/display.cpp
  Source/utils/file_util.cpp
  Source/utils/language.cpp
  Source/utils/paths.cpp
  Source/utils/sdl_bilinear_scale.cpp
  Source/utils/sdl_thread.cpp
  Source/DiabloUI/art.cpp
  Source/DiabloUI/art_draw.cpp
  Source/DiabloUI/button.cpp
  Source/DiabloUI/credits.cpp
  Source/DiabloUI/credits_lines.cpp
  Source/DiabloUI/diabloui.cpp
  Source/DiabloUI/dialogs.cpp
  Source/DiabloUI/errorart.cpp
  Source/DiabloUI/fonts.cpp
  Source/DiabloUI/mainmenu.cpp
  Source/DiabloUI/progress.cpp
  Source/DiabloUI/scrollbar.cpp
  Source/DiabloUI/selconn.cpp
  Source/DiabloUI/selgame.cpp
  Source/DiabloUI/selhero.cpp
  Source/DiabloUI/selok.cpp
  Source/DiabloUI/selyesno.cpp
  Source/DiabloUI/support_lines.cpp
  Source/DiabloUI/text_draw.cpp
  Source/DiabloUI/title.cpp
  Source/DiabloUI/ttf_render_wrapped.cpp
  Source/panels/charpanel.cpp
  Source/panels/mainpanel.cpp
  Source/dvlnet/abstract_net.cpp
  Source/dvlnet/base.cpp
  Source/dvlnet/cdwrap.cpp
  Source/dvlnet/frame_queue.cpp
  Source/dvlnet/loopback.cpp
  Source/dvlnet/packet.cpp
  Source/storm/storm.cpp
  Source/storm/storm_file_wrapper.cpp
  Source/storm/storm_net.cpp
  Source/storm/storm_sdl_rw.cpp
  Source/storm/storm_svid.cpp
  Source/miniwin/misc_msg.cpp)

if(USE_SDL1)
  list(APPEND libdevilutionx_SRCS Source/utils/sdl2_to_1_2_backports.cpp)
endif()

if(NOSOUND)
  list(APPEND libdevilutionx_SRCS
    Source/effects_stubs.cpp
    Source/sound_stubs.cpp)
else()
  list(APPEND libdevilutionx_SRCS
    Source/effects.cpp
    Source/sound.cpp
    Source/utils/push_aulib_decoder.cpp
    Source/utils/soundsample.cpp)
endif()

if(NOT NONET)
  if(NOT DISABLE_TCP)
    list(APPEND libdevilutionx_SRCS
      Source/dvlnet/tcp_client.cpp
      Source/dvlnet/tcp_server.cpp)
  endif()
  if(NOT DISABLE_ZERO_TIER)
    list(APPEND libdevilutionx_SRCS
      Source/dvlnet/protocol_zt.cpp
      Source/dvlnet/zerotier_native.cpp
      Source/dvlnet/zerotier_lwip.cpp)
  endif()
endif()

if (VIRTUAL_GAMEPAD)
  list(APPEND libdevilutionx_SRCS
    Source/controls/touch/event_handlers.cpp
    Source/controls/touch/gamepad.cpp
    Source/controls/touch/renderers.cpp)
endif()

set(BIN_TARGET devilutionx)

if(NINTENDO_SWITCH)
  list(APPEND libdevilutionx_SRCS
    Source/platform/switch/network.cpp
    Source/platform/switch/keyboard.cpp
    Source/platform/switch/docking.cpp
    Source/platform/switch/random.cpp
    Source/platform/switch/asio/pause.c
    Source/platform/switch/asio/net/if.c
    Source/platform/switch/asio/sys/signal.c)
endif()

if(VITA)
  list(APPEND libdevilutionx_SRCS
    Source/platform/vita/keyboard.cpp
    Source/platform/vita/touch.cpp)
endif()

if(NINTENDO_3DS)
  list(APPEND libdevilutionx_SRCS
    Source/platform/ctr/system.cpp
    Source/platform/ctr/keyboard.cpp
    Source/platform/ctr/display.cpp
    Source/platform/ctr/messagebox.cpp
    Source/platform/ctr/random.cpp
    Source/platform/ctr/sockets.cpp
    Source/platform/ctr/locale.cpp
    Source/platform/ctr/asio/net/if.c
    Source/platform/ctr/asio/sys/socket.c
    Source/platform/ctr/asio/sys/uio.c)
  set(BIN_TARGET ${BIN_TARGET}.elf)
endif()

if(RUN_TESTS)
  set(devilutionxtest_SRCS
    test/appfat_test.cpp
    test/automap_test.cpp
    test/control_test.cpp
    test/cursor_test.cpp
    test/codec_test.cpp
    test/dead_test.cpp
    test/diablo_test.cpp
    test/drlg_l1_test.cpp
    test/effects_test.cpp
    test/file_util_test.cpp
    test/inv_test.cpp
    test/lighting_test.cpp
    test/main.cpp
    test/missiles_test.cpp
    test/pack_test.cpp
    test/path_test.cpp
    test/player_test.cpp
    test/quests_test.cpp
    test/random_test.cpp
    test/scrollrt_test.cpp
    test/stores_test.cpp
    test/writehero_test.cpp
    test/animationinfo_test.cpp)
endif()

add_library(libdevilutionx OBJECT ${libdevilutionx_SRCS})
if (ANDROID)
  add_library(${BIN_TARGET} SHARED Source/main.cpp)
else()
  add_executable(${BIN_TARGET}
    WIN32
    MACOSX_BUNDLE
    Source/main.cpp
    Source/devilutionx.exe.manifest
    Packaging/macOS/AppIcon.icns
    Packaging/resources/CharisSILB.ttf
    Packaging/windows/devilutionx.rc)
endif()
target_link_libraries(${BIN_TARGET} PRIVATE libdevilutionx)

# Copy the font and devilutionx.mpq to the build directory to it works from the build directory
file(COPY "Packaging/resources/CharisSILB.ttf" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
file(COPY "Packaging/resources/devilutionx.mpq" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")

# Use file GENERATE instead of configure_file because configure_file
# does not support generator expressions.
get_property(is_multi_config GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(is_multi_config)
  set(CONFIG_PATH $<CONFIG>/config.h)
  target_include_directories(libdevilutionx PUBLIC ${CMAKE_BINARY_DIR}/$<CONFIG>)
else()
  set(CONFIG_PATH config.h)
endif()
file(GENERATE OUTPUT ${CONFIG_PATH} CONTENT
"#pragma once
#define PROJECT_NAME \"${PROJECT_NAME}\"
#define PROJECT_VERSION \"${PROJECT_VERSION}${VERSION_SUFFIX}\"
#define PROJECT_VERSION_MAJOR ${PROJECT_VERSION_MAJOR}
#define PROJECT_VERSION_MINOR ${PROJECT_VERSION_MINOR}
#define PROJECT_VERSION_PATCH ${PROJECT_VERSION_PATCH}
")

if(RUN_TESTS)
  add_executable(devilutionx-tests WIN32 MACOSX_BUNDLE ${devilutionxtest_SRCS})
  include(CTest)
  include(GoogleTest)
  find_package(GTest REQUIRED)
  add_definitions(-DRUN_TESTS)
  target_include_directories(devilutionx-tests PRIVATE ${GTEST_INCLUDE_DIRS})
  target_link_libraries(devilutionx-tests PRIVATE libdevilutionx)
  target_link_libraries(devilutionx-tests PRIVATE ${GTEST_LIBRARIES})
  target_include_directories(devilutionx-tests PRIVATE 3rdParty/PicoSHA2)
  if(ENABLE_CODECOVERAGE)
    if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
      message(WARNING "Codecoverage not supported with MSVC")
    else()
      target_compile_options(devilutionx-tests PRIVATE -fprofile-arcs -ftest-coverage)
      target_compile_options(libdevilutionx PRIVATE -fprofile-arcs -ftest-coverage)
      target_compile_options(${BIN_TARGET} PRIVATE -fprofile-arcs -ftest-coverage)
      target_link_options(devilutionx-tests PRIVATE -fprofile-arcs)
      target_link_options(libdevilutionx PRIVATE -fprofile-arcs)
      target_link_options(${BIN_TARGET} PRIVATE -fprofile-arcs)
    endif()
  endif()
  gtest_add_tests(devilutionx-tests "" AUTO)
endif()

if(GPERF)
  find_package(Gperftools REQUIRED)
endif()

if(USE_GETTEXT_FROM_VCPKG)
  # vcpkg doesn't add its own tools directory to the search path
  list(APPEND Gettext_ROOT ${CMAKE_CURRENT_BINARY_DIR}/vcpkg_installed/${VCPKG_TARGET_TRIPLET}/tools/gettext/bin)
endif()
find_package(Gettext)
if (Gettext_FOUND)
  foreach(lang bg da de es fr hr it ko_KR pt_BR ru sv zh_CN zh_TW)
    GETTEXT_PROCESS_PO_FILES(${lang} ALL PO_FILES ${CMAKE_CURRENT_LIST_DIR}/Translations/${lang}.po)
    list(APPEND devilutionx_TRANSLATIONS ${CMAKE_CURRENT_BINARY_DIR}/${lang}.gmo)

    if(VITA)
      list(APPEND VITA_TRANSLATIONS_LIST "FILE" "${CMAKE_CURRENT_BINARY_DIR}/${lang}.gmo" "${lang}.gmo")
    endif()
  endforeach(lang)

  if(ANDROID)
    add_custom_target(copy_translations ALL
      COMMAND ${CMAKE_COMMAND} -E copy ${devilutionx_TRANSLATIONS} ${DevilutionX_SOURCE_DIR}/android-project/app/src/main/assets
      DEPENDS ${devilutionx_TRANSLATIONS})

    add_dependencies(${BIN_TARGET} copy_translations)
  endif()
endif()

target_include_directories(libdevilutionx PUBLIC
  Source
  ${CMAKE_CURRENT_BINARY_DIR})

if(NOT NINTENDO_3DS)
  target_link_libraries(libdevilutionx PUBLIC
  Threads::Threads)
endif()

target_link_libraries(libdevilutionx PUBLIC
  PKWare
  StormLib
  smacker
  simpleini)

if(WIN32)
  target_link_libraries(libdevilutionx PUBLIC find_steam_game)
endif()

if(NOT NONET)
  if(NOT DISABLE_TCP)
    target_link_libraries(libdevilutionx PUBLIC asio)
  endif()
  target_link_libraries(libdevilutionx PUBLIC sodium)
endif()

target_link_libraries(libdevilutionx PUBLIC fmt::fmt)
target_link_libraries(libdevilutionx PUBLIC PNG::PNG)

genex_for_option(DEBUG)
target_compile_definitions(libdevilutionx PUBLIC "$<${DEBUG_GENEX}:_DEBUG>")

if(NOT NONET AND NOT DISABLE_TCP)
  target_compile_definitions(libdevilutionx PUBLIC ASIO_STANDALONE)
endif()

# Defines without value
foreach(
  def_name
  NOSOUND
  NONET
  PREFILL_PLAYER_NAME
  DISABLE_TCP
  DISABLE_ZERO_TIER
  DISABLE_STREAMING_MUSIC
  DISABLE_STREAMING_SOUNDS
  GPERF
  GPERF_HEAP_MAIN
  GPERF_HEAP_FIRST_GAME_ITERATION
  STREAM_ALL_AUDIO
  VIRTUAL_GAMEPAD
)
if(${def_name})
  list(APPEND def_list ${def_name})
endif()
endforeach(def_name)

# Defines with value
foreach(
  def_name
  DEFAULT_WIDTH
  DEFAULT_HEIGHT
  DEFAULT_AUDIO_SAMPLE_RATE
  DEFAULT_AUDIO_CHANNELS
  DEFAULT_AUDIO_BUFFER_SIZE
  DEFAULT_AUDIO_RESAMPLING_QUALITY
  MO_LANG_DIR
  TTF_FONT_DIR
  TTF_FONT_NAME
  SDL1_VIDEO_MODE_BPP
  SDL1_VIDEO_MODE_FLAGS
  SDL1_FORCE_SVID_VIDEO_MODE
  SDL1_FORCE_DIRECT_RENDER
  HAS_KBCTRL
  KBCTRL_BUTTON_DPAD_LEFT
  KBCTRL_BUTTON_DPAD_RIGHT
  KBCTRL_BUTTON_DPAD_UP
  KBCTRL_BUTTON_DPAD_DOWN
  KBCTRL_BUTTON_B
  KBCTRL_BUTTON_A
  KBCTRL_BUTTON_Y
  KBCTRL_BUTTON_X
  KBCTRL_BUTTON_LEFTSTICK
  KBCTRL_BUTTON_RIGHTSTICK
  KBCTRL_BUTTON_RIGHTSHOULDER
  KBCTRL_BUTTON_LEFTSHOULDER
  KBCTRL_BUTTON_TRIGGERLEFT
  KBCTRL_BUTTON_TRIGGERRIGHT
  KBCTRL_BUTTON_START
  KBCTRL_BUTTON_BACK
  KBCTRL_IGNORE_1
  JOY_AXIS_LEFTX
  JOY_AXIS_LEFTY
  JOY_AXIS_RIGHTX
  JOY_AXIS_RIGHTY
  JOY_HAT_DPAD_UP_HAT
  JOY_HAT_DPAD_UP
  JOY_HAT_DPAD_DOWN_HAT
  JOY_HAT_DPAD_DOWN
  JOY_HAT_DPAD_LEFT_HAT
  JOY_HAT_DPAD_LEFT
  JOY_HAT_DPAD_RIGHT_HAT
  JOY_HAT_DPAD_RIGHT
  JOY_BUTTON_DPAD_LEFT
  JOY_BUTTON_DPAD_RIGHT
  JOY_BUTTON_DPAD_UP
  JOY_BUTTON_DPAD_DOWN
  JOY_BUTTON_B
  JOY_BUTTON_A
  JOY_BUTTON_Y
  JOY_BUTTON_X
  JOY_BUTTON_LEFTSTICK
  JOY_BUTTON_RIGHTSTICK
  JOY_BUTTON_RIGHTSHOULDER
  JOY_BUTTON_LEFTSHOULDER
  JOY_BUTTON_TRIGGERLEFT
  JOY_BUTTON_TRIGGERRIGHT
  JOY_BUTTON_START
  JOY_BUTTON_BACK
  REMAP_KEYBOARD_KEYS
)
  if(DEFINED ${def_name})
    list(APPEND def_list ${def_name}=${${def_name}})
  endif()
endforeach(def_name)

genex_for_option(UBSAN)
target_compile_options(libdevilutionx PUBLIC $<${UBSAN_GENEX}:-fsanitize=undefined>)
target_link_libraries(libdevilutionx PUBLIC $<${UBSAN_GENEX}:-fsanitize=undefined>)

if(TSAN)
  target_compile_options(libdevilutionx PUBLIC -fsanitize=thread)
  target_link_libraries(libdevilutionx PUBLIC -fsanitize=thread)
else()
  genex_for_option(ASAN)
  target_compile_options(libdevilutionx PUBLIC "$<${ASAN_GENEX}:-fsanitize=address;-fsanitize-recover=address>")
  target_link_libraries(libdevilutionx PUBLIC "$<${ASAN_GENEX}:-fsanitize=address;-fsanitize-recover=address>")
endif()

if(USE_SDL1)
  target_link_libraries(libdevilutionx PUBLIC
    ${SDL_TTF_LIBRARY}
    ${SDL_LIBRARY})
  target_compile_definitions(libdevilutionx PUBLIC USE_SDL1)
else()
  target_link_libraries(libdevilutionx PUBLIC
    SDL2::SDL2
    ${SDL2_MAIN}
    SDL2::SDL2_ttf)
endif()
target_link_libraries(libdevilutionx PUBLIC SDL_image)

if(NOT NOSOUND)
  target_link_libraries(libdevilutionx PUBLIC SDL_audiolib)
endif()

if(AMIGA)
  target_link_libraries(libdevilutionx PUBLIC
    ${FREETYPE_LIBRARY}
    ${ZLIB_LIBRARY})
  if(NOT WARPOS)
    target_link_libraries(libdevilutionx PUBLIC -ldebug)
  endif()
endif()

if (VITA)
  target_link_libraries(libdevilutionx PUBLIC
        ScePower_stub
        SceAppUtil_stub
        freetype
        z
  )
  target_compile_definitions(libdevilutionx PUBLIC VITA)
endif()

if(NINTENDO_3DS)
  target_link_libraries(libdevilutionx PUBLIC 3ds::freetype 3ds::bzip2)
  target_link_libraries(libdevilutionx PUBLIC 3ds::citro3d 3ds::ctrulib)
endif()

target_compile_definitions(libdevilutionx PUBLIC ${def_list})

if (GPERF)
  target_link_libraries(libdevilutionx PUBLIC ${GPERFTOOLS_LIBRARIES})
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU" AND NOT GPERF AND NOT HAIKU AND NOT VITA)
  target_link_libraries(libdevilutionx PUBLIC "$<$<NOT:$<CONFIG:Debug>>:-static-libgcc;-static-libstdc++>")
endif()

if(WIN32)
  target_link_libraries(libdevilutionx PUBLIC shlwapi wsock32 ws2_32 wininet)

  if(NOT CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    target_compile_options(libdevilutionx PUBLIC $<$<CONFIG:Debug>:-gstabs>)
  endif()
endif()

if(NOT WIN32 AND NOT APPLE AND NOT ${CMAKE_SYSTEM_NAME} STREQUAL FreeBSD)
  # Enable POSIX extensions such as `readlink` and `ftruncate`.
  add_definitions(-D_POSIX_C_SOURCE=200809L)
endif()

if(HAIKU)
  target_link_libraries(libdevilutionx PUBLIC network)
endif()

if(NOT CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
  # Change __FILE__ to only show the path relative to the project folder
  get_target_property(libdevilutionx_SRCS ${BIN_TARGET} SOURCES)
  foreach(SOURCE_FILE ${libdevilutionx_SRCS})
    set_source_files_properties(${SOURCE_FILE} PROPERTIES
      COMPILE_DEFINITIONS __FILE__="${SOURCE_FILE}"
    )
  endforeach(SOURCE_FILE)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-builtin-macro-redefined")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-builtin-macro-redefined")

  # Note: For Valgrind support.
  genex_for_option(DEBUG)
  target_compile_options(libdevilutionx PUBLIC $<${DEBUG_GENEX}:-fno-omit-frame-pointer>)

  # Warnings for devilutionX
  target_compile_options(libdevilutionx PUBLIC -Wall -Wextra -Wno-unused-parameter)

  # For ARM and other default unsigned char platforms
  target_compile_options(libdevilutionx PUBLIC -fsigned-char)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
  target_compile_options(libdevilutionx PUBLIC "/W3")
  target_compile_definitions(libdevilutionx PUBLIC _CRT_SECURE_NO_WARNINGS)
  target_compile_options(libdevilutionx PUBLIC "/Zc:__cplusplus")
endif()

if(APPLE)
  set_source_files_properties("./Packaging/macOS/AppIcon.icns" PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
  set_source_files_properties("./Packaging/resources/CharisSILB.ttf" PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
  set(MACOSX_BUNDLE_GUI_IDENTIFIER com.diasurgical.devilutionx)
  set(MACOSX_BUNDLE_COPYRIGHT Unlicense)
  set(MACOSX_BUNDLE_BUNDLE_NAME devilutionx)
  set(MACOSX_BUNDLE_INFO_STRING ${PROJECT_VERSION})
  set(MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION})
  set(MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION})
  set(MACOSX_BUNDLE_LONG_VERSION_STRING "Version ${PROJECT_VERSION}")
  set(CMAKE_OSX_DEPLOYMENT_TARGET "10.12.0")
  set_target_properties(${BIN_TARGET} PROPERTIES MACOSX_BUNDLE_ICON_FILE "AppIcon")
  set_target_properties(${BIN_TARGET} PROPERTIES MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/Packaging/macOS/Info.plist")

  install (TARGETS ${BIN_TARGET} DESTINATION ./)

  if(DIST)
      install(CODE "
        include(BundleUtilities)
        fixup_bundle(${CMAKE_BINARY_DIR}/${MACOSX_BUNDLE_BUNDLE_NAME}.app \"\" \"\")
        "
        COMPONENT Runtime)
  endif()

  set(MACOSX_BUNDLE_LONG_VERSION_STRING "Version ${PROJECT_VERSION}")
  set(CPACK On)
endif()

if(NINTENDO_SWITCH)

  target_link_libraries(libdevilutionx PUBLIC
        freetype
        bz2
        z
  )

  nx_generate_nacp (${BIN_TARGET}.nacp
    NAME    "DevilutionX"
    AUTHOR  "Devilution Team"
    VERSION "${PROJECT_VERSION}"
  )

  nx_create_nro(${BIN_TARGET}
    NACP ${BIN_TARGET}.nacp
    ICON "${PROJECT_SOURCE_DIR}/Packaging/switch/icon.jpg"
  )

endif()

if(VITA)
  set(VITA_APP_NAME "devilutionX")
  set(VITA_TITLEID  "DVLX00001")
  set(VITA_VERSION  "01.00")
  set(VITA_MKSFOEX_FLAGS "${VITA_MKSFOEX_FLAGS} -d PARENTAL_LEVEL=1")
  set(VITA_MKSFOEX_FLAGS "${VITA_MKSFOEX_FLAGS} -d ATTRIBUTE2=12")
  vita_create_self(devilutionx.self devilutionx UNSAFE)
  vita_create_vpk(devilutionx.vpk ${VITA_TITLEID} devilutionx.self
    VERSION ${VITA_VERSION}
    NAME ${VITA_APP_NAME}
    FILE Packaging/vita/sce_sys/icon0.png sce_sys/icon0.png
    FILE Packaging/vita/sce_sys/pic0.png sce_sys/pic0.png
    FILE Packaging/vita/sce_sys/livearea/contents/bg.png sce_sys/livearea/contents/bg.png
    FILE Packaging/vita/sce_sys/livearea/contents/startup.png sce_sys/livearea/contents/startup.png
    FILE Packaging/vita/sce_sys/livearea/contents/logo0.png sce_sys/livearea/contents/logo0.png
    FILE Packaging/vita/sce_sys/livearea/contents/template.xml sce_sys/livearea/contents/template.xml
    FILE Packaging/resources/CharisSILB.ttf CharisSILB.ttf
    FILE Packaging/resources/devilutionx.mpq devilutionx.mpq
    ${VITA_TRANSLATIONS_LIST}
  )
endif()

if(NINTENDO_3DS)
  set(APP_TITLE       "DevilutionX")
  set(APP_DESCRIPTION "DevilutionX port for 3DS")
  set(APP_AUTHOR      "Diasurgical Team")
  set(APP_ICON        "${PROJECT_SOURCE_DIR}/Packaging/ctr/icon.png")
  set(APP_BANNER      "${PROJECT_SOURCE_DIR}/Packaging/ctr/banner.png")
  set(APP_AUDIO       "${CMAKE_BINARY_DIR}/banner_audio.wav")
  set(APP_RSF         "${PROJECT_SOURCE_DIR}/Packaging/ctr/template.rsf")
  set(APP_ROMFS       "${CMAKE_BINARY_DIR}/romfs")
  list(APPEND APP_ROMFS_FILES
    ${PROJECT_SOURCE_DIR}/Packaging/resources/CharisSILB.ttf
    ${PROJECT_SOURCE_DIR}/Packaging/resources/devilutionx.mpq
    ${devilutionx_TRANSLATIONS})
  set(APP_VERSION ${PROJECT_VERSION})

  find_program(FFMPEG ffmpeg)
  if(FFMPEG)
    add_custom_command(OUTPUT ${APP_AUDIO}
      COMMAND ${FFMPEG} -y -ss 3.3 -t 3 -i "${PROJECT_SOURCE_DIR}/Packaging/resources/shareware-startup.wav" -af "afade=t=in:st=0:d=0.1,afade=t=out:st=2.9:d=0.1" ${APP_AUDIO}
      DEPENDS ${PROJECT_SOURCE_DIR}/Packaging/resources/shareware-startup.wav
      VERBATIM)
  else()
    add_custom_command(OUTPUT ${APP_AUDIO}
      COMMAND ${CMAKE_COMMAND} -E copy ${PROJECT_SOURCE_DIR}/Packaging/ctr/audio_silent.wav ${APP_AUDIO}
      DEPENDS ${PROJECT_SOURCE_DIR}/Packaging/ctr/audio_silent.wav
      VERBATIM)
  endif()

  add_custom_target(romfs_directory
    COMMAND ${CMAKE_COMMAND} -E make_directory ${APP_ROMFS})

  add_custom_target(romfs_files
    COMMAND ${CMAKE_COMMAND} -E copy ${APP_ROMFS_FILES} ${APP_ROMFS}
    DEPENDS romfs_directory)

  include(Tools3DS)
  add_3dsx_target(${BIN_TARGET})
  add_cia_target(${BIN_TARGET} ${APP_RSF} ${APP_BANNER} ${APP_AUDIO})

  get_filename_component(APP_TARGET_PREFIX ${BIN_TARGET} NAME_WE)
  add_dependencies(${APP_TARGET_PREFIX}_3dsx romfs_files)
  add_dependencies(${APP_TARGET_PREFIX}_cia romfs_files)
endif()

if(CPACK)
  if(WIN32)
    if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
      set(SDL2_WIN32_DLLS_DIR "${CMAKE_BINARY_DIR}")
    else()
      set(SDL2_WIN32_DLLS_DIR "${SDL2_EXEC_PREFIX}/bin")
    endif()
    set(SDL2_WIN32_LICENSES_DIR "${PROJECT_SOURCE_DIR}/Packaging/resources")

    file(GLOB SDL2_WIN32_ALL_DLLS
      LIST_DIRECTORIES false
      "${SDL2_WIN32_DLLS_DIR}/*.dll")
    file(GLOB SDL2_WIN32_ALL_LICENSES
      LIST_DIRECTORIES false
      "${SDL2_WIN32_LICENSES_DIR}/LICENSE*.txt"
      "${SDL2_WIN32_LICENSES_DIR}/README*.txt")

    set(CPACK_PACKAGE_FILE_NAME "devilutionx")
    set(CPACK_PACKAGE_NAME ${project_name})
    set(CPACK_GENERATOR "ZIP")
    set(CPACK_STRIP_FILES TRUE)
    install(TARGETS ${BIN_TARGET} DESTINATION .)
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/windows/README.txt"
      DESTINATION "."
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/CharisSILB.ttf"
      DESTINATION "."
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/devilutionx.mpq"
      DESTINATION "."
    )

    foreach(_SDL2_WIN32_DLL_PATH ${SDL2_WIN32_ALL_DLLS})
      install(FILES "${_SDL2_WIN32_DLL_PATH}"
        DESTINATION "."
      )
    endforeach()

    foreach(_SDL2_WIN32_LICENSE_PATH ${SDL2_WIN32_ALL_LICENSES})
      install(FILES "${_SDL2_WIN32_LICENSE_PATH}"
        DESTINATION "LICENSE"
      )
    endforeach()

  elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    string(TOLOWER ${PROJECT_NAME} project_name)
    set(CPACK_PACKAGE_NAME ${project_name})
    set(CPACK_GENERATOR "7Z")
    # Common *nix files
    set(CPACK_STRIP_FILES TRUE)
    install(TARGETS ${BIN_TARGET} DESTINATION bin)
    set(desktop_file "${PROJECT_SOURCE_DIR}/Packaging/nix/${project_name}.desktop")
    set(desktop_file_hellfire "${PROJECT_SOURCE_DIR}/Packaging/nix/${project_name}-hellfire.desktop")

    find_program(DFI desktop-file-install)
    if(DFI)
      execute_process(COMMAND ${DFI} --dir=${CMAKE_BINARY_DIR} ${desktop_file})
      set(desktop_file "${CMAKE_BINARY_DIR}/${project_name}.desktop")
      execute_process(COMMAND ${DFI} --dir=${CMAKE_BINARY_DIR} ${desktop_file_hellfire})
      set(desktop_file_hellfire "${CMAKE_BINARY_DIR}/${project_name}-hellfire.desktop")
    endif()

    install(FILES "${desktop_file}"
      DESTINATION "share/applications"
    )
    install(FILES "${desktop_file_hellfire}"
      DESTINATION "share/applications"
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/nix/README.txt"
      DESTINATION "share/diasurgical/${project_name}"
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/devilutionx.mpq"
      DESTINATION "share/diasurgical/${project_name}"
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/icon.png"
      DESTINATION "share/icons/hicolor/512x512/apps"
      RENAME "${project_name}.png"
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/hellfire.png"
      DESTINATION "share/icons/hicolor/512x512/apps"
      RENAME "${project_name}-hellfire.png"
    )
    install(FILES "${PROJECT_SOURCE_DIR}/Packaging/resources/CharisSILB.ttf"
      DESTINATION "share/fonts/truetype"
    )

    # -G DEB
    set(CPACK_PACKAGE_CONTACT "anders@jenbo.dk")
    if(USE_SDL1)
    set(CPACK_DEBIAN_PACKAGE_DEPENDS "libsdl1.2debian, libsdl-ttf2.0-0")
    else()
    set(CPACK_DEBIAN_PACKAGE_DEPENDS "libsdl2-2.0-0, libsdl2-ttf-2.0-0")
    endif()
    set(CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)

    # -G RPM
    set(CPACK_RPM_FILE_NAME RPM-DEFAULT)

    find_program(RPMBUILD rpmbuild)
    if(RPMBUILD)
      list(APPEND CPACK_GENERATOR "RPM")
    endif()
    find_program(DPKG dpkg)
    if(DPKG)
      list(APPEND CPACK_GENERATOR "DEB")
    endif()

  elseif(APPLE)
    set(CPACK_PACKAGE_FILE_NAME "devilutionx")
    set(CPACK_DMG_DISABLE_APPLICATIONS_SYMLINK "ON")
    set(CPACK_STRIP_FILES TRUE)
    set(CPACK_GENERATOR "DragNDrop")
  endif()

  set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
  include(CPack)
endif()

if(NOT NONET AND NOT DISABLE_ZERO_TIER)
  add_subdirectory(3rdParty/libzt)
  target_link_libraries(libdevilutionx PUBLIC zt-static)
endif()
