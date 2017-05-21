cmake_minimum_required(VERSION 2.8.11)

enable_testing()

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

add_library(librcksum STATIC rsum.c hash.c state.c range.c md4.c)
set_target_properties(librcksum PROPERTIES PREFIX "")

add_executable(md4test md4test.c md4.c)
add_test(md4test md4test)
