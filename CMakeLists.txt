cmake_minimum_required(VERSION 3.21.0 FATAL_ERROR)

# Make sure that custom modules are found
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# Enables the Visibility Property on all target types
cmake_policy(SET CMP0063 NEW)
# Enables the MSVC_RUNTIME_LIBRARY property on targets
cmake_policy(SET CMP0091 NEW)

# Define the Project Name and Description
project(
    dbo
    VERSION 0.0.1
    LANGUAGES CXX
    DESCRIPTION "A modern C++ ORM"
)
# ######################################################################################################################
# Include guards: Prevent building in the source directory
# ######################################################################################################################
if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there.\n")
endif()
#

set(CMAKE_PROJECT_VERSION ${CMAKE_PROJECT_VERSION_MAJOR}.${CMAKE_PROJECT_VERSION_MINOR}.${CMAKE_PROJECT_VERSION_PATCH})
option(${PROJECT_NAME}_USE_ALT_NAMES "Use alternative names for the project, such as naming the include directory all lowercase." ON)

if(${PROJECT_NAME}_USE_ALT_NAMES)
    string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWERCASE)
    string(TOUPPER ${PROJECT_NAME} PROJECT_NAME_UPPERCASE)
else()
    set(PROJECT_NAME_LOWERCASE ${PROJECT_NAME})
    set(PROJECT_NAME_UPPERCASE ${PROJECT_NAME})
endif()

# ######################################################################################################################
# The location where the project's version header will be placed should match the project's regular header paths. This
# allows users to install and find the library via `find_package()`.
# ######################################################################################################################
set(VERSION_HEADER_LOCATION "${PROJECT_NAME_LOWERCASE}/version.hpp")
configure_file(${CMAKE_CURRENT_LIST_DIR}/cmake/version.hpp.in ${VERSION_HEADER_LOCATION} @ONLY)


# ######################################################################################################################
# Define Options
# ######################################################################################################################
include(Utils)
include(StandardSettings)
include(StaticAnalyzers)
if (UNIX)
    # This will allow to use same _DEBUG macro available in both Linux as well as Windows - MSCV environment. 
    # Easy to put Debug specific code.
    add_compile_options("$<$<CONFIG:DEBUG>:-D_DEBUG>")    
endif (UNIX)
message(STATUS "Started CMake for ${PROJECT_NAME} v${PROJECT_VERSION}...\n")

# ######################################################################################################################
# Project Settings
# ######################################################################################################################

