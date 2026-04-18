cmake_minimum_required(VERSION 3.16)
project(tinyrwm LANGUAGES C VERSION 0.1)

# Use C99 and strict warnings (matching Meson defaults)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

add_compile_options(-D_POSIX_C_SOURCE=200809L)
add_compile_options(-Wno-unused-parameter)  # Optional; remove if you prefer strictness

# Find required packages
find_package(PkgConfig REQUIRED)
pkg_check_modules(WAYLAND_CLIENT REQUIRED wayland-client)
pkg_check_modules(XKBCOMMON REQUIRED xkbcommon)

# Find wayland-scanner (required for protocol code generation)
find_program(WAYLAND_SCANNER wayland-scanner REQUIRED)

# Define protocol XML files (adjust paths if your protocol/ dir structure differs)
set(PROTOCOL_XML_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/protocol/river-window-management-v1.xml
    ${CMAKE_CURRENT_SOURCE_DIR}/protocol/river-xkb-bindings-v1.xml
)

# Generate C header + code for each protocol
set(GENERATED_SOURCES "")
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

    # Generate client code (implementation)
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c
        COMMAND ${WAYLAND_SCANNER} private-code ${xml}
                ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c
        DEPENDS ${xml}
        VERBATIM
    )

    list(APPEND GENERATED_SOURCES
        ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-client-protocol.h
        ${CMAKE_CURRENT_BINARY_DIR}/${proto_name}-protocol.c
    )
endforeach()

# Main executable
add_executable(tinyrwm
    tinyrwm.c
    ${GENERATED_SOURCES}
)

target_include_directories(tinyrwm PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

target_link_libraries(tinyrwm PRIVATE
    ${WAYLAND_CLIENT_LIBRARIES}
    ${XKBCOMMON_LIBRARIES}
)

# Install the binary
install(TARGETS tinyrwm RUNTIME DESTINATION bin)

