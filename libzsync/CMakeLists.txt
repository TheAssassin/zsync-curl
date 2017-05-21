cmake_minimum_required(VERSION 2.8.11)

enable_testing()

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

add_library(libzsync STATIC zsync.c zmap.c sha1.c)
set_target_properties(libzsync PROPERTIES PREFIX "")
target_link_libraries(libzsync librcksum libdeflate libinflate)

add_executable(sha1test sha1.c sha1test.c)
add_test(sha1test sha1test)
