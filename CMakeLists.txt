cmake_minimum_required(VERSION 3.5)

# add modules
set (CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")

# policy for boost
if(POLICY CMP0167)
    cmake_policy(SET CMP0167 NEW)
endif()

# set version
set(LIBNPA_VERSION_MAJOR "0")
set(LIBNPA_VERSION_MINOR "5")
set(LIBNPA_VERSION_PATCH "3")
set(LIBNPA_VERSION "${LIBNPA_VERSION_MAJOR}.${LIBNPA_VERSION_MINOR}.${LIBNPA_VERSION_PATCH}")

# make sure all symbols are exported on Windows
# currently requires cmake-dev (CMake 3.3.20150721 or newer)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

# append git revision if available
include(get-git-revision)
get_git_revision(LIBNPA_VERSION_GIT)
if(NOT "${LIBNPA_VERSION_GIT}" STREQUAL "")
    set(LIBNPA_VERSION "${LIBNPA_VERSION}-${LIBNPA_VERSION_GIT}")
endif()

message(STATUS "Configuring libnpa version " ${LIBNPA_VERSION})

# project name and language
project (libnpa CXX)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -O2")

# include version number in header
configure_file(${PROJECT_SOURCE_DIR}/include/npaversion.hpp.in
    ${PROJECT_SOURCE_DIR}/include/npaversion.hpp)

find_package(Boost REQUIRED COMPONENTS system locale filesystem CONFIG)
find_package(ZLIB REQUIRED)
find_package(FLEX REQUIRED)
find_package(BISON REQUIRED)

string(REPLACE "." ";" VERSION_LIST ${FLEX_VERSION})
list(GET VERSION_LIST 0 FLEX_VERSION_MAJOR)
list(GET VERSION_LIST 1 FLEX_VERSION_MINOR)
list(GET VERSION_LIST 2 FLEX_VERSION_PATCH)
add_definitions(-DFLEX_VERSION_MAJOR=${FLEX_VERSION_MAJOR})
add_definitions(-DFLEX_VERSION_MINOR=${FLEX_VERSION_MINOR})
add_definitions(-DFLEX_VERSION_PATCH=${FLEX_VERSION_PATCH})

include_directories(${PROJECT_SOURCE_DIR}/include
    ${Boost_INCLUDE_DIR}
    ${ZLIB_INCLUDE_DIR}
)

# msvc doesn't have unistd.h, the equivalent is io.h
# thus, a dummy unistd.h that includes io.h is used
# if msvc is detected
if(MSVC)
	include_directories(${PROJECT_SOURCE_DIR}/include/msvc)
endif()

flex_target(lexer ${CMAKE_SOURCE_DIR}/src/lexer.l ${CMAKE_SOURCE_DIR}/src/lexer.cpp)
bison_target(parser ${CMAKE_SOURCE_DIR}/src/parser.y ${CMAKE_SOURCE_DIR}/src/parser.cpp)
add_flex_bison_dependency(lexer parser)

set(LIBNPA_SOURCES
    src/inpafile.cpp
    src/npafile.cpp
    src/nsbmagic.cpp
    src/onpafile.cpp
    src/scriptfile.cpp
    src/exefile.cpp
    src/fscommon.cpp
    src/inipafile.cpp
    src/isgfile.cpp
    src/npakeys.cpp
    src/buffer.cpp
    src/nsscompiler.cpp
    ${BISON_parser_OUTPUTS}
    ${FLEX_lexer_OUTPUTS}
    src/nsbconstants.cpp
)
# Shared Library
add_library(npa_shared SHARED ${LIBNPA_SOURCES})
set_target_properties(npa_shared PROPERTIES OUTPUT_NAME "npa")
target_link_libraries(npa_shared ${Boost_LIBRARIES} ${ZLIB_LIBRARIES})

# Static Library
add_library(npa_static STATIC ${LIBNPA_SOURCES})
set_target_properties(npa_static PROPERTIES OUTPUT_NAME "npa")
target_link_libraries(npa_static ${Boost_LIBRARIES} ${ZLIB_LIBRARIES})

# install headers and library
install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/
    DESTINATION include/libnpa
    FILES_MATCHING PATTERN "*.hpp"
)
install(TARGETS npa_shared npa_static
    ARCHIVE DESTINATION lib   # Static
    LIBRARY DESTINATION lib   # Shared
)

# create packages
set(CPACK_GENERATOR "TBZ2")
set(CPACK_PACKAGE_VERSION ${LIBNPA_VERSION})
include(CPack)
