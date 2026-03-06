cmake_minimum_required(VERSION 3.5...3.24)
include(FetchContent)
include(CSharpUtilities)
include(CMakePackageConfigHelpers)

set(CMAKE_SYSTEM_VERSION 10.0 CACHE STRING INTERNAL FORCE)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_DEBUG_POSTFIX d)
set(FETCHCONTENT_QUIET FALSE)
set(MY_PRODUCT_NUMBER 1)
set(MY_PRODUCT_VERSION 0)
set(MY_BUILD_NUMBER 0)
set(VERSION "${MY_PRODUCT_NUMBER}.${MY_PRODUCT_VERSION}.${MY_BUILD_NUMBER}")

project(steamdatapp VERSION ${VERSION})
enable_language(CXX ASM CSharp)
include(GNUInstallDirs)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX
    "${CMAKE_BINARY_DIR}/rundir"
    CACHE STRING "Directory to install after building" FORCE)
endif()

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

option(MSVC_STATIC_RUNTIME "static crt" OFF)
option(DISABLE_INSTALL "Disable installation targets" OFF)

if(MSVC_STATIC_RUNTIME)
  if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
  endif()

  set(STATIC_CRT STATIC_CRT)
else()
  if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
  endif()

  unset(STATIC_CRT)
endif()

FetchContent_Declare(
  OCmakeUtil
  GIT_REPOSITORY git@github.com:wsw364321644/OCmakeUtil.git
  GIT_TAG main
)
FetchContent_MakeAvailable(OCmakeUtil)
include(ProjectImport)
include(TargetImport)
include(ProgramImport)
include(ResourceDownload)
cmake_host_system_information(RESULT HOST_IS_64BIT QUERY IS_64BIT)

ImportProject(absl ${STATIC_CRT} SSH TAG 20260107.1)
ImportProject(Protobuf ${STATIC_CRT} SSH TAG v34.0)
ImportProject(re2 ${STATIC_CRT} SSH TAG 2025-11-05)
ImportProject(simdjson ${STATIC_CRT} ${SSH} TAG v4.2.4)
ImportProject(spdlog ${STATIC_CRT} STATIC SSH TAG v1.17.0)

ImportProgram(SteamLanguageParser SSH TAG origin/master)

ResourceDownload(
  SteamKit
  GIT_REPOSITORY https://github.com/SteamRE/SteamKit.git
  GIT_TAG 3.4.0
)

ResourceDownload(
  SteamDBProtobufs
  GIT_REPOSITORY https://github.com/SteamDatabase/Protobufs.git
  GIT_TAG origin/master
)
add_subdirectory(src/steam_language)
add_subdirectory(src/steam_protobufs)

if(NOT DISABLE_INSTALL)
  ExportFromInstall(${PROJECT_NAME})
endif()