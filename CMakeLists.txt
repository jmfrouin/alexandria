cmake_minimum_required(VERSION 3.26)

# Detect, which version of FileDialog to use for
# We always assume that GTK is used on platforms other than Windows and macOS,
# as there is no other implementation available now
if( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
    set( wxIS_MAC on )
elseif( CMAKE_SYSTEM_NAME MATCHES "Windows" )
    set ( wxIS_WIN on )
else()
    set ( wxIS_GTK on)
endif()

find_package(wxWidgets COMPONENTS core base OPTIONAL_COMPONENTS net)

string( TIMESTAMP __TDATE__ "%Y%m%d" )
if( AUDACITY_BUILD_LEVEL EQUAL 0 )
    set( AUDACITY_SUFFIX "-alpha-${__TDATE__}" )
elseif(  AUDACITY_BUILD_LEVEL EQUAL 1 )
    set( AUDACITY_SUFFIX "-beta-${__TDATE__}" )
else()
    set( AUDACITY_SUFFIX "" )
endif()

if( APPLE )
    # CMAKE_HOST_SYSTEM_PROCESSOR is empty before the project() command
    execute_process(
            COMMAND uname -m
            OUTPUT_VARIABLE HOST_SYSTEM_PROCESSOR
            OUTPUT_STRIP_TRAILING_WHITESPACE
    )

    set( MACOS_ARCHITECTURE "${HOST_SYSTEM_PROCESSOR}" CACHE STRING "Target architecture for macOS builds. One of x86_64/arm64" )

    if( MACOS_ARCHITECTURE STREQUAL "x86_64" )
        set( MIN_MACOS_VERSION 10.9 )
        set( TARGET_MACOS_VERSION 10.13 )
        set( CMAKE_OSX_ARCHITECTURES x86_64 CACHE INTERNAL "" )
    elseif( MACOS_ARCHITECTURE STREQUAL "arm64" )
        set( MIN_MACOS_VERSION 10.13 )
        set( TARGET_MACOS_VERSION 10.13 )
        set( CMAKE_OSX_ARCHITECTURES arm64 CACHE INTERNAL "" )
    else()
        message( FATAL_ERROR "MACOS_ARCHITECTURE must be x86_64 or arm64")
    endif()
    # Generate schema files
    set( CMAKE_XCODE_GENERATE_SCHEME ON )

    # Define the OSX compatibility parameters
    set( CMAKE_OSX_DEPLOYMENT_TARGET ${MIN_MACOS_VERSION} CACHE INTERNAL "" )
    set( CMAKE_OSX_SYSROOT macosx CACHE INTERNAL "" )

    # This prevents a link error when building with the 10.9 or older SDKs
    set( CMAKE_XCODE_ATTRIBUTE_CLANG_LINK_OBJC_RUNTIME OFF )

    # Shouldn't cmake do this???
    string( APPEND CMAKE_CXX_FLAGS " -stdlib=libc++" )
endif()

ADD_DEFINITIONS(-Wall -MD -fPIC -pg -g -DLEAK_DETECTOR -DwxUSE_UNICODE -DDEBUG -DVERBOSE)

# Generate config file
#if( CMAKE_SYSTEM_NAME MATCHES "Windows" )
#    configure_file( src/1poster_config.h.in src/private/configwin.h )
#elseif( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
#    set( HAVE_VISIBILITY 1 )
#    configure_file( src/1poster_config.h.in src/private/configmac.h )
#else()
#    set( HAVE_VISIBILITY 1 )
#    configure_file( src/1poster_config.h.in src/private/configunix.h )
#endif()

# Set the required C++ standard
set( CMAKE_CXX_STANDARD 17 )
set( CMAKE_CXX_STANDARD_REQUIRED ON )

project(alexandria)

# XCode 14 no longer allows building unsigned binaries.
# So for the XCode 14 `-` is passed as the code sign identity, which stands for
# local signing. `--deep` is passed, because 3d party libraries are copied unsigned.
# XCODE_VERSION is defined only after the project() command

if( APPLE )
    if (XCODE_VERSION VERSION_LESS 14)
        set( CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "" CACHE INTERNAL "" )
    else()
        set( CMAKE_XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "-" CACHE INTERNAL "" )
        set( CMAKE_XCODE_ATTRIBUTE_OTHER_CODE_SIGN_FLAGS "--deep -o linker-signed --timestamp" CACHE INTERNAL "")
    endif()
endif()

message( STATUS "Build Info:" )
message( STATUS "  Host System: ${CMAKE_HOST_SYSTEM}" )
message( STATUS "  Host System Name: ${CMAKE_HOST_SYSTEM_NAME}" )
message( STATUS "  Host System Processor: ${CMAKE_HOST_SYSTEM_PROCESSOR}" )
message( STATUS "  Host System Version: ${CMAKE_HOST_SYSTEM_VERSION}" )

if( IS_64BIT )
    message( STATUS "  Host System Architecture: 64-bit" )
else()
    message( STATUS "  Host System Architecture: 32-bit" )
endif()

message( STATUS )
message( STATUS "  Compiler: ${CMAKE_CXX_COMPILER}" )
message( STATUS "  Compiler Version: ${CMAKE_CXX_COMPILER_VERSION}" )
message( STATUS "  Compiler Standard: ${CMAKE_CXX_STANDARD}" )
message( STATUS "  Compiler Standard Required: ${CMAKE_CXX_STANDARD_REQUIRED}" )
message( STATUS "  Compiler Extensions: ${CMAKE_CXX_EXTENSIONS}" )
message( STATUS )


if( CMAKE_GENERATOR MATCHES "Visual Studio" )
    message( STATUS "  MSVC Version: ${MSVC_VERSION}" )
    message( STATUS "  MSVC Toolset: ${MSVC_TOOLSET_VERSION}" )
    message( STATUS )
elseif( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
    if( CMAKE_GENERATOR MATCHES "Xcode" )
        message( STATUS "  Xcode Version: ${XCODE_VERSION}" )
    endif()
    message( STATUS "  MacOS SDK: ${CMAKE_OSX_SYSROOT}" )
    message( STATUS )

    if(${_OPT}has_crashreports)
        set(CMAKE_XCODE_ATTRIBUTE_DEBUG_INFORMATION_FORMAT "dwarf-with-dsym")
    endif()

endif()

# Try to get the current commit information
set( GIT_COMMIT_SHORT "unknown" )
set( GIT_COMMIT_LONG "unknown" )
set( GIT_COMMIT_TIME "unknown" )
find_package( Git QUIET )
#if( GIT_FOUND )
#    execute_process(
#            COMMAND
#            ${GIT_EXECUTABLE} show -s "--format=%h;%H;%cd"
#            WORKING_DIRECTORY
#            ${topdir}
#            OUTPUT_VARIABLE
#            git_output
#            OUTPUT_STRIP_TRAILING_WHITESPACE
#            ERROR_QUIET
#    )
#
#    if( git_output )
#        list( GET git_output 0 GIT_COMMIT_SHORT )
#        list( GET git_output 1 GIT_COMMIT_LONG )
#        list( GET git_output 2 GIT_COMMIT_TIME )
#
#        file(WRITE
#                "${CMAKE_BINARY_DIR}/Variables.cmake"
#                "set( AUDACITY_BUILD_LEVEL ${AUDACITY_BUILD_LEVEL} )\nset( AUDACITY_REV_LONG \"${GIT_COMMIT_LONG}\" )\nset( AUDACITY_REV_TIME \"${GIT_COMMIT_TIME}\" )\n"
#        )
#    endif()
#endif()
message( STATUS "  Current Commit: ${GIT_COMMIT_SHORT}" )
message( STATUS )

# Adjust them for the Mac
if( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
    set( _APPDIR "1poster.app/Contents" )
    set( _DEST "${_DESTDIR}/${_APPDIR}" )
    set( _EXEDIR "${_APPDIR}/MacOS" )
    set( _MODDIR "${_APPDIR}/modules" )
    set( _PKGLIB "${_APPDIR}/Frameworks" )

    set( CMAKE_MACOSX_RPATH OFF )
    set( CMAKE_INSTALL_RPATH "@loader_path/../Frameworks" )
    set( CMAKE_BUILD_WITH_INSTALL_NAME_DIR ON)
    set( CMAKE_INSTALL_NAME_DIR "@loader_path/../Frameworks" )
    set( CMAKE_BUILD_WITH_INSTALL_RPATH ON )
elseif( WIN32 )
    set( _EXEDIR "" )
    set( _PKGLIB "" )
    set( _MODDIR "modules" )
endif()


include(${wxWidgets_USE_FILE})

#add_subdirectory("src")
#add_subdirectory("src/audacity")

add_executable(alexandria main.cpp)

target_link_libraries(alexandria ${wxWidgets_LIBRARIES})