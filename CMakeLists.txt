# requires CMake >= 3.1.3 due to use of CMAKE_C_STANDARD
cmake_minimum_required(VERSION 3.1.3)

project(zsync_curl)

set(USE_CURL_WITH_MBEDTLS OFF CACHE BOOL "download, build and link to cURL with mbedtls instead of using the system libraries")

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

# zsync_curl's code base is pretty old
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffast-math")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -s")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)
# contains generated config.h
include_directories(${CMAKE_CURRENT_BINARY_DIR})

if(${USE_CURL_WITH_MBEDTLS})
    message(STATUS "downloading, building and linking to cURL with mbedtls instead of using the system libraries")

    # use ExternalProject feature to fetch dependencies
    include(ExternalProject)

    # to avoid setting INSTALL_COMMAND to "" (which returns in a library rebuild on every build -> highly inefficient)
    # we use the solution from https://stackoverflow.com/a/37855224
    # TODO: fix "warning: jobserver unavailable: using -j1" to speed up builds
    ExternalProject_Add(mbedtls
        GIT_REPOSITORY https://github.com/ARMmbed/mbedtls.git
        BUILD_COMMAND ${CMAKE_COMMAND} --build . --target mbedtls
        CMAKE_ARGS
            -DUSE_SHARED_MBEDTLS_LIBRARY=On
        INSTALL_COMMAND cmake -E echo "Skipping install step."
        UPDATE_DISCONNECTED 1
    )

    ExternalProject_Get_Property(mbedtls source_dir)
    ExternalProject_Get_Property(mbedtls binary_dir)

    set(MBEDTLS_INCLUDE_DIR ${source_dir}/include)
    set(MBEDTLS_LIBRARY_DIR ${binary_dir}/library)

    ExternalProject_Add(curl
        DEPENDS mbedtls
        GIT_REPOSITORY https://github.com/TheAssassin/curl.git
        GIT_TAG patch-1
        BUILD_COMMAND ${CMAKE_COMMAND} --build . --target libcurl
        CMAKE_ARGS
            -DHTTP_ONLY=1
            -DCMAKE_USE_LIBSSH2=0
            -DCMAKE_USE_MBEDTLS=1
            -DMBEDTLS_INCLUDE_DIRS=${MBEDTLS_INCLUDE_DIR}
            -DMBEDTLS_LIBRARY=${MBEDTLS_LIBRARY_DIR}/libmbedtls.so
            -DMBEDX509_LIBRARY=${MBEDTLS_LIBRARY_DIR}/libmbedx509.so
            -DMBEDCRYPTO_LIBRARY=${MBEDTLS_LIBRARY_DIR}/libmbedcrypto.so
        INSTALL_COMMAND cmake -E echo "Skipping install step."
        UPDATE_DISCONNECTED 1
    )

    ExternalProject_Get_Property(curl source_dir)
    ExternalProject_Get_Property(curl binary_dir)

    set(CURL_SOURCE_DIR ${source_dir})
    set(CURL_BINARY_DIR ${binary_dir})

    # set include directories, including the ones with auto generated files in binary_dir
    include_directories(${CURL_SOURCE_DIR}/include)
    include_directories(${CURL_BINARY_DIR}/include)
    include_directories(${CURL_BINARY_DIR}/include/curl)

    # set path to libcurl
    set(CURL_LIBRARY ${CURL_BINARY_DIR}/lib/libcurl.so)

# link to system wide curl, e.g. for testing
else()
    message(STATUS linking to system cURL)

    find_library(LIBCURL curl REQUIRED)
    find_library(LIBSOCKET socket REQUIRED)

    if(LIBCURL_FOUND)
        add_definitions(-DHAVE_LIBCURL=1)
    endif()
    if(LIBSOCKET_FOUND)
        add_definitions(-DHAVE_LIBSOCKET=1)
    endif()

    find_package(CURL REQUIRED)

    if(NOT CURL_FOUND)
        message(FATAL_ERROR "libcurl not found!")
    endif()

    include_directories(${CURL_INCLUDE_DIR})
endif()

# config.h contains some global defines, our config.h.in is a modified version of the upstream config.h.in to make it integrate with CMake
set(VERSION "0.6.2")
configure_file(src/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)
add_definitions(-DHAVE_CONFIG_H)

add_subdirectory(doc)
add_subdirectory(librcksum)
add_subdirectory(zlib)

add_subdirectory(libzsync)

add_executable(zsync_curl src/progress.c src/http.c src/url.c src/base64.c src/client.c)
target_link_libraries(zsync_curl libzsync ${CURL_LIBRARY})

if(${USE_CURL_WITH_MBEDTLS})
    # add the dependency manually so that the build is triggered in the right order
    add_dependencies(zsync_curl curl)
endif()

install(TARGETS zsync_curl DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

# fixes error: 'PACKAGE' undeclared and its friends
set_source_files_properties(src/client.c PROPERTY COMPILE_FLAGS "-include config.h")