# Only do these if this is the main project, and not if it is included through add_subdirectory
if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
    ##############################################################################
    #                         C M A K E    C O N T R O L                          #
    ###############################################################################
    # Put the libraries and binaries that get built into directories at the top of the build tree rather than in
    # hard-to-find leaf directories. This simplifies manual testing and the use of the build tree rather than installed
    # Boost libraries.
    set(MAINFOLDER ${PROJECT_SOURCE_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
    # Windows DLLs are "runtime" for CMake. Output them to "bin" like the Visual Studio projects do.
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

    # Must use GNUInstallDirs to install libraries into correct locations on all platforms.
    include(GNUInstallDirs)

    # Let's nicely support folders in IDE's
    set_property(GLOBAL PROPERTY USE_FOLDERS ON)

    # Generate the compile_commands.json file
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
    if(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/compile_commands.json")
        execute_process(
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/compile_commands.json
                    ${CMAKE_CURRENT_SOURCE_DIR}/compile_commands.json
        )
    endif()

    set(CMAKE_DEBUG_POSTFIX d)
    # Always use '-fPIC'/'-fPIE' option.
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    #
    set(CMAKE_CONFIGURATION_TYPES "Debug" CACHE STRING "" FORCE)

    # Set the Relative Path Configurations
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

    # Testing only available if this is the main app Note this needs to be done in the main CMakeLists since it calls
    # enable_testing, which must be in the main CMakeLists.
    include(CTest)
    
    # ##################################################################################################################
    # Define CMake Module Imports
    # ##################################################################################################################
    include(CMakePackageConfigHelpers)
    include(InstallRequiredSystemLibraries)
    include(GenerateExportHeader)
    include(CMakeParseArguments)
    
    # Setup C/C++ compiler options Compiler settings - special settings for known compilers
    include(CheckCXXCompilerFlag)
    include(CMakePackageConfigHelpers)
    include(CheckTypeSize)
    include(CMakePrintHelpers)
    include(CompilerWarnings)
    
    set(CMAKE_CXX_STANDARD 20)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)

    #
    option(CPM_USE_LOCAL_PACKAGES "Always try to use `find_package` to get dependencies" TRUE)
 
endif()

# ######################################################################################################################
# Add dependencies
# ######################################################################################################################

include(CPM)
CPMAddPackage("gh:TheLartians/Format.cmake@1.7.3")
# PackageProject.cmake will be used to make our target installable
CPMAddPackage("gh:TheLartians/PackageProject.cmake@1.8.0")

# Collect the built libraries and include directories
# A semicolon separated list of the project's dependencies
set(${PROJECT_NAME}_DEPENDENCIES "")

CPMAddPackage(
  NAME fmt
  GIT_TAG 8.1.1
  GITHUB_REPOSITORY fmtlib/fmt
  OPTIONS "FMT_INSTALL YES" # create an installable target
)
list(APPEND ${PROJECT_NAME}_DEPENDENCIES "fmt 8.1.1")

CPMAddPackage(
    NAME spdlog
    GITHUB_REPOSITORY gabime/spdlog
    VERSION 1.9.2
    OPTIONS "SPDLOG_INSTALL YES" # create an installable target
)
list(APPEND ${PROJECT_NAME}_DEPENDENCIES "spdlog 1.9.2")

# ######################################################################################################################
# Source tree
# ######################################################################################################################
add_subdirectory(extern)
add_subdirectory(src)

# ######################################################################################################################
# Unit testing setup
# ######################################################################################################################
if(${PROJECT_NAME_UPPERCASE}_ENABLE_TESTING OR ENABLE_TESTING)
    message(STATUS "Build unit tests for the project. Tests should always be found in the test folder\n")
    add_subdirectory(test)
endif()

# ######################################################################################################################
# PACKING
# ######################################################################################################################
packageProject(
    # the name of the target to export
    NAME ${PROJECT_NAME}
    # the version of the target to export
    VERSION ${PROJECT_VERSION}
    # (optional) install your library with a namespace (Note: do NOT add extra '::')
    NAMESPACE ${PROJECT_NAME}
    # a temporary directory to create the config files
    BINARY_DIR ${PROJECT_BINARY_DIR}
    # location of the target's public headers
    # A.K.A Install the `include` directory
    INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include
    # Install the `include` directory.
    # NOTE: should match the target's INSTALL_INTERFACE include directory
    INCLUDE_DESTINATION include
    # (optional) create a header containing the version info
    # Note: that the path to headers should be lowercase
    VERSION_HEADER "${VERSION_HEADER_LOCATION}"
    
    # (optional) create a export header using GenerateExportHeader module
    #EXPORT_HEADER "${PROJECT_NAME}/export.hpp"

    # (optional) define the project's version compatibility, defaults to `AnyNewerVersion`
    # supported values: `AnyNewerVersion|SameMajorVersion|SameMinorVersion|ExactVersion`
    COMPATIBILITY SameMajorVersion
    # semicolon separated list of the project's dependencies
    DEPENDENCIES "${${PROJECT_NAME}_DEPENDENCIES}"
    # (optional) option to disable the versioning of install destinations
    DISABLE_VERSION_SUFFIX YES
    # (optional) option to ignore target architecture for package resolution
    # defaults to YES for header only (i.e. INTERFACE) libraries
    ARCH_INDEPENDENT YES
)

# ######################################################################################################################
# Add uninstall target
# ######################################################################################################################
CONFIGURE_FILE(
   "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
   "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
   IMMEDIATE @ONLY)

ADD_CUSTOM_TARGET(uninstall
    "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
)

# ######################################################################################################################
# INFO
# ######################################################################################################################
if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
    message(STATUS "CMake ${CMAKE_VERSION} successfully configured ${PROJECT_NAME} using ${CMAKE_GENERATOR} generator")
    message(STATUS "${PROJECT_NAME} package version: ${CMAKE_PROJECT_VERSION}")
    message(STATUS "${PROJECT_NAME} package dependencies: ${${PROJECT_NAME}_DEPENDENCIES}")
    if(BUILD_SHARED_LIBS)
        message(STATUS "Building dynamic libraries")
    else()
        message(STATUS "Building static libraries")
    endif()
    message(STATUS "[cmake] Installation target path: ${CMAKE_INSTALL_PREFIX}")
    if(CMAKE_TOOLCHAIN_FILE)
        message(STATUS "[cmake] Use toolchain file:		${CMAKE_TOOLCHAIN_FILE}")
    endif()
    message(STATUS "[cmake] Build for OS type:      ${CMAKE_SYSTEM_NAME}")
    message(STATUS "[cmake] Build for OS version:   ${CMAKE_SYSTEM_VERSION}")
    message(STATUS "[cmake] Build for CPU type:     ${CMAKE_SYSTEM_PROCESSOR}")
    message(STATUS "[cmake] Build type:             ${CMAKE_BUILD_TYPE}")
    string(TOUPPER "${CMAKE_BUILD_TYPE}" BUILD_TYPE)
    message(STATUS "[cmake] Build with cxx flags:   ${CMAKE_CXX_FLAGS_${BUILD_TYPE}} ${CMAKE_CXX_FLAGS}")
    message(STATUS "[cmake] Build with c flags:     ${CMAKE_C_FLAGS_${BUILD_TYPE}} ${CMAKE_C_FLAGS}")
endif()
