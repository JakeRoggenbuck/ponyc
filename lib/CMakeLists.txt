cmake_minimum_required(VERSION 3.11 FATAL_ERROR)

file(STRINGS "../VERSION" PONYC_PROJECT_VERSION)
project(ponyclibs VERSION ${PONYC_PROJECT_VERSION} LANGUAGES C CXX)

include(ExternalProject)

if(NOT DEFINED PONYC_LIBS_BUILD_TYPE)
    set(PONYC_LIBS_BUILD_TYPE Release)
endif()

set(PONYC_GBENCHMARK_URL https://github.com/google/benchmark/archive/v1.5.4.tar.gz)
if(${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD")
    set(PONYC_GBENCHMARK_URL https://github.com/google/benchmark/archive/v1.5.3.tar.gz)
endif()

ExternalProject_Add(gbenchmark
    URL ${PONYC_GBENCHMARK_URL}
    CMAKE_ARGS -DCMAKE_BUILD_TYPE=${PONYC_LIBS_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DBENCHMARK_ENABLE_GTEST_TESTS=OFF -DCMAKE_CXX_FLAGS=-fpic --no-warn-unused-cli
)

ExternalProject_Add(googletest
    URL https://github.com/google/googletest/archive/release-1.10.0.tar.gz
    CMAKE_ARGS -DCMAKE_BUILD_TYPE=${PONYC_LIBS_BUILD_TYPE} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DCMAKE_CXX_FLAGS=-fpic -Dgtest_force_shared_crt=ON --no-warn-unused-cli
)

add_library(blake2 STATIC blake2/blake2b-ref.c)
set_property(TARGET blake2 PROPERTY POSITION_INDEPENDENT_CODE ON)

install(TARGETS blake2
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    COMPONENT library
)

find_package(Git)

set(LLVM_DESIRED_HASH "fed41342a82f5a3a9201819a82bf7a48313e296b")
set(PATCHES_DESIRED_HASH "3a655193262fd9b2e87340e096efcbd96726a07fe6dd42a263f3a4fc2dc0192e")

if(GIT_FOUND)
    if(EXISTS "${PROJECT_SOURCE_DIR}/../.git")
        # Update submodules as needed
        option(GIT_SUBMODULE "Check submodules during build" ON)
        if(GIT_SUBMODULE)
            message(STATUS "Updating submodules...")
            execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive --depth 1
                            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                            RESULT_VARIABLE git_submod_result)
            #message("git_submod_result ${git_submod_result}")
            if(NOT git_submod_result EQUAL "0")
                message(FATAL_ERROR "git submodule update --init --recursive --depth 1 failed with ${git_submod_result}, please checkout submodules")
            endif()

            # we check to make sure the submodule hash matches
            # the reason the submodule hash is in this file is to be able to use this file as a key for caching the libs in CI
            execute_process(COMMAND ${GIT_EXECUTABLE} submodule status
                            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                            OUTPUT_VARIABLE git_submod_output)
            #message("git_submod_output ${git_submod_output}")
            string(FIND "${git_submod_output}" "${LLVM_DESIRED_HASH}" LLVM_SUBMOD_POS)
            if(LLVM_SUBMOD_POS EQUAL "-1")
                message(FATAL_ERROR "Expecting the lib/llvm/src submodule to be at hash '${LLVM_DESIRED_HASH}'; found '${git_submod_output}'; update the LLVM_DESIRED_HASH variable in lib/CMakeLists.txt if you've updated the submodule.")
            endif()
        endif()
    endif()

    # Apply patches
    message("Applying patches...")
    file(GLOB PONY_LLVM_PATCHES "${PROJECT_SOURCE_DIR}/llvm/patches/*.diff")

    # check to see if the patch hashes match
    message("Checking patches ${PONY_LLVM_PATCHES}")
    set(PATCHES_ACTUAL_HASH "")
    foreach (PATCH ${PONY_LLVM_PATCHES})
        file(STRINGS ${PATCH} patch_file NEWLINE_CONSUME)
        string(REPLACE "\n" " " patch_file ${patch_file})
        string(SHA256 patch_file_hash ${patch_file})
        # message("${PATCH}: '${patch_file_hash}'")
        string(CONCAT PATCHES_ACTUAL_HASH ${PATCHES_ACTUAL_HASH} ${patch_file_hash})
        # message("concat is '${PATCHES_ACTUAL_HASH}'")
    endforeach()
    string(SHA256 PATCHES_ACTUAL_HASH ${PATCHES_ACTUAL_HASH})
    # message("Desired hash ${PATCHES_DESIRED_HASH}")
    # message("Actual hash  ${PATCHES_ACTUAL_HASH}")
    if(NOT PATCHES_ACTUAL_HASH EQUAL "${PATCHES_DESIRED_HASH}")
        message(FATAL_ERROR "Patch hash actual ${PATCHES_ACTUAL_HASH} does not match desired ${PATCHES_DESIRED_HASH}")
    endif()

    foreach (PATCH ${PONY_LLVM_PATCHES})
        message("  Checking ${PATCH}")
        execute_process(COMMAND ${GIT_EXECUTABLE} apply --check -p 1 --ignore-whitespace --whitespace=nowarn ${PATCH}
                        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/llvm/src"
                        ERROR_VARIABLE _err_out
                        RESULT_VARIABLE git_apply_check_result)
        if(git_apply_check_result EQUAL "0")
            message("    Applying ${PATCH}")
            execute_process(COMMAND ${GIT_EXECUTABLE} apply -p 1 --ignore-whitespace --whitespace=nowarn ${PATCH}
                            WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/llvm/src"
                            RESULT_VARIABLE git_apply_result)
            if(NOT git_apply_result EQUAL "0")
                message(FATAL_ERROR "Unable to apply ${PATCH}")
            endif()
        else()
            message("    Already applied ${PATCH}")
        endif()
    endforeach()
else()
    message(FATAL_ERROR "Git not found!")
endif()

message("Building targets: ${LLVM_TARGETS_TO_BUILD}")

set(LLVM_ENABLE_BINDINGS OFF)
set(LLVM_ENABLE_OCAMLDOC OFF)
set(LLVM_ENABLE_PIC OFF)
set(LLVM_ENABLE_TERMINFO OFF)
set(LLVM_ENABLE_WARNINGS OFF)
set(LLVM_ENABLE_Z3_SOLVER OFF)
set(LLVM_INCLUDE_BENCHMARKS OFF)
set(LLVM_INCLUDE_TESTS OFF)
set(LLVM_TOOL_REMARKS_SHLIB_BUILD OFF)
set(LLVM_ENABLE_ZLIB OFF)
set(LLVM_ENABLE_ZLIB:STRING OFF)

if(${CMAKE_HOST_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mmacosx-version-min=10.14 -DUSE_SCHEDULER_SCALING_PTHREADS")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -mmacosx-version-min=10.14")
endif()

add_subdirectory(llvm/src/llvm)
