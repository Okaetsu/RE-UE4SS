# RE-UE4SS Linux Recursion Bug - Development Plan

## Status
**Completed:** Stability fixes implemented and verified.

## Objective
Fix the infinite directory recursion bug (the `z:bin\X11` error) that causes `BPModLoaderMod` and LogicMods to crash or lag severely when running via Proton/Wine on Linux.

## Implemented Fixes

### 1. Robust Directory Iteration
Modified `UE4SS/src/Mod/LuaMod.cpp` to use `std::filesystem::directory_options::skip_permission_denied` and added explicit checks for `is_symlink()`.

### 2. CPU Efficiency
Updated the async event loop to use `std::atomic<bool>` for pausing and added `std::this_thread::sleep_for(5ms)` to eliminate 100% CPU busy-wait.

### 3. LogicMods Abstraction
Centralized path resolution in `UE4SSProgram::get_logic_mods_directory()` to avoid hardcoded strings and improve maintainability.

## Build Instructions (Linux)
```bash
export XWIN_DIR=~/.xwin
./tools/buildscripts/build.sh --toolchain xwin-clang-cl
```
