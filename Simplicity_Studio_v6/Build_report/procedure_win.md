---
sort: 2
---
## Procedure on Windows 11

for Linux or MacOS please go to [this page](procedure_linux_mac.md).

Once you have created a project in Simplicity Studio and opened it in VSCode, you get access to its directory tree.

In this tree, there a subfolder name **cmake_gcc** holding all the compilation oriented files for the Cmake tool. This is also were the compilation will happen, creating a sub directory name build for this purpose.

But lets stay first in this directory. Once the project is created or even changed from the studio GUI by adding a component for example, some of thoise files are automaticaly updated, therefore any change we could had in purpose of generating a build report would be lost. The best is to take advantage of the **CMakeLists.txt** file here.

## Modifications to CMakeLists.txt:

The **CMakeLists.txt** file defines quite some compilation rules, and the one of interest for our build report is the post build related one, usually at the end of the file. We will add 3 lines to set the working directory and launching some commands. Here's the end of the file and the modifications to be made. Don't forget to change YOUR_PROJECT_NAME by the one in your project file!
Explanations on the lines will come in the next chapters.

```diff
# Create .bin, .hex and .s37 artifacts after building the project
add_custom_command(TARGET YOUR_PROJECT_NAME
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_SREC_CMD} "$<TARGET_FILE:YOUR_PROJECT_NAME>" "$<TARGET_FILE_DIR:rfsense_config_202562>/$<TARGET_FILE_BASE_NAME:YOUR_PROJECT_NAME>.s37"
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_IHEX_CMD} "$<TARGET_FILE:YOUR_PROJECT_NAME>" "$<TARGET_FILE_DIR:rfsense_config_202562>/$<TARGET_FILE_BASE_NAME:YOUR_PROJECT_NAME>.hex"
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_BIN_CMD}  "$<TARGET_FILE:YOUR_PROJECT_NAME>" "$<TARGET_FILE_DIR:rfsense_config_202562>/$<TARGET_FILE_BASE_NAME:YOUR_PROJECT_NAME>.bin" 
+    WORKING_DIRECTORY "$<TARGET_FILE_DIR:YOUR_PROJECT_NAME>"
+    COMMAND ${CMAKE_SIZE_UTIL} -A --radix=10 "$<TARGET_FILE_BASE_NAME:YOUR_PROJECT_NAME>.out" > output.txt
+    COMMAND powershell -ExecutionPolicy Bypass -File "${CMAKE_SOURCE_DIR}/build_report.ps1"
    )

# Run post-build pipeline to perform additional post-processing
if(post_build_command)
add_custom_command(TARGET YOUR_PROJECT_NAME
    POST_BUILD
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/..
    COMMAND ${post_build_command}
)
endif()
```

## Reporting Flash and RAM usage:

The last 2 lines added above helps running 2 tools.

To report Flash and RAM usage, we will have to use the output of **arm-none-eabi-size tool**. As below we will reformat the output, let's store it in a temporary file name **output.txt**.

The second call a powershell batch file to generate and print the build report.

Create a **[build_report.ps1](https://gist.github.com/siliconlabs-southemea/72e9b3713fc9e01315893fc83a1266c0)** file in your **cmake_gcc** directory and add those lines:

```bash
function Print-MemorySummary([string]$path) {
    if (-not (Test-Path $path)) {
        Write-Host "`n-----------------------------------"
        Write-Host "Build summary:"
        Write-Host "-----------------------------------"
        Write-Host "output.txt not found at '$path'." 
        return
    }
 
    $ramNoHeap = 0L
    $heap = 0L
    $flash = 0L
    $nvm = 0L
    $dataSize = 0L
 
    # Expect lines like: "<section> <size>"
    Get-Content -LiteralPath $path | ForEach-Object {
        $line = $_.Trim()
        if ($line -eq "") { return }
 
        $parts = $line -split "\s+"
        if ($parts.Count -lt 2) { return }
 
        $name = $parts[0]
        if (-not ($parts[1] -match '^\d+$')) { return }
        $size = [int64]$parts[1]
 
        switch ($name) {
            ".stack" { $ramNoHeap += $size; continue }
            ".bss" { $ramNoHeap += $size; continue }
            "text_application_ram" { $ramNoHeap += $size; continue }
            ".data" { $ramNoHeap += $size; $dataSize = $size; continue }
            ".memory_manager_heap" { $heap += $size; continue }
            ".vectors" { $flash += $size; continue }
            ".text" { $flash += $size; continue }
            ".ARM.exidx" { $flash += $size; continue }
            ".copy.table" { $flash += $size; continue }
            ".zero.table" { $flash += $size; continue }
            ".nvm" { $flash += $size; $nvm = $size; continue }
            default { }
        }
    }
 
    $flashWithImage = $flash + $dataSize
    $totalRam = $ramNoHeap + $heap
 
    Write-Host "`n-----------------------------------"
    Write-Host "Build summary:"
    Write-Host "-----------------------------------"
    Write-Host ("RAM:   {0} bytes (with HEAP: {1} bytes)" -f $ramNoHeap, $totalRam)
    # Keep the same trailing CR as your awk print for FLASH line
    Write-Host ("FLASH: {0} bytes (including {1} bytes of NVM)`r" -f $flashWithImage, $nvm)
}

