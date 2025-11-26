---
sort: 2
---
# Simplicity Studio v6 - IAR Build workflow

## Build using CMake CLI on Windows

With Simplicity studio, create a project selecting CMAKE (GCC / IAR)

Once generated, open a console in the created project folder

Then call below commands in the following order.

Edit IAR path `IAR_ARM_DIR` to be set in environment variables according to your own :

```bash
cd cmake_iar
$env:IAR_ARM_DIR = "C:\Program Files\IAR Systems\Embedded Workbench 9.2"
cmake --preset project 
cmake --build --preset default_config
```

**Note: these steps can also be followed to build the same project using GCC by cd'ing into `cmake_gcc`**

Due to IAR Licensing, calling `iccarm.exe` in multiple threads will **NOT** speed up build process as License checks are sequential

It is therefore recommended to follow next step, and use IAR EWARM IDE instead, or dedicated cross buid tools :

[IAR Cross Build](https://github.com/iarsystems/cmake-tutorial)

## Build using EWARM IDE on Windows

With Simplicity studio, create a project selecting CMAKE (GCC / IAR)

Then follow below steps :

1. Once generated, open the `<project_name>.slcp` file (should be automatic)
2. Click the "Edit SDK / Generators" button
3. Add `IAR EWARM` to the list
4. Save the file, and wait for Studio 6 to finish generating the project
5. Go to the containing folder of the generated project and open the `<project_name>.eww`

**Note: such a project would support being built using all three options (CMAKE GCC, CMAKE IAR, EW IAR)**

## Build with IAR on MACOS (and probably Linux) - not working - suggestions are welcome

For users using Mac OS or Linux based systems, it is possible to call IAR toolchin via wine

### Prerequisites

* Install wine using your system package manager (`brew` or `apt`)
* Install IAR Embedded Workbench using Wine
* Download IAR wrappers scripts from : `TODO`
* Run IAR Licence Manager using wrapper script and activate

Failing to activate IAR Licence will result in build failure at cmake calls below

### Project Creation and cmake edit

1. With Simplicity studio, create a project selecting CMAKE (GCC / IAR)
2. Once generated, open a console or vscode in the created project folder
3. Go into `cmake_iar` folder
4. Edit the `toolchain.cmake` file and add lines :

```cmake
# -------------- Stop CMake from “detecting” --------------
# Provide the ID/version so CMake doesn’t guess:
set(CMAKE_C_COMPILER_ID "IAR")
set(CMAKE_CXX_COMPILER_ID "IAR")
set(CMAKE_ASM_COMPILER_ID "IAR")

set(CMAKE_C_COMPILER_VERSION "9.40.1")
set(CMAKE_CXX_COMPILER_VERSION "9.40.1")
set(CMAKE_ASM_COMPILER_VERSION "9.40.1")

set(CMAKE_C_COMPILER_ARCHITECTURE_ID ARM)
set(CMAKE_ASM_COMPILER_ARCHITECTURE_ID ARM)
set(CMAKE_CXX_COMPILER_ARCHITECTURE_ID ARM)

set(CMAKE_CXX_FLAGS_INIT "--c++")
```

5. Edit your TOOLCHAIN_DIR path to point towards your wrappers location

**NOTE : Path MUST be absolute, otherwise cmake calls will fail**

Once done you should be able to build your project as in windows :

```bash
cd cmake_iar
cmake --preset project 
cmake --build --preset default_config
```

Due to that architecture, build times are much slower than they would be using IAR EWARM or native GCC

*Non explored aleternatives are :*

Use IAR’s native Linux tools (best for CI/build servers)

IAR provides IAR Build Tools for Arm as native Linux command-line compilers/linkers (no GUI).

If your goal is headless builds, this is the cleanest path on Linux.

(They specifically list Linux OS support for these build tools.)
