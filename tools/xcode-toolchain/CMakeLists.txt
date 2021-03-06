# OS X 10.11 El Capitan has just been released. One of the new features, System
# Integrity Protection, prevents modifying the base OS install, even with sudo.
# This prevents LLVM developers on OS X from being able to easily install new
# system compilers. The feature can be disabled, but to make it easier for
# developers to work without disabling SIP, this file can generate an Xcode
# toolchain. Xcode toolchains are a mostly-undocumented feature that allows
# multiple copies of low level tools to be installed to different locations, and
# users can easily switch between them.

# Setting an environment variable TOOLCHAINS to the toolchain's identifier will
# result in /usr/bin/<tool> or xcrun <tool> to find the tool in the toolchain.

# To make this work with Xcode 7.1 and later you can install the toolchain this
# file generates anywhere on your system and set EXTERNAL_TOOLCHAINS_DIR to the
# path specified by $CMAKE_INSTALL_PREFIX/Toolchains

# This file generates a custom install-xcode-toolchain target which constructs
# and installs a toolchain with the identifier in the pattern:
# org.llvm.${PACKAGE_VERSION}. This toolchain can then be used to override the
# system compiler by setting TOOLCHAINS=org.llvm.${PACKAGE_VERSION} in the
# in the environment.

# Example usage:
# cmake -G Ninja -DLLVM_CREATE_XCODE_TOOLCHAIN=On
#   -DCMAKE_INSTALL_PREFIX=$PWD/install
# ninja install-xcode-toolchain
# export EXTERNAL_TOOLCHAINS_DIR=$PWD/install/Toolchains
# export TOOLCHAINS=org.llvm.3.8.0svn

# `xcrun -find clang` should return the installed clang, and `clang --version`
# should show 3.8.0svn.

if(NOT APPLE)
  return()
endif()

option(LLVM_CREATE_XCODE_TOOLCHAIN "Create a target to install LLVM into an Xcode toolchain" Off)

if(NOT LLVM_CREATE_XCODE_TOOLCHAIN)
  return()
endif()

execute_process(
  COMMAND xcrun -find otool
  OUTPUT_VARIABLE clang_path
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_FILE /dev/null
)
string(REGEX MATCH "(.*/Toolchains)/.*" toolchains_match ${clang_path})
if(NOT toolchains_match)
  message(FATAL_ERROR "Could not identify toolchain dir")
endif()
set(toolchains_dir ${CMAKE_MATCH_1})

set(XcodeDefaultInfo "${toolchains_dir}/XcodeDefault.xctoolchain/ToolchainInfo.plist")
set(LLVMToolchainDir "${CMAKE_INSTALL_PREFIX}/Toolchains/LLVM${PACKAGE_VERSION}.xctoolchain/")

add_custom_command(OUTPUT ${LLVMToolchainDir}
                    COMMAND ${CMAKE_COMMAND} -E make_directory ${LLVMToolchainDir})

add_custom_command(OUTPUT ${LLVMToolchainDir}/ToolchainInfo.plist
                  DEPENDS ${LLVMToolchainDir}
                  COMMAND ${CMAKE_COMMAND} -E copy "${XcodeDefaultInfo}" "${LLVMToolchainDir}/ToolchainInfo.plist"
                  COMMAND /usr/libexec/PlistBuddy -c "Set:Identifier org.llvm.${PACKAGE_VERSION}" "${LLVMToolchainDir}/ToolchainInfo.plist")

add_custom_target(install-xcode-toolchain
                  DEPENDS ${LLVMToolchainDir}/ToolchainInfo.plist
                  COMMAND "${CMAKE_COMMAND}" --build ${CMAKE_BINARY_DIR} --target all
                  COMMAND "${CMAKE_COMMAND}"
                          -DCMAKE_INSTALL_PREFIX=${LLVMToolchainDir}/usr/
                          -P "${CMAKE_BINARY_DIR}/cmake_install.cmake"
                  ${cmake_3_2_USES_TERMINAL})
