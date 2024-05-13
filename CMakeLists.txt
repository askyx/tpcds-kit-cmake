cmake_minimum_required(VERSION 3.22.0)

project(tpcds VERSION 2.10.0
  LANGUAGES CXX C)

set(CMAKE_EXPORT_COMPILE_COMMANDS on)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format -Wno-implicit-int -Wno-deprecated-declarations")

list(APPEND TPCDS_COMPILE_OPTIONS "-D_FILE_OFFSET_BITS=64" "-D_LARGEFILE_SOURCE" "-DLINUX")

add_executable(mkheader
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mkheader/mkheader.c)

target_compile_definitions(mkheader
  PRIVATE
    ${TPCDS_COMPILE_OPTIONS}
)

target_include_directories(mkheader PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src/include)

add_custom_command(
  OUTPUT
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tables.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/streams.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/columns.h
  COMMAND mkheader ${CMAKE_CURRENT_SOURCE_DIR}/src/mkheader/column_list.txt
  DEPENDS mkheader
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/include
)

add_custom_target(
  COLUMS_HEADERS
  DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tables.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/streams.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/columns.h
)

file(
  GLOB_RECURSE
  TPCDS_COMMON
  CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/common/*.c
)

add_library(tpcds_common STATIC ${TPCDS_COMMON})

target_include_directories(tpcds_common PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src/include)

target_compile_definitions(tpcds_common
  PUBLIC
  ${TPCDS_COMPILE_OPTIONS}
)

target_link_libraries(tpcds_common m)

add_dependencies(tpcds_common COLUMS_HEADERS)

file(
  GLOB_RECURSE
  TPCDS_SRC
  CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/dsdgen/*.c
)

add_executable(dsdgen ${TPCDS_SRC})

target_include_directories(dsdgen PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src/include)

target_compile_definitions(dsdgen
  PRIVATE
    ${TPCDS_COMPILE_OPTIONS}
)

target_link_libraries(dsdgen m tpcds_common)

add_dependencies(dsdgen COLUMS_HEADERS)

file(
  GLOB_RECURSE
  DISTCOMP_SRC
  CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/distcomp/*.c
)

add_executable(distcomp ${DISTCOMP_SRC})

target_compile_definitions(distcomp
  PRIVATE
    ${TPCDS_COMPILE_OPTIONS}
)

target_link_libraries(distcomp m)

add_custom_command(
  OUTPUT
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tpcds.idx.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tpcds.idx
  COMMAND distcomp -i tpcds.dst -o tpcds.idx
  DEPENDS distcomp
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/include
)

add_custom_target(DISCOMP
  DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tpcds.idx.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include/tpcds.idx
)

target_compile_definitions(distcomp
  PRIVATE
    ${TPCDS_COMPILE_OPTIONS}
)

target_include_directories(distcomp PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src/include)

target_link_libraries(distcomp m tpcds_common)

add_dependencies(distcomp COLUMS_HEADERS)
add_dependencies(dsdgen DISCOMP)

find_package(FLEX REQUIRED)
find_package(BISON 3.5.0 REQUIRED)

list(APPEND BISON_OPTIONS "-d -v")

bison_target(parser
    ${PROJECT_SOURCE_DIR}/src/dsqgen/qgen.y
    ${PROJECT_SOURCE_DIR}/src/dsqgen/y.tab.c
    COMPILE_FLAGS ${BISON_OPTIONS})

flex_target(scanner
    ${PROJECT_SOURCE_DIR}/src/dsqgen/tokenizer.l
    ${PROJECT_SOURCE_DIR}/src/dsqgen/tokenizer.c)

ADD_FLEX_BISON_DEPENDENCY(scanner parser)

file(
  GLOB_RECURSE
  TPCDS_LOAD_SRC
  CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/dsqgen/*.c
)

add_executable(dsqgen
  ${TPCDS_LOAD_SRC}
  ${BISON_parser_OUTPUTS}
  ${FLEX_scanner_OUTPUTS}
)

target_compile_definitions(dsqgen
  PRIVATE
    ${TPCDS_COMPILE_OPTIONS}
)

target_link_libraries(dsqgen m tpcds_common)

add_dependencies(dsqgen DISCOMP)