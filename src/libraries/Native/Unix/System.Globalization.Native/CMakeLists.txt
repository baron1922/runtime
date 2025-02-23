project(System.Globalization.Native C)

if(CLR_CMAKE_TARGET_UNIX)
    add_compile_options(-Wno-switch-enum)
    add_compile_options(-Wno-covered-switch-default)

    # Workaround for warnings produced by ICU headers
    add_compile_options(-Wno-reserved-id-macro)
    add_compile_options(-Wno-documentation)
    add_compile_options(-Wno-documentation-unknown-command)

    # Workaround for https://unicode-org.atlassian.net/browse/ICU-20601
    add_compile_options(-Wno-extra-semi-stmt)
    add_compile_options(-Wno-unknown-warning-option)

    if (NOT CLR_CMAKE_TARGET_ANDROID)
        if (CLR_CMAKE_TARGET_OSX OR CLR_CMAKE_TARGET_MACCATALYST)
            execute_process(COMMAND  brew --prefix OUTPUT_VARIABLE brew_prefix OUTPUT_STRIP_TRAILING_WHITESPACE)
            set(ICU_HOMEBREW_INC_PATH "${brew_prefix}/opt/icu4c/include")
        endif()

        find_path(UCURR_H "unicode/ucurr.h" PATHS ${ICU_HOMEBREW_INC_PATH})
        if(UCURR_H STREQUAL UCURR_H-NOTFOUND)
            message(FATAL_ERROR "Cannot find ucurr.h, try installing libicu-dev (or the appropriate package for your platform)")
            return()
        endif()

        if(CLR_CMAKE_TARGET_OSX OR CLR_CMAKE_TARGET_MACCATALYST)
            add_definitions(-DOSX_ICU_LIBRARY_PATH="/usr/lib/libicucore.dylib")
            add_definitions(-DU_DISABLE_RENAMING)
        else()
            find_library(ICUUC icuuc)
            if(ICUUC STREQUAL ICUUC-NOTFOUND)
                message(FATAL_ERROR "Cannot find libicuuc, try installing libicu-dev (or the appropriate package for your platform)")
                return()
            endif()

            find_library(ICUI18N icui18n)
            if(ICUI18N STREQUAL ICUI18N-NOTFOUND)
                message(FATAL_ERROR "Cannot find libicui18n, try installing libicu-dev (or the appropriate package for your platform)")
                return()
            endif()
        endif()

        include_directories(${UCURR_H})
    endif()
endif()

if ((CMAKE_CXX_COMPILER_ID STREQUAL "GNU") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 10.0))
    add_compile_options(-Wno-error=stringop-truncation)
endif()

include(configure.cmake)

set(NATIVEGLOBALIZATION_SOURCES
    pal_calendarData.c
    pal_casing.c
    pal_collation.c
    pal_idna.c
    pal_locale.c
    pal_localeNumberData.c
    pal_localeStringData.c
    pal_normalization.c
    pal_icushim.c
)

# time zone names are filtered out of icu data for the browser and associated functionality is disabled
if (NOT CLR_CMAKE_TARGET_BROWSER)
    set(NATIVEGLOBALIZATION_SOURCES ${NATIVEGLOBALIZATION_SOURCES} pal_timeZoneInfo.c)
endif()

if (NOT GEN_SHARED_LIB AND NOT CLR_CMAKE_TARGET_MACCATALYST AND NOT CLR_CMAKE_TARGET_IOS AND NOT CLR_CMAKE_TARGET_TVOS AND NOT CLR_CMAKE_TARGET_ANDROID AND NOT CLR_CMAKE_TARGET_BROWSER)
    set(NATIVEGLOBALIZATION_SOURCES ${NATIVEGLOBALIZATION_SOURCES} entrypoints.c)
endif()

if (MSVC)
    set_source_files_properties(${NATIVEGLOBALIZATION_SOURCES} PROPERTIES LANGUAGE CXX)
endif()

include_directories("../Common")

if (GEN_SHARED_LIB)
    add_library(System.Globalization.Native
        SHARED
        ${NATIVEGLOBALIZATION_SOURCES}
        ${VERSION_FILE_PATH}
    )

    target_link_libraries(System.Globalization.Native
        dl
    )

    install_with_stripped_symbols (System.Globalization.Native PROGRAMS .)
endif()


add_library(System.Globalization.Native-Static
    STATIC
    ${NATIVEGLOBALIZATION_SOURCES}
)

if(CLR_CMAKE_TARGET_UNIX)
    set_target_properties(System.Globalization.Native-Static PROPERTIES OUTPUT_NAME System.Globalization.Native  CLEAN_DIRECT_OUTPUT 1)
endif()

install (TARGETS System.Globalization.Native-Static DESTINATION ${STATIC_LIB_DESTINATION} COMPONENT libs)

if(NOT CLR_CMAKE_TARGET_OSX AND NOT CLR_CMAKE_TARGET_MACCATALYST AND NOT CLR_CMAKE_TARGET_IOS AND NOT CLR_CMAKE_TARGET_TVOS AND NOT CLR_CMAKE_TARGET_ANDROID)
    if (GEN_SHARED_LIB)
        add_custom_command(TARGET System.Globalization.Native POST_BUILD
            COMMENT "Verifying System.Globalization.Native.so dependencies"
            COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/../verify-so.sh
                $<TARGET_FILE:System.Globalization.Native>
                "Verification failed. System.Globalization.Native.so has undefined dependencies. These are likely ICU APIs that need to be added to icushim.h."
            VERBATIM
        )
    endif()
endif()
