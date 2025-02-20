# get the millennium version from version file
file(STRINGS "${CMAKE_SOURCE_DIR}/version" VERSION_LINES LIMIT_COUNT 2)
list(GET VERSION_LINES 1 MILLENNIUM_VERSION)
set(MILLENNIUM_VERSION "${MILLENNIUM_VERSION}")

message(STATUS "Millennium Version: ${MILLENNIUM_VERSION}")

cmake_minimum_required(VERSION 3.5.0)
set(BUILD_SHARED_LIBS OFF)

# set c++ directives
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD_REQUIRED ON)

if (NOT APPLE)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m32")
endif()

if (WIN32)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -s")
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS OFF)
endif()

project(Millennium LANGUAGES CXX)

if (WIN32 AND NOT GITHUB_ACTION_BUILD)
  # debug output paths
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "C:/Program Files (x86)/Steam")
  set(LIBRARY_OUTPUT_DIRECTORY "C:/Program Files (x86)/Steam")
elseif(UNIX AND NOT GITHUB_ACTION_BUILD)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "$ENV{HOME}/.millennium/")
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "$ENV{HOME}/.millennium/")
endif()

# Set version information
add_compile_definitions(MILLENNIUM_VERSION="${MILLENNIUM_VERSION}")

if (WIN32)
  string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
  string(REPLACE "/MDd" "/MTd" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
elseif(UNIX)
  set(CMAKE_CXX_FLAGS "-fpermissive")
endif()

# build boxer (message-box lib)
# add_subdirectory(vendor/boxer)

include_directories(
  ${CMAKE_SOURCE_DIR}/src
  ${CMAKE_SOURCE_DIR}/vendor/fmt/include
  ${CMAKE_SOURCE_DIR}/vendor/asio/asio/include
  ${CMAKE_SOURCE_DIR}/vendor/nlohmann/include
  ${CMAKE_SOURCE_DIR}/vendor/websocketpp
  ${CMAKE_SOURCE_DIR}/vendor/crow/include
  ${CMAKE_SOURCE_DIR}/vendor/ini/src
)

add_compile_definitions(
  CURL_STATICLIB
  _WEBSOCKETPP_CPP11_THREAD_
  _WEBSOCKETPP_CPP11_TYPE_TRAITS_
  _WEBSOCKETPP_CPP11_RANDOM_DEVICE_
  ASIO_STANDALONE
  ASIO_HAS_STD_INVOKE_RESULT
  FMT_HEADER_ONLY
  _CRT_SECURE_NO_WARNINGS
)

if (WIN32)
  add_subdirectory(win32)
endif()

add_subdirectory(cli)

find_package(CURL REQUIRED) # used for web requests. 
include_directories(${LIBGIT2_INCLUDE_DIRS})

if(WIN32)
  include_directories(${CMAKE_SOURCE_DIR}/vendor/python/include)
elseif(UNIX)
  if (APPLE)
    include_directories("$ENV{HOME}/.pyenv/versions/3.11.8/include/python3.11")
  else()
    include_directories("$ENV{HOME}/Documents/LibPython/")
  endif()
endif()

set(SOURCE_FILES 
  "src/main.cc"
  "src/core/loader.cc"
  "src/core/py_controller/co_spawn.cc"
  "src/core/py_controller/logger.cc"
  "src/core/ffi/c_python.cc"
  "src/core/ffi/javascript.cc"
  "src/core/ffi/gil.cc"
  "src/core/co_initialize/co_stub.cc"
  "src/core/co_initialize/events.cc"
  "src/core/hooks/web_load.cc"
  "src/core/ipc/pipe.cc"
  "src/core/ftp/serv.cc"
  "src/sys/log.cc"
  "src/sys/io.cc"
  "src/sys/settings.cc"
  "src/api/executor.cc"
)

if (MSVC)
  set(SOURCE_FILES "${SOURCE_FILES} version.rc") # compile version information on msvc
endif()

if (WIN32)
  add_library(Millennium SHARED "${SOURCE_FILES}")
  # add_executable(Millennium "${SOURCE_FILES}" version.rc)
elseif(UNIX)
  # add_executable(Millennium "${SOURCE_FILES}")
  # add_compile_definitions(MILLENNIUM_EXECUTABLE)
  add_library(Millennium SHARED "${SOURCE_FILES}")
  add_compile_definitions(MILLENNIUM_SHARED)
endif()

if (NOT APPLE)
  set_target_properties(Millennium PROPERTIES COMPILE_FLAGS "-m32" LINK_FLAGS "-m32")
  target_compile_options (Millennium PRIVATE -m32 )
endif()

if (WIN32)
  set_target_properties(Millennium PROPERTIES OUTPUT_NAME "millennium")
  set_target_properties(Millennium PROPERTIES PREFIX "")
  set_target_properties(Millennium PROPERTIES NO_EXPORT TRUE)
endif()

if(MSVC)
    # prevent MSVC from generating .lib and .exp archives
    set_target_properties(Millennium PROPERTIES ARCHIVE_OUTPUT_NAME "" LINK_FLAGS "/NOEXP")
endif()

if (GITHUB_ACTION_BUILD)
    message("${GITHUB_ACTION_BUILD}")
endif()

find_program(WINDRES windres)
if (WINDRES)
    add_custom_command(
        OUTPUT ${CMAKE_BINARY_DIR}/version.o
        COMMAND ${WINDRES} -i ${CMAKE_SOURCE_DIR}/scripts/version.rc -o ${CMAKE_BINARY_DIR}/version.o
        DEPENDS ${CMAKE_SOURCE_DIR}/scripts/version.rc
    )

    add_custom_target(resource DEPENDS ${CMAKE_BINARY_DIR}/version.o)
    add_dependencies(Millennium resource)
    target_link_libraries(Millennium ${CMAKE_BINARY_DIR}/version.o)
endif()

target_link_libraries(Millennium CURL::libcurl)

if(WIN32)
	target_link_libraries(Millennium Ws2_32.lib wsock32 Iphlpapi winhttp)

	if (GITHUB_ACTION_BUILD)
		target_link_libraries(Millennium "D:/a/Millennium/Millennium/Python-3.11.8/PCbuild/win32/python311.lib")
	else()
		target_link_libraries(Millennium ${CMAKE_SOURCE_DIR}/vendor/python/python311.lib ${CMAKE_SOURCE_DIR}/vendor/python/python311_d.lib)
	endif()
elseif(UNIX)
	if (APPLE)
		target_link_libraries(Millennium "$ENV{HOME}/.pyenv/versions/3.11.8/lib/libpython3.11.dylib")
	else()
		if (GITHUB_ACTION_BUILD)
			target_link_libraries(Millennium "$ENV{HOME}/.millennium/libpython-3.11.8.so")
		else()
			target_link_libraries(Millennium "$ENV{HOME}/Documents/LibPython/libpython-3.11.8.so")
		endif()
	endif()
endif()
