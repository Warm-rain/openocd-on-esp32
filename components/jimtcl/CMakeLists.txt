idf_component_register(REQUIRES newlib pthread vfs
                       INCLUDE_DIRS jimtcl)

idf_build_get_property(c_compile_options C_COMPILE_OPTIONS GENERATOR_EXPRESSION)
idf_build_get_property(compile_options COMPILE_OPTIONS GENERATOR_EXPRESSION)
idf_build_get_property(compile_definitions COMPILE_DEFINITIONS GENERATOR_EXPRESSION)
set(interface_include_directories $<TARGET_PROPERTY:idf::jimtcl,INTERFACE_INCLUDE_DIRECTORIES>)

set (opts "${CMAKE_C_FLAGS}")
string(APPEND opts " -I${COMPONENT_DIR}/../platform_include")
string(APPEND opts " -I$<JOIN:${interface_include_directories}, -I>")
string(APPEND opts " $<JOIN:${compile_options}, >")
string(APPEND opts " $<JOIN:${c_compile_options}, >")
if(${IDF_VERSION_MAJOR} EQUAL 5 AND ${IDF_VERSION_MINOR} EQUAL 0)
    string(APPEND opts " $<JOIN:${compile_definitions}, >")
else()
    string(APPEND opts " -D$<JOIN:${compile_definitions}, -D>")
endif()
string(APPEND opts " -Wno-implicit-fallthrough -Wno-missing-field-initializers")
string(APPEND opts " -Wno-error=maybe-uninitialized")
string(APPEND opts " -DHAVE_USLEEP=1")
if(CMAKE_HOST_WIN32)
    string(REGEX REPLACE "@\"([^\"]+)\"" "@\\1" opts "${opts}")
endif()

if(NOT CMAKE_BUILD_EARLY_EXPANSION)
    include(ExternalProject)
    get_filename_component(gcc_name "${CMAKE_C_COMPILER}" NAME_WE)
    string(REPLACE "-gcc" "" host_prefix "${gcc_name}")

    set(jimtcl_configure_args
        --disable-install-jim
        --with-ext=json
        --minimal
        --disable-ssl
        --host=${host_prefix}
        "CC=${CMAKE_C_COMPILER}"
        "CFLAGS=${opts}"
    )
    set(jimtcl_configure_command ./configure ${jimtcl_configure_args})
    set(jimtcl_build_command make libjim.a initjimsh.o)

    if(CMAKE_HOST_WIN32)
        find_program(JIMTCL_SH_EXECUTABLE sh
            HINTS "C:/Program Files/Git/bin" "C:/Program Files/Git/usr/bin")
        find_program(JIMTCL_MAKE_EXECUTABLE make
            HINTS "$ENV{LOCALAPPDATA}/Microsoft/WinGet/Packages/ezwinports.make_Microsoft.Winget.Source_8wekyb3d8bbwe/bin")

        if(NOT JIMTCL_SH_EXECUTABLE)
            message(FATAL_ERROR "Git sh.exe is required to configure JimTcl on Windows")
        endif()
        if(NOT JIMTCL_MAKE_EXECUTABLE)
            message(FATAL_ERROR "GNU Make is required to build JimTcl on Windows")
        endif()

        set(jimtcl_configure_command ${JIMTCL_SH_EXECUTABLE} ./configure ${jimtcl_configure_args})
        set(jimtcl_build_command ${JIMTCL_MAKE_EXECUTABLE} libjim.a initjimsh.o)
    endif()

    ExternalProject_Add(jimtcl_build
        PREFIX ${COMPONENT_DIR}
        SOURCE_DIR ${COMPONENT_DIR}/jimtcl
        CONFIGURE_COMMAND ${jimtcl_configure_command}
        BUILD_IN_SOURCE 1
        # These two options are set so that Ninja immediately outputs
        # the subproject build to the terminal. Otherwise it looks like the
        # build process "hangs" for too long until jimtcl build is complete.
        USES_TERMINAL_CONFIGURE TRUE
        USES_TERMINAL_BUILD TRUE
        BUILD_COMMAND ${jimtcl_build_command}
        INSTALL_COMMAND ""
        BUILD_BYPRODUCTS ${COMPONENT_DIR}/jimtcl/libjim.a ${COMPONENT_DIR}/jimtcl/initjimsh.o
        )

    add_library(jim_imp STATIC IMPORTED)
    add_dependencies(jim_imp jimtcl_build)

    set_target_properties(jim_imp PROPERTIES IMPORTED_LOCATION
        ${COMPONENT_DIR}/jimtcl/libjim.a)
    set_target_properties(jim_imp PROPERTIES INTERFACE_INCLUDE_DIRECTORIES
        ${COMPONENT_DIR}/jimtcl)
    set_target_properties(jim_imp PROPERTIES IMPORTED_LINK_INTERFACE_LIBRARIES
        "idf::newlib;idf::compat")
    set_target_properties(jim_imp PROPERTIES ADDITIONAL_CLEAN_FILES
        ${COMPONENT_DIR}/jimtcl/libjim.a)

    add_library(initjimsh STATIC IMPORTED)
    add_dependencies(initjimsh jimtcl_build)
    set_target_properties(initjimsh PROPERTIES IMPORTED_LOCATION
        ${COMPONENT_DIR}/jimtcl/initjimsh.o)
    set_target_properties(initjimsh PROPERTIES ADDITIONAL_CLEAN_FILES
        ${COMPONENT_DIR}/jimtcl/initjimsh.o)

    target_link_libraries(jim_imp INTERFACE idf::newlib idf::compat)

    target_link_libraries(${COMPONENT_TARGET} INTERFACE jim_imp initjimsh)
endif()

add_custom_target(debug_jimtcl_opts COMMAND ${CMAKE_COMMAND} -E echo "${opts}")
