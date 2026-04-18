cmake_minimum_required(VERSION 3.16)
project(waldmeister LANGUAGES C CXX VERSION 0.1)

set(CMAKE_CXX_STANDARD 26)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(-D_POSIX_C_SOURCE=200809L)
add_compile_options(-Wno-unused-parameter)

find_package(PkgConfig REQUIRED)
pkg_check_modules(WAYLAND_CLIENT REQUIRED wayland-client)
pkg_check_modules(XKBCOMMON REQUIRED xkbcommon)

find_program(WAYLAND_SCANNER wayland-scanner REQUIRED)

# Protocol XML files
set(PROTOCOL_XML_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/protocol/river-window-management-v1.xml
    ${CMAKE_CURRENT_SOURCE_DIR}/protocol/river-xkb-bindings-v1.xml
)

set(GENERATED_SOURCES "")
set(GENERATED_HEADERS "")

foreach(xml ${PROTOCOL_XML_FILES})
    get_filename_component(proto_name ${xml} NAME_WE)

    # Generate client header
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-client-protocol.h
        COMMAND ${WAYLAND_SCANNER} client-header ${xml}
                ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-client-protocol.h
        DEPENDS ${xml}
        VERBATIM
    )

    # Generate private code (implementation)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c
        COMMAND ${WAYLAND_SCANNER} private-code ${xml}
                ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c
        DEPENDS ${xml}
        VERBATIM
    )

    list(APPEND GENERATED_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-client-protocol.h)
    list(APPEND GENERATED_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c)
endforeach()

# Main executable
add_executable(waldmeister
    waldmeister.cxx
    ${GENERATED_SOURCES}
)

target_include_directories(waldmeister PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

# Crucial: Compile generated .c files as C, not as C++
set_source_files_properties(${GENERATED_SOURCES} PROPERTIES LANGUAGE C)

target_link_libraries(waldmeister PRIVATE
    ${WAYLAND_CLIENT_LIBRARIES}
    ${XKBCOMMON_LIBRARIES}
)

install(TARGETS waldmeister RUNTIME DESTINATION bin)
