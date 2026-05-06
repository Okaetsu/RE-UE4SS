# Linux / Proton Crash Context

## The Problem
When running Palworld on Linux (via Proton or Wine with the Goldberg emulator) using RE-UE4SS, the game experiences severe camera stuttering eventually leading to a hard crash.

## The Cause
The crash is caused by the `BPModLoaderMod` (Blueprint Mod Loader) component of UE4SS. Specifically, the `IterateGameDirectories` function in C++ relies on `<filesystem>` to traverse directories.

On Linux/Wine, the root filesystem is typically mapped to the `Z:` drive. Within this filesystem, there are symbolic links that create circular references (e.g., `/usr/bin/X11` often symlink back to itself or its parent). The C++ traversal code does not check for symbolic links, causing it to enter an infinite recursion loop.

## Log Evidence
This is the error that floods `UE4SS.log` thousands of times per second, saturating the CPU and disk IO until the game crashes:

```text
[2026-05-06 01:31:38.2093999] Error processing directory entry: directory_entry::status: The filename cannot be resolved.: "z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\text{z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11"
[2026-05-06 01:31:38.4894379] Error processing directory entry: directory_entry::status: The filename cannot be resolved.: "z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\text{z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\text{z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11"
[2026-05-06 01:31:38.7690931] Error processing directory entry: directory_entry::status: The filename cannot be resolved.: "z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\text{z:bin\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11\X11"
[... repeats infinitely ...]
```

## Solution Strategy
The upcoming patch in our fork will introduce a check using `std::filesystem::is_symlink(entry.path())` inside `LuaMod.cpp` (around line 2628+) to skip these entries and break the loop.
