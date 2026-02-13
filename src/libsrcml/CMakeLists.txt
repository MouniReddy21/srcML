# SPDX-License-Identifier: GPL-3.0-only
##
# @file CMakeLists.txt
#
# @copyright Copyright (C) 2013-2024 srcML, LLC. (www.srcML.org)
#
# CMake files for the library libsrcml
find_package(LibXml2 REQUIRED)
find_package(LibXslt REQUIRED)
find_package(Iconv REQUIRED)
find_package(Threads REQUIRED)

file(GLOB LIBSRCML_SOURCE *.cpp)
add_library(libsrcml OBJECT ${LIBSRCML_SOURCE})
set_target_properties(libsrcml PROPERTIES POSITION_INDEPENDENT_CODE ON)

target_include_directories(libsrcml
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE .
)

target_include_directories(libsrcml BEFORE PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)

target_link_libraries(libsrcml PRIVATE 
    LibXml2::LibXml2 
    Iconv::Iconv 
    LibXslt::LibXslt 
    LibXslt::LibExslt 
    Threads::Threads 
    parser
)

set(LIBSRCML_OBJECTS $<TARGET_OBJECTS:libsrcml> $<TARGET_OBJECTS:parser> $<TARGET_OBJECTS:antlr>)

if(NOT MSVC)
    set(LIBSRCML_OBJECT_FILENAME libsrcml${CMAKE_CXX_OUTPUT_EXTENSION})
    
    set(archs ${CMAKE_OSX_ARCHITECTURES})
    list(LENGTH archs NUM_OSX_ARCHITECTURES)

    if(${CMAKE_SYSTEM_NAME} STREQUAL "Emscripten")
        # web assembly
        add_custom_command(
            OUTPUT ${LIBSRCML_OBJECT_FILENAME}
            COMMAND emcc -r ${LIBSRCML_OBJECTS} -o ${LIBSRCML_OBJECT_FILENAME}
            DEPENDS libsrcml parser antlr
            COMMAND_EXPAND_LISTS
        )
    elseif(${NUM_OSX_ARCHITECTURES} EQUAL 2)
        # macOS UNIVERSAL BINARY ---
        add_custom_command(
          OUTPUT ${LIBSRCML_OBJECT_FILENAME}
          COMMAND ld -r -arch arm64 ${LIBSRCML_OBJECTS} -o libsrcml.arm64.o
          COMMAND ld -r -arch x86_64 ${LIBSRCML_OBJECTS} -o libsrcml.x86_64.o
          COMMAND lipo -create libsrcml.arm64.o libsrcml.x86_64.o -output ${LIBSRCML_OBJECT_FILENAME}
          DEPENDS libsrcml parser antlr
          COMMAND_EXPAND_LISTS
        )
    elseif(${NUM_OSX_ARCHITECTURES} EQUAL 1)
        # macOS SINGLE ARCH ---
        add_custom_command(
          OUTPUT ${LIBSRCML_OBJECT_FILENAME}
          COMMAND ld -arch ${CMAKE_OSX_ARCHITECTURES} -r ${LIBSRCML_OBJECTS} -o ${LIBSRCML_OBJECT_FILENAME}
          DEPENDS libsrcml parser antlr
          COMMAND_EXPAND_LISTS
        )
    else()
        # STANDARD LINUX BRANCH ---
        add_custom_command(
          OUTPUT ${LIBSRCML_OBJECT_FILENAME}
          COMMAND ld -r ${LIBSRCML_OBJECTS} -o ${LIBSRCML_OBJECT_FILENAME}
          DEPENDS libsrcml parser antlr
          COMMAND_EXPAND_LISTS
        )
    endif()

    set(LIBSRCML_OBJECTS ${LIBSRCML_OBJECT_FILENAME})
endif()

add_library(libsrcml_static STATIC ${LIBSRCML_OBJECTS})
set_target_properties(libsrcml_static PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    POSITION_INDEPENDENT_CODE ON
    LINKER_LANGUAGE CXX
    CXX_VISIBILITY_PRESET hidden
    C_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON
)
target_include_directories(libsrcml_static PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)
target_link_libraries(libsrcml_static PRIVATE LibXml2::LibXml2 Iconv::Iconv LibXslt::LibXslt LibXslt::LibExslt)

if(WIN32)
    set(SRCML_LIBSRCML_SHARED_OUTPUT_NAME "libsrcml")
    set(SRCML_LIBSRCML_STATIC_OUTPUT_NAME "libsrcml_static")
else()
    set(SRCML_LIBSRCML_SHARED_OUTPUT_NAME "srcml")
    set(SRCML_LIBSRCML_STATIC_OUTPUT_NAME "srcml")
endif()
set_target_properties(libsrcml_static PROPERTIES OUTPUT_NAME "${SRCML_LIBSRCML_STATIC_OUTPUT_NAME}")

if(NOT EMSCRIPTEN)
    add_library(libsrcml_shared SHARED)
    set_target_properties(libsrcml_shared PROPERTIES 
        POSITION_INDEPENDENT_CODE ON
        ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
        VERSION "${PROJECT_VERSION}"
        SOVERSION "${PROJECT_VERSION_MAJOR}"
        CXX_VISIBILITY_PRESET hidden
        C_VISIBILITY_PRESET hidden
        VISIBILITY_INLINES_HIDDEN ON
        OUTPUT_NAME "${SRCML_LIBSRCML_SHARED_OUTPUT_NAME}"
    )
    target_link_libraries(libsrcml_shared PRIVATE libsrcml parser antlr)
    target_include_directories(libsrcml_shared PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)

    # packaging and CMake Config installation logic
    option(SRCML_INSTALL_CMAKE_CONFIG "Install the cmake configuration" ON)
    if(SRCML_INSTALL_CMAKE_CONFIG)
        include(CMakePackageConfigHelpers)
        set(CONFIG_FILE_DESTINATION "${CMAKE_INSTALL_DATADIR}/cmake/srcml")
        
        install(TARGETS libsrcml_static EXPORT libsrcml_static_Targets COMPONENT DEVLIBS)
        install(EXPORT libsrcml_static_Targets DESTINATION "${CONFIG_FILE_DESTINATION}" NAMESPACE srcML:: FILE srcML-static-targets.cmake COMPONENT DEVLIBS)
    endif()

    if(WIN32)
        install(TARGETS libsrcml_shared EXPORT libsrcml_shared_Targets RUNTIME COMPONENT SRCML ARCHIVE COMPONENT DEVLIBS)
    else()
        install(TARGETS libsrcml_shared EXPORT libsrcml_shared_Targets LIBRARY COMPONENT SRCML NAMELINK_COMPONENT DEVLIBS)
    endif()

    # Ubuntu/Linux Specific
    if(DISTRO AND DISTRO MATCHES "Ubuntu")
        find_program(LDCONFIG ldconfig)
        if(LDCONFIG)
            install(CODE "execute_process(COMMAND ${LDCONFIG})" COMPONENT LOCAL)
        endif()
    endif()
endif()

# global aiases and header Installation
add_library(srcML::LibsrcML ALIAS "libsrcml_static")
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/srcml.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} COMPONENT DEVLIBS)

# VCPKG Windows DLL support
if(WIN32 AND DEFINED VCPKG_TARGET_TRIPLET)
    file(GLOB VCPKG_SRCML_DEPENDENCIES "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin/*${CMAKE_SHARED_LIBRARY_SUFFIX}")
    install(FILES ${VCPKG_SRCML_DEPENDENCIES} DESTINATION bin COMPONENT SRCML)
endif()
