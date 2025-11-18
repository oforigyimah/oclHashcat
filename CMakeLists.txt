cmake_minimum_required(VERSION 3.10)
project(hashcat C)

# Detect Operating System
if(NOT (UNIX OR CYGWIN))
    message(FATAL_ERROR "Your Operating System is not supported by hashcat")
endif()

# Options
option(DEBUG "Enable debug build" OFF)
option(BUILD_API "Build API version" OFF)

# Installation paths (Linux only)
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Install prefix" FORCE)
endif()

set(INSTALL_FOLDER "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Binary installation folder")
set(SHARED_FOLDER "${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME}" CACHE PATH "Shared files folder")
set(DOCUMENT_FOLDER "${CMAKE_INSTALL_PREFIX}/share/doc/${PROJECT_NAME}" CACHE PATH "Documentation folder")

# Dependencies
set(OPENCL_HEADERS_KHRONOS "deps/OpenCL-Headers")

# Version information
execute_process(
        COMMAND git describe --tags --dirty=+
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE VERSION_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
)

if(NOT VERSION_TAG)
    set(VERSION_TAG "unknown")
endif()

string(TIMESTAMP COMPTIME "%s")

# Compiler flags
set(CMAKE_C_STANDARD 99)
add_compile_options(-pipe -W -Wall)
include_directories(include OpenCL ${OPENCL_HEADERS_KHRONOS})

if(DEBUG)
    add_definitions(-DDEBUG)
    set(CMAKE_BUILD_TYPE Debug)
    add_compile_options(-g -O0)
    if(DEBUG EQUAL 2)
        add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
        add_link_options(-fsanitize=address)
    endif()
else()
    set(CMAKE_BUILD_TYPE Release)
    set(CMAKE_C_FLAGS_RELEASE "-O2")
endif()

# Platform-specific settings
if(APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9")
    set(BINARY_NAME "${PROJECT_NAME}.app")
    add_definitions(-D_POSIX -DOSX)
    set(PLATFORM_LIBS pthread)
elseif(UNIX AND NOT CYGWIN)
    set(BINARY_NAME "${PROJECT_NAME}")
    add_definitions(-D_POSIX -DLINUX -DHAVE_HWMON)
    if(NOT DEBUG)
#        add_link_options(-s)
    endif()
    set(PLATFORM_LIBS pthread dl)
elseif(CYGWIN)
    set(BINARY_NAME "${PROJECT_NAME}")
    add_definitions(-D_POSIX)
    set(PLATFORM_LIBS pthread)
endif()

# Source files
set(NATIVE_SOURCES
        src/ext_OpenCL.c
        src/shared.c
        src/rp_kernel_on_cpu.c
)

if(UNIX AND NOT APPLE AND NOT CYGWIN)
    list(APPEND NATIVE_SOURCES
            src/ext_ADL.c
            src/ext_nvapi.c
            src/ext_nvml.c
            src/ext_xnvctrl.c
    )
endif()

# Main executable
add_executable(${PROJECT_NAME} src/hashcat.c ${NATIVE_SOURCES})

if(BUILD_API)
    target_sources(${PROJECT_NAME} PRIVATE src/hashcat_api.c)
    target_compile_definitions(${PROJECT_NAME} PRIVATE API)
    target_link_libraries(${PROJECT_NAME} m)
endif()

target_compile_definitions(${PROJECT_NAME} PRIVATE
        COMPTIME=${COMPTIME}
        VERSION_TAG="${VERSION_TAG}"
        INSTALL_FOLDER="${INSTALL_FOLDER}"
        SHARED_FOLDER="${SHARED_FOLDER}"
        DOCUMENT_FOLDER="${DOCUMENT_FOLDER}"
)

target_link_libraries(${PROJECT_NAME} ${PLATFORM_LIBS})
set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME ${BINARY_NAME})

# Installation rules
if(UNIX AND NOT APPLE)
    install(TARGETS ${PROJECT_NAME} DESTINATION ${INSTALL_FOLDER})
    install(FILES example.dict example0.hash example400.hash example500.hash
            DESTINATION ${DOCUMENT_FOLDER})
    install(PROGRAMS example0.sh example400.sh example500.sh
            DESTINATION ${DOCUMENT_FOLDER})
    install(FILES hashcat.hcstat hashcat.hctune DESTINATION ${SHARED_FOLDER})
    install(DIRECTORY docs/ DESTINATION ${DOCUMENT_FOLDER}/docs)
    install(DIRECTORY charsets/ DESTINATION ${SHARED_FOLDER}/charsets)
    install(DIRECTORY masks/ DESTINATION ${SHARED_FOLDER}/masks)
    install(DIRECTORY OpenCL/ DESTINATION ${SHARED_FOLDER}/OpenCL)
    install(DIRECTORY rules/ DESTINATION ${SHARED_FOLDER}/rules)
    install(DIRECTORY extra/tab_completion/
            DESTINATION ${DOCUMENT_FOLDER}/extra/tab_completion)
endif()

# Copy runtime resources into the build directory
message(STATUS "Copying resources to build dir: OpenCL/, hashcat.hctune, hashcat.hcstat, nv_instaruc.txt")

file(COPY ${CMAKE_SOURCE_DIR}/OpenCL DESTINATION ${CMAKE_BINARY_DIR})
file(COPY
     ${CMAKE_SOURCE_DIR}/hashcat.hctune
     ${CMAKE_SOURCE_DIR}/hashcat.hcstat
     ${CMAKE_SOURCE_DIR}/nv_instruc.txt
     DESTINATION ${CMAKE_BINARY_DIR}
)
