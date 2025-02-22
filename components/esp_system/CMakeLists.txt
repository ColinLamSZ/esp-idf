idf_build_get_property(target IDF_TARGET)

set(srcs "esp_err.c")

if(CONFIG_IDF_ENV_FPGA)
    list(APPEND srcs "fpga_overrides.c")
endif()

if(BOOTLOADER_BUILD)
        # "_esp_error_check_failed()" requires spi_flash module
        # Bootloader relies on some Kconfig options defined in esp_system.
        idf_component_register(SRCS "${srcs}" REQUIRES spi_flash)
else()
    list(APPEND srcs "crosscore_int.c"
            "esp_ipc.c"
            "esp_err.c"
            "freertos_hooks.c"
            "int_wdt.c"
            "panic.c"
            "esp_system.c"
            "startup.c"
            "system_time.c"
            "stack_check.c"
            "task_wdt.c"
            "ubsan.c"
            "xt_wdt.c"
            "debug_stubs.c")

    if(NOT (${target} STREQUAL "esp32c2"))
        list(APPEND srcs "debug_stubs.c")
    endif()

    if(CONFIG_ESP_SYSTEM_USE_EH_FRAME)
        list(APPEND srcs "eh_frame_parser.c")
    endif()

    idf_component_register(SRCS "${srcs}"
                        INCLUDE_DIRS include
                        PRIV_REQUIRES spi_flash esp_timer
                                    # [refactor-todo] requirements due to init code,
                                    # should be removable once using component init functions
                                    # link-time registration is used.
                                    # [refactor-todo] requires "driver" for headers:
                                    # - spi_common_internal.h
                                    pthread bootloader_support efuse driver
                        LDFRAGMENTS "linker.lf" "app.lf")
    add_subdirectory(port)

    # After system initialization, `start_app` (and its other cores variant) is called.
    # This is provided by the user or from another component. Since we can't establish
    # dependency on what we don't know, force linker to not drop the symbol regardless
    # of link line order.
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u start_app")

    if(NOT CONFIG_ESP_SYSTEM_SINGLE_CORE_MODE)
        target_link_libraries(${COMPONENT_LIB} INTERFACE "-u start_app_other_cores")
    endif()

    # Disable stack protection in files which are involved in initialization of that feature
    set_source_files_properties(
        "startup.c" "stack_check.c"
        PROPERTIES COMPILE_FLAGS
        -fno-stack-protector)

    include(${CMAKE_CURRENT_LIST_DIR}/ld/ld.cmake)
endif()

if(CONFIG_IDF_ENV_FPGA)
    # Forces the linker to include fpga stubs from this component
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u esp_common_include_fpga_overrides")
endif()

# Force linking UBSAN hooks. If UBSAN is not enabled, the hooks will ultimately be removed
# due to -ffunction-sections -Wl,--gc-sections options.
target_link_libraries(${COMPONENT_LIB} INTERFACE "-u __ubsan_include")


# [refactor-todo] requirements due to init code, should be removable
# once link-time registration of component init functions is used.
if(CONFIG_APPTRACE_ENABLE)
    idf_component_optional_requires(PRIVATE app_trace)
endif()

if(CONFIG_ESP_COREDUMP_ENABLE)
    idf_component_optional_requires(PRIVATE espcoredump)
endif()

# [refactor-todo] requirement from the panic handler,
# need to introduce panic "event" concept to remove this dependency (IDF-2194)
idf_component_optional_requires(PRIVATE esp_gdbstub)

idf_component_optional_requires(PRIVATE app_update)

if(CONFIG_PM_ENABLE)
    idf_component_optional_requires(PRIVATE pm)
endif()

if(CONFIG_VFS_SUPPORT_IO)
    idf_component_optional_requires(PRIVATE vfs)
endif()

if(CONFIG_SW_COEXIST_ENABLE OR CONFIG_EXTERNAL_COEX_ENABLE)
    idf_component_optional_requires(PRIVATE esp_wifi)
endif()

if(NOT BOOTLOADER_BUILD)
    if(CONFIG_SPIRAM)
        idf_component_optional_requires(PRIVATE esp_psram)
    endif()
endif()
