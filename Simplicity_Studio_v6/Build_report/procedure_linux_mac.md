---
sort: 2
---
## Procedure on Linux or MacOS

for Windows 11 please go to [this page](procedure_win.md).

Once you have created a project in Simplicity Studio and opened it in VSCode, you get access to its directory tree.

In this tree, there a subfolder name **cmake_gcc** holding all the compilation oriented files for the Cmake tool. This is also were the compilation will happen, creating a sub directory name build for this purpose.

But lets stay first in this directory. Once the project is created or even changed from the studio GUI by adding a component for example, some of thoise files are automaticaly updated, therefore any change we could had in purpose of generating a build report would be lost. The best is to take advantage of the **CMakeLists.txt** file here.

The CMakeLists.txt file defines quite some compilation rules, and the one of interest for our build report is the post build related one, usually at the end of the file, starting with:

```c
add_custom_command(TARGET YOUR_PROJECT_NAME
    POST_BUILD
```

We will add 2 lines to set the working directory and launching some commands.

## Adding the working directory path:

Just add the last line of the below in your POST_BUILD command list, changing YOUR_PROJECT_NAME accordingly. The name is the one in the project section at the beginning of the CMakeLists.txt file.

```c
add_custom_command(TARGET YOUR_PROJECT_NAME
    POST_BUILD
... // keep existing lines and add the next one
    WORKING_DIRECTORY $<TARGET_FILE_DIR:YOUR_PROJECT_NAME>
```

## Reporting Flash and RAM usage:

To report Flash and RAM usage, we will have to use the output of **arm-none-eabi-size tool**. As below we will reformat the output, let's store it in a temporary file name **output.txt**.

Just add those lines after the WORKING_DIRECTORY one, of course changing YOUR_PROJECT_NAME again.

The second line call a batch file to generate and print the build report.

```c
COMMAND ${CMAKE_SIZE_UTIL} -A --radix=10 "$<TARGET_FILE_BASE_NAME:YOUR_PROJECT_NAME>.out" > output.txt
COMMAND bash ${CMAKE_SOURCE_DIR}/build_report.sh
```

Create a **build_report.sh** file in your **cmake_gcc** directory and add those lines:

```bash
awk '
BEGIN{
print"\n-----------------------------------\n\
Build summary:\
\n-----------------------------------"
  ram_no_heap=0; heap=0; flash=0; nvm=0; data_size=0
}
# Process only lines where column 2 is numeric
$2 ~ /^[0-9]+$/ {
  name=$1; size=$2

  # RAM without heap
  if (name==".stack" || name==".bss" || name=="text_application_ram" || name==".data") {
    ram_no_heap += size
  }

  # Heap
  if (name==".memory_manager_heap") {
    heap += size
  }

  # Flash (code + tables + NVM)
  if (name==".vectors" || name==".text" || name==".ARM.exidx" || name==".copy.table" || name==".zero.table" || name==".nvm") {
    flash += size
  }

  # Track NVM separately
  if (name==".nvm") { nvm = size }

  # .data initialization image included in flash
  if (name==".data") { data_size = size }
}
END{
  flash_with_image = flash + data_size
  total_ram = ram_no_heap + heap

  printf("RAM:   %d bytes (with HEAP: %d bytes)\n", ram_no_heap, total_ram)
  printf("FLASH: %d bytes (including %d bytes of NVM)\n\r", flash_with_image, nvm)
}' output.txt
```

this will generate from the output something like:

```c
-----------------------------------
Build summary:
-----------------------------------
RAM:   11612 bytes (with HEAP: 32768 bytes)
FLASH: 245756 bytes (including 40960 bytes of NVM)
```

## Reporting Build time:

To report build time, we will take advantage of the **.ninja_log** file generated at compile time and stored in  the **build** directory. This files contains start and stop times in ms for all the cmake actions, therefore all file compilation and  linking time to generate the **.out**.

adding those lines to the build_report.sh will parse the file to generate useful summary:

```bash
awk 'NR>1 {sum += $2 - $1
} 
END {print "\n-----------------------------------\n\
Build time:\
\n-----------------------------------"
    sum = sum
    if (sum >= 60000) {
        m = int(sum / 60000)
        s = int((sum % 60000) / 1000)
        ms = sum % 1000
        printf "TOTAL:         %dm %ds %dms\n", m, s, ms
    } else if (sum >= 1000) {
        s = int(sum / 1000)
        ms = sum % 1000
        printf "TOTAL:         %ds %dms\n", s, ms
    } else {
        printf "TOTAL:         %dms\n", sum
        }
    }' ../.ninja_log
awk 'NR>1 {
    if ($1 < min || NR==2) min=$1
    if ($2 > max) max=$2
}
END {
    total = (max - min)
    if (total > 60000) {
        m = int(total / 60000)
        s = int((total % 60000) / 1000)
        ms = total % 1000
        printf "MULTITHREADED: %dm %ds %dms\n", m, s, ms
    } else if (total >= 1000) {
        s = int(total / 1000)
        ms = total % 1000
        printf "MULTITHREADED: %ds %dms\n", s, ms
    } else {
        printf "MULTITHREADED: %dms\n", total
    }
}' ../.ninja_log
```

this will generate on the standart output something like:

```c
-----------------------------------
Build time:
-----------------------------------
TOTAL:         21m 57s 858ms
MULTITHREADED: 1m 19s 584ms
```
