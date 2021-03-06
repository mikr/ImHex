cmake_minimum_required(VERSION 3.16)
project(HexEditor)

SET(CMAKE_FIND_PACKAGE_SORT_ORDER NATURAL)
SET(CMAKE_FIND_PACKAGE_SORT_DIRECTION DEC)

set(CMAKE_CXX_STANDARD 20)


find_package(PkgConfig REQUIRED)
pkg_search_module(GLFW REQUIRED glfw3)
pkg_search_module(GLM REQUIRED glm)
pkg_search_module(CRYPTO REQUIRED libcrypto)
pkg_search_module(CAPSTONE REQUIRED capstone)
if (APPLE)
    pkg_search_module(OPENSSL REQUIRED OpenSSL)
endif (APPLE)

if (NOT LLVM_DEMANGLE)
    include(CheckCXXSymbolExists)
    check_cxx_symbol_exists(abi::__cxa_demangle "cxxabi.h" HAVE_CXXABI)
endif (NOT LLVM_DEMANGLE)

find_package(OpenGL REQUIRED)
if (LLVM_DEMANGLE OR NOT HAVE_CXXABI)
    find_package(LLVM REQUIRED CONFIG)
endif (LLVM_DEMANGLE OR NOT HAVE_CXXABI)
find_package(nlohmann_json REQUIRED)
find_package(Python COMPONENTS Interpreter Development)
if (APPLE)
    find_package(Python3 REQUIRED)
    find_package(glfw3 REQUIRED)
    find_package(OpenSSL REQUIRED)
    find_library(CAPSTONE_LIBRARY capstone)
    find_library(MAGIC_LIBRARY magic)
endif (APPLE)


if(Python_VERSION LESS 3)
    message(STATUS ${PYTHON_VERSION_MAJOR_MINOR})
    message(FATAL_ERROR "No valid version of Python 3 was found.")
endif()


if (LLVM_FOUND)
    llvm_map_components_to_libnames(_llvm_libs demangle)
    llvm_expand_dependencies(llvm_libs ${_llvm_libs})
    include_directories(include ${LLVM_INCLUDE_DIRS})
endif (LLVM_FOUND)

include_directories(include ${GLFW_INCLUDE_DIRS} ${CAPSTONE_INCLUDE_DIRS} libs/ImGui/include libs/glad/include ${Python_INCLUDE_DIRS})
if (APPLE)
    include_directories(include ${OPENSSL_INCLUDE_DIRS} ${CAPSTONE_INCLUDE_DIRS})
endif (APPLE)


# Get Python major and minor
string(REPLACE "." ";" PYTHON_VERSION_MAJOR_MINOR ${Python_VERSION})
list(REMOVE_AT PYTHON_VERSION_MAJOR_MINOR 2)
list(JOIN PYTHON_VERSION_MAJOR_MINOR "." PYTHON_VERSION_MAJOR_MINOR)

SET(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -DIMGUI_IMPL_OPENGL_LOADER_GLAD -DPYTHON_VERSION_MAJOR_MINOR=\"\\\"${PYTHON_VERSION_MAJOR_MINOR}\"\\\"")

if (WIN32)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++ -static-libgcc -static")
    SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wl,-subsystem,windows")
endif (WIN32)

SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -DRELEASE")
SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DDEBUG")

add_definitions(-DLLVM_DEMANGLE=${LLVM_DEMANGLE})
if (APPLE)
    set(VAR "Python_LINK_OPTIONS" CACHE STRING "LINKER:-rpath,/Library/Developer/CommandLineTools/Library/Frameworks")
    add_link_options(${Python_LINK_OPTIONS})
endif (APPLE)

add_executable(ImHex
        source/main.cpp
        source/window.cpp

        source/helpers/utils.cpp
        source/helpers/crypto.cpp
        source/helpers/patches.cpp
        source/helpers/math_evaluator.cpp
        source/helpers/project_file_handler.cpp
        source/helpers/loader_script_handler.cpp

        source/lang/preprocessor.cpp
        source/lang/lexer.cpp
        source/lang/parser.cpp
        source/lang/validator.cpp
        source/lang/evaluator.cpp

        source/providers/file_provider.cpp

        source/views/view_hexeditor.cpp
        source/views/view_pattern.cpp
        source/views/view_pattern_data.cpp
        source/views/view_hashes.cpp
        source/views/view_information.cpp
        source/views/view_help.cpp
        source/views/view_tools.cpp
        source/views/view_strings.cpp
        source/views/view_data_inspector.cpp
        source/views/view_disassembler.cpp
        source/views/view_bookmarks.cpp
        source/views/view_patches.cpp

        libs/glad/source/glad.c

        libs/ImGui/source/imgui.cpp
        libs/ImGui/source/imgui_draw.cpp
        libs/ImGui/source/imgui_widgets.cpp
        libs/ImGui/source/imgui_demo.cpp
        libs/ImGui/source/imgui_impl_glfw.cpp
        libs/ImGui/source/imgui_impl_opengl3.cpp
        libs/ImGui/source/ImGuiFileBrowser.cpp
        libs/ImGui/source/TextEditor.cpp

        resource.rc
        )

if (LLVM_FOUND)
    target_link_directories(ImHex PRIVATE ${LLVM_LIBRARY_DIR})
endif (LLVM_FOUND)

if (APPLE)
    target_link_libraries(ImHex PRIVATE ${GLFW_LIBRARIES} ${MAGIC_LIBRARY} ${OPENSSL_CRYPTO_LIBRARY} ${CAPSTONE_LIBRARY} ${Python_LIBRARIES})
endif (APPLE)

if (WIN32)
    target_link_libraries(ImHex libglfw3.a libgcc.a libstdc++.a libmagic.a libgnurx.a libtre.a libintl.a libiconv.a shlwapi.lib libcrypto.a libwinpthread.a libcapstone.a ${llvm_libs} ${Python_LIBRARIES} nlohmann_json::nlohmann_json)
endif (WIN32)

if (UNIX AND NOT APPLE)
    target_link_libraries(ImHex libglfw.so libmagic.so libcrypto.so libdl.so libcapstone.so ${llvm_libs} ${Python_LIBRARIES} nlohmann_json::nlohmann_json)
endif (UNIX AND NOT APPLE)

install(TARGETS ImHex DESTINATION bin)
