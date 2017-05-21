cmake_minimum_required(VERSION 2.8.11)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

add_library(libinflate STATIC inflate.c adler32.c inftrees.c zutil.c crc32.c)
set_target_properties(libinflate PROPERTIES PREFIX "")

add_library(libdeflate STATIC deflate.c compress.c trees.c)
set_target_properties(libdeflate PROPERTIES PREFIX "")