# ---------- run ----------
# Adjust paths if your build dir layout is different.
$outputTxt = Join-Path -Path $PSScriptRoot -ChildPath ".\build\base\output.txt"

Print-MemorySummary -path $outputTxt
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

adding those lines to the **[build_report.ps1](https://gist.github.com/siliconlabs-southemea/72e9b3713fc9e01315893fc83a1266c0)** will parse the file to generate useful summary:

```bash
$ErrorActionPreference = "Stop"
 
# ---------- helpers ----------
function Format-Duration([long]$ms) {
    if ($ms -ge 60000) {
        $m = [math]::Floor($ms / 60000)
        $s = [math]::Floor(($ms % 60000) / 1000)
        $ms = $ms % 1000
        return ("{0}m {1}s {2}ms" -f $m, $s, $ms)
    }
    elseif ($ms -ge 1000) {
        $s = [math]::Floor($ms / 1000)
        $ms = $ms % 1000
        return ("{0}s {1}ms" -f $s, $ms)
    }
    else {
        return ("{0}ms" -f $ms)
    }
}
 
function Read-NinjaLog([string]$path) {
    if (-not (Test-Path $path)) { throw ".ninja_log not found at '$path'." }
    $lines = Get-Content -LiteralPath $path
    if ($lines.Count -lt 2) { return @() }
 
    # Skip header, keep only entries that start with digits; split on tabs.
    $entries =
    $lines |
    Select-Object -Skip 1 |
    Where-Object { $_ -match '^\d' } |
    ForEach-Object {
        $p = $_ -split "`t"
        # start end restat output hash
        [pscustomobject]@{
            Start = [int64]$p[0]
            End   = [int64]$p[1]
            Out   = if ($p.Count -gt 3) { $p[3] } else { "" }
        }
    }
 
    return $entries
}
 
function Print-BuildTimes([string]$ninjaLogPath) {
    $entries = Read-NinjaLog $ninjaLogPath
    if ($entries.Count -eq 0) {
        Write-Host "`n-----------------------------------"
        Write-Host "Build time:"
        Write-Host "-----------------------------------"
        Write-Host "TOTAL:         0ms"
        Write-Host "MULTITHREADED: 0ms"
        return
    }
 
    # Sum of per-edge durations (serial time)
    $sum = ($entries | ForEach-Object { $_.End - $_.Start } | Measure-Object -Sum).Sum
 
    # Wall time: max(end) - min(start)
    $minStart = ($entries | Measure-Object -Property Start -Minimum).Minimum
    $maxEnd = ($entries | Measure-Object -Property End   -Maximum).Maximum
    $wall = [int64]($maxEnd - $minStart)
 
    Write-Host "`n-----------------------------------"
    Write-Host "Build time:"
    Write-Host "-----------------------------------"
    Write-Host ("TOTAL:         {0}" -f (Format-Duration $sum))
    Write-Host ("MULTITHREADED: {0}" -f (Format-Duration $wall))
}

# ---------- run ----------
# Adjust paths if your build dir layout is different.
$ninjaLog = Join-Path -Path $PSScriptRoot -ChildPath ".\build\.ninja_log"
 
Print-BuildTimes -ninjaLogPath $ninjaLog
```

this will generate on the standart output something like:

```c
-----------------------------------
Build time:
-----------------------------------
TOTAL:         21m 57s 858ms
MULTITHREADED: 1m 19s 584ms
```
