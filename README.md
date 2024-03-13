[![Build Test](https://github.com/jasonyang-ee/STM32-CMAKE-TEMPLATE/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/jasonyang-ee/STM32-CMAKE-TEMPLATE/actions/workflows/build.yml)
[![Deploy Docs](https://github.com/jasonyang-ee/STM32-CMAKE-TEMPLATE/actions/workflows/mdbook.yml/badge.svg)](https://github.com/jasonyang-ee/STM32-CMAKE-TEMPLATE/actions/workflows/mdbook.yml)


# STM32 CMake Template

A CMake template repo to allow quick porting to start a new STM32 project.

This instruction will be focusing on Windows environment setup with using VS Code.

Project using STM32L432KC as example. Test hardware is NUCLEO-L432KC.

## Documentation

### Visit [Documentation](https://doc.jasony.org/STM32-CMAKE-TEMPLATE/) for more information.

&nbsp;

&nbsp;

&nbsp;

# Simplified Instruction

## Toolchain

- [ARM GNU](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
- [CMake](https://cmake.org/download/)
- [Ninja](https://github.com/ninja-build/ninja/releases)
- ST Link GDB Server (Copy from CubeIDE Installation).
- STM32_Programmer_CLI (Copy from CubeIDE Installation)


&nbsp;


## VS Code Extensions

- CMake
- CMake Tools
- Cortex-Debug
- Memory View
- RTOS Views


&nbsp;



## CMakeList.txt file

This is the main CMake setup file.

- Make new file in project root: `CMakeList.txt`

- Modify `project name`, `linker file`, and `MCU sepecific setting`.

- If you are using this project folder structure, you may run the bash script `.\getIncludeList.sh` and `.\getSourceList.sh` to auto scan `/Application` folder for generating CMake source list.
  
- Otherwise, you will have to modify `/camke/IncludeList.cmake` and `/cmake/IncludeList.cmake`.

> CMakeList.txt

```makefile
# Define needed CMake verion
cmake_minimum_required(VERSION 3.22)


# Setup cmake module path and compiler settings
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
# Print current build type to console
message("Build type: "              ${CMAKE_BUILD_TYPE})
# Setup C and C++ version
set(CMAKE_C_STANDARD                11)
set(CMAKE_C_STANDARD_REQUIRED       ON)
set(CMAKE_C_EXTENSIONS              ON)
set(CMAKE_CXX_STANDARD              17)
set(CMAKE_CXX_STANDARD_REQUIRED     ON)
set(CMAKE_CXX_EXTENSIONS            ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS   ON)
# Define current path for shorter reference below
set(PROJ_PATH                       ${CMAKE_CURRENT_SOURCE_DIR})
# Define .cmake module for toolchain compile flags that does holds true for all ARM projects
# This path is defined in the list() function above
set(CMAKE_TOOLCHAIN_FILE            gcc-arm-none-eabi)


# Project Name    --- MUST EDIT ---
project(L432KC-Template)
# Part of project name but made seperate for ease of editing project name
enable_language(C CXX ASM)
# Linker File     --- MUST EDIT ---
set(linker_script_SRC               ${PROJ_PATH}/Core/STM32L432KCUX_FLASH.ld)
# The use project name for binary file name
set(EXECUTABLE                      ${CMAKE_PROJECT_NAME})


# MCU Sepecific Setting    --- MUST EDIT ---
# Make multiple for various STM32 core
# This path is defined in the list() function above
include(STM32L432xx_HAL_PARA)

# .cmake module generated by using .\getIncludeList.sh and .\getSourceList.sh
# Those two file contains all the project source file list and include list
# This path is defined in the list() function above
include(SourceList)
include(IncludeList)


# Executable files
add_executable(${EXECUTABLE} ${source_list})
# Include paths
target_include_directories(${EXECUTABLE} PRIVATE ${include_list})
# Project symbols
target_compile_definitions(${EXECUTABLE} PRIVATE ${compiler_define})
# Compiler options
target_compile_options(${EXECUTABLE} PRIVATE
	${CPU_PARAMETERS}
	-Wall
	-Wpedantic
	-Wno-unused-parameter
)
# Linker options
target_link_options(${EXECUTABLE} PRIVATE
	-T${linker_script_SRC}
	${CPU_PARAMETERS}
	-Wl,-Map=${CMAKE_PROJECT_NAME}.map
	--specs=nosys.specs
	#-u _printf_float                # STDIO float formatting support
	-Wl,--start-group
	-lc
	-lm
	-lstdc++
	-lsupc++
	-Wl,--end-group
	-Wl,--print-memory-usage
)
# Execute post-build to print size
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD
	COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${EXECUTABLE}>
)
# Convert output to hex and binary
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD
	COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${EXECUTABLE}> ${EXECUTABLE}.hex
)
# Convert to bin file -> add conditional check?
add_custom_command(TARGET ${EXECUTABLE} POST_BUILD
	COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${EXECUTABLE}> ${EXECUTABLE}.bin
)
```



&nbsp;




## Toolchain file

CMake needs to be aware about toolchain we would like to use to finally compile the project with. This file will be universal across projects.

- Make new folder in project root: `cmake`
- Make new file in folder /cmake: `./cmake/gcc-arm-none-eabi.cmake`

> gcc-arm-none-eabi.cmake

```makefile
set(CMAKE_SYSTEM_NAME               Generic)
set(CMAKE_SYSTEM_PROCESSOR          arm)

# Some default GCC settings
# arm-none-eabi- must be part of path environment
set(TOOLCHAIN_PREFIX                arm-none-eabi-)
set(FLAGS                           "-fdata-sections -ffunction-sections --specs=nano.specs -Wl,--gc-sections")
set(CPP_FLAGS                       "-fno-rtti -fno-exceptions -fno-threadsafe-statics")

# Define compiler settings
set(CMAKE_C_COMPILER                ${TOOLCHAIN_PREFIX}gcc ${FLAGS})
set(CMAKE_ASM_COMPILER              ${CMAKE_C_COMPILER})
set(CMAKE_CXX_COMPILER              ${TOOLCHAIN_PREFIX}g++ ${FLAGS} ${CPP_FLAGS})
set(CMAKE_OBJCOPY                   ${TOOLCHAIN_PREFIX}objcopy)
set(CMAKE_SIZE                      ${TOOLCHAIN_PREFIX}size)

set(CMAKE_EXECUTABLE_SUFFIX_ASM     ".elf")
set(CMAKE_EXECUTABLE_SUFFIX_C       ".elf")
set(CMAKE_EXECUTABLE_SUFFIX_CXX     ".elf")

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
```




&nbsp;




## MCU sepecific file

Each MCU has their own ARM compiler flags. Those are defined in a individual module for portability.

> STM32L432xx_HAL_PARA.cmake

```makefile
set(CPU_PARAMETERS ${CPU_PARAMETERS}
    -mthumb
    -mcpu=cortex-m4
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
)

set(compiler_define ${compiler_define}
    "USE_HAL_DRIVER"
    "STM32L432xx"
)
```

> **General rule for settings would be as per table below:**

| STM32 Family | -mcpu           | -mfpu         | -mfloat-abi |
| ------------ | --------------- | ------------- | ----------- |
| STM32F0      | `cortex-m0`     | `Not used`    | `soft`      |
| STM32F1      | `cortex-m3`     | `Not used`    | `soft`      |
| STM32F2      | `cortex-m3`     | `Not used`    | `soft`      |
| STM32F3      | `cortex-m4`     | `fpv4-sp-d16` | `hard`      |
| STM32F4      | `cortex-m4`     | `fpv4-sp-d16` | `hard`      |
| STM32F7 SP   | `cortex-m7`     | `fpv5-sp-d16` | `hard`      |
| STM32F7 DP   | `cortex-m7`     | `fpv5-d16`    | `hard`      |
| STM32G0      | `cortex-m0plus` | `Not used`    | `soft`      |
| STM32C0      | `cortex-m0plus` | `Not used`    | `soft`      |
| STM32G4      | `cortex-m4`     | `fpv4-sp-d16` | `hard`      |
| STM32H7      | `cortex-m7`     | `fpv5-d16`    | `hard`      |
| STM32L0      | `cortex-m0plus` | `Not used`    | `soft`      |
| STM32L1      | `cortex-m3`     | `Not used`    | `soft`      |
| STM32L4      | `cortex-m4`     | `fpv4-sp-d16` | `hard`      |
| STM32L5      | `cortex-m33`    | `fpv5-sp-d16` | `hard`      |
| STM32U5      | `cortex-m33`    | `fpv5-sp-d16` | `hard`      |
| STM32WB      | `cortex-m4`     | `fpv4-sp-d16` | `hard`      |
| STM32WL CM4  | `cortex-m4`     | `Not used`    | `soft`      |
| STM32WL CM0  | `cortex-m0plus` | `Not used`    | `soft`      |




&nbsp;





## Source List and Include List file

Project source and include list are required for CMake to build the project in `/cmake` folder.

The format of the list must be full path.


&nbsp;



## Auto Scan Source and Include List

Auto scan bash script has been made for STM32CubeMX generated files structure

- In terminal `` Ctrl + ` ``, run `.\getIncludeList.sh` and `.\getSourceList.sh`

- A list of scanned source and header will be saved in `/cmake` folder.

> You may modify bash file to expend the auto file searching for more folders.

> The bash simply scan `.c` `.cpp` `.s` file for source. And, it scan `/Inc` `/Include` for include path.




&nbsp;





## CMakePresets.json file

`CMakePresets.json` provides definition for user configuration. Having this file allows developer to quickly change between debug and release mode.

- Create file `CMakePresets.json` in Project Root

> CMakePresets.json

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "default",
      "hidden": true,
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "toolchainFile": "${sourceDir}/cmake/gcc-arm-none-eabi.cmake",
      "cacheVariables": {
      "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    },
    {
      "name": "Debug",
      "inherits": "default",
      "cacheVariables": {
      "CMAKE_BUILD_TYPE": "Debug"
      }
    },
    {
      "name": "RelWithDebInfo",
      "inherits": "default",
      "cacheVariables": {
      "CMAKE_BUILD_TYPE": "RelWithDebInfo"
      }
    },
    {
      "name": "Release",
      "inherits": "default",
      "cacheVariables": {
      "CMAKE_BUILD_TYPE": "Release"
      }
    },
    {
      "name": "MinSizeRel",
      "inherits": "default",
      "cacheVariables": {
      "CMAKE_BUILD_TYPE": "MinSizeRel"
      }
    }
  ]
}
```


&nbsp;





## Debug Project 

This is using VS Code `Tasks` feature and Extention `cortex-debug`

- Create `.vscode/launch.json`

- Open debug tab. And our named debug preset `ST-Link` should be available to run `Green Icon` or `F5`.

> launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "ST-Link",
      "cwd": "${workspaceFolder}",
      "executable": "${command:cmake.launchTargetPath}",
      "request": "launch",
      "type": "cortex-debug",
      "servertype": "stlink",
      "interface": "swd",
      "showDevDebugOutput": "both",
      "v1": false,                            // ST-Link version
      "device": "STM32L432KC",                // MCU used [optional]
      "serialNumber": "",                     // Set ST-Link ID if you use multiple at the same time [optional]
      "runToEntryPoint": "main",              // Run to main and stop there [optional]
      "svdFile": "STM32_svd/STM32L4x2.svd"    // SVD file to see registers [optional]

      // "servertype": "stlink", will try to run command "STM32_Programmer_CLI", "ST-LINK_gdbserver", and  which must exist in your system PATH.

      // If using SWO to see serial wire view, you will have to setup "servertype": "OpenOCD". Please refer to the extension github page to learn details.
    }
  ]
}
```



&nbsp;





## Flash to Target

We are using VS Code Task `Ctrl + Shift + P` -> Enter `Tasks: run task`. This will allow auto excution of custom terminal commands.

Setting keyboard short cut `Ctrl + T` for this is going to help you very much.

The configuration can be defined by creating file `.vscode/tasks.json`

> tasks.json

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "shell",
      "label": "Windows: Flash Firmware",
      "command": "STM32_Programmer_CLI",
      "args": [
        "--connect",
        "port=swd",
        "--download",
        "${command:cmake.launchTargetPath}",
        "-rst",
        "-run"
      ],
      "options": {
        "cwd": "${workspaceFolder}"
      },
      "problemMatcher": []
    }
  ]
}
```



&nbsp;



## Docker Container for STM32 CMake & Ninja Compiling

### Dockerfile Details: https://github.com/jasonyang-ee/STM32-Dockerfile.git

-+- TL;DR -+-

This docker image auto clone an online git repo and compile the CMake & Ninja supported STM32 project locally on your computer with mounted volume.
```bash
docker run -v "{Local_Full_Path}":"/home" jasonyangee/stm32-builder:ubuntu-latest -r {Git_Repo_URL}
```

![Run](docs_src/page/img/run_time.gif)


### Docker Image

Public Registry:
> ghcr.io/jasonyang-ee/stm32-builder:ubuntu-latest

> ghcr.io/jasonyang-ee/stm32-builder:alpine-latest

> ghcr.io/jasonyang-ee/stm32-builder:arch-latest

> jasonyangee/stm32-builder:ubuntu-latest

> jasonyangee/stm32-builder:alpine-latest

> jasonyangee/stm32-builder:arch-latest