# requires CMake >= 3.1.3 due to use of CMAKE_C_STANDARD
cmake_minimum_required(VERSION 3.1.3)

project(zsync_curl)

include(CTest)

include(CheckFunctionExists)
include(CheckIncludeFiles)

# platform checks
foreach(header inttypes memory stdint stdlib strings sys/stat sys/types unistd)
    string(REPLACE "/" "_" underscore_header "${header}")
    string(TOUPPER ${underscore_header} upper_header)
    check_include_files(${header}.h HAVE_${upper_header})
    if(HAVE_${upper_header})
        add_definitions(-DHAVE_${upper_header}_H=1)
    endif()
endforeach()

foreach(function fseeko getaddrinfo memcpy mkstemp pread pwrite)
    string(TOUPPER ${function} upper_function)
    check_function_exists(${function} HAVE_${upper_function})
    if(HAVE_${upper_function})
        add_definitions(-DHAVE_${upper_function}=1)
    endif()
endforeach()

find_library(LIBCURL curl REQUIRED)
find_library(LIBSOCKET socket REQUIRED)

if(LIBCURL_FOUND)
    add_definitions(-DHAVE_LIBCURL=1)
endif()
if(LIBSOCKET_FOUND)
    add_definitions(-DHAVE_LIBSOCKET=1)
endif()

# config.h contains some global defines, our config.h.in is a modified version of the upstream config.h.in to make it integrate with CMake
set(VERSION "0.6.2")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

set(CMAKE_C_STANDARD 99)
add_definitions(-DHAVE_CONFIG_H)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffast-math")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -s")

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

add_subdirectory(doc)
add_subdirectory(librcksum)
add_subdirectory(zlib)

add_subdirectory(libzsync)

find_package(CURL REQUIRED)

if(NOT CURL_FOUND)
    message(FATAL_ERROR "libcurl not found!")
endif()

include_directories(${CURL_INCLUDE_DIR})

add_executable(zsync_curl progress.c http.c url.c base64.c client.c)
target_link_libraries(zsync_curl libzsync ${CURL_LIBRARY})

install(TARGETS zsync_curl DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

# fixes error: 'PACKAGE' undeclared and its friends
set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/client.c PROPERTY COMPILE_FLAGS "-include config.h")


