# Design Spec: Bug Fixes and Stability Improvements (Linux/Concurrency/Logic)

## Objective
Address several critical and serious bugs in RE-UE4SS related to Linux compatibility (recursion/symlinks), high CPU usage during pause (busy-wait), data races in hook management, and incorrect Lua table indexing.

## Background
The current implementation has several flaws that lead to:
- Crashes on Linux due to unhandled exceptions in directory iterators and infinite recursion via symlinks.
- 100% CPU usage of one core when the mod loader is paused.
- Silent failures in mods due to holes in Lua tables when iterating files.
- Potential Use-After-Free (UAF) when unregistering hooks.

## Technical Design

### 1. Robust Directory Iteration (Linux/Proton)
**Target:** `UE4SS/src/Mod/LuaMod.cpp` (`IterateGameDirectories` and `__files` handler)

**Problem:** `directory_iterator` construction can throw if permissions are denied or if the path is invalid, even if an `error_code` is provided (it only protects `operator++`).

**Fix:**
- Wrap constructor calls in `try-catch`.
- Explicitly use `std::filesystem::directory_options::skip_permission_denied`.
- Verify the path is a directory and not a symlink before iterating.

### 2. Async Thread Optimization (Busy-Wait)
**Target:** `UE4SS/src/Mod/LuaMod.cpp` (`LuaMod::update_async`) and `UE4SS/include/Mod/LuaMod.hpp`

**Problem:** The `update_async` loop uses `continue` when paused without sleeping, causing a spin-loop.

**Fix:**
- Change `m_pause_events_processing` to `std::atomic<bool>` in `LuaMod.hpp` and `UE4SSProgram.hpp`.
- Add a 5ms sleep inside the pause check block in `LuaMod::update_async`.

### 3. Lua Table Indexing Consistency
**Target:** `UE4SS/src/Mod/LuaMod.cpp` (`__files` handler)

**Problem:** The `index` counter increments for every item in a directory, but items are only added to the Lua table if they are NOT directories. This creates "holes" (nil values) in the resulting Lua table.

**Fix:**
- Move the `++index` increment inside the scope where the file is actually added to the table.

### 4. Hook Unregistration Safety (Mutex & UAF)
**Target:** `UE4SS/src/Mod/LuaMod.cpp` (`lua_unreal_script_function_hook_post`)

**Problem:** `std::erase_if` is called on a global vector without synchronization. Furthermore, the `lua_data` reference passed to the callback becomes dangling immediately after `erase_if`.

**Fix:**
- Add a static `std::mutex g_hook_data_mutex` to protect `g_hooked_script_function_data`.
- In `lua_unreal_script_function_hook_post`:
    1. Lock the mutex.
    2. **Explicit Data Capture:** Copy all required IDs and references from `lua_data` to local stack variables (e.g., `pre_cb_id`, `post_cb_id`, `cb_ref`, etc.) before any modification to the vector.
    3. Store the address of the target object: `void* const target_ptr = &lua_data;`.
    4. Perform `std::erase_if` using `target_ptr` as the search criteria.
    5. Unlock the mutex.
    6. Ensure `lua_data` is never accessed after step 4. Use only local copies for any logging or cleanup.

### 5. String Conversion & Quality
**Target:** `UE4SS/src/Mod/LuaMod.cpp` (`__files` handler)

**Problem:** Manual char-to-wchar cast loop corrupts non-ASCII paths. LogicMods path logic is hardcoded in a general handler.

**Fix:**
- Replace manual conversion with `RC::to_wstring()`.
- Expand the directory blacklist for Linux (`.snapshots`, `dev`, `run`, etc.).
- Add `get_logic_mods_directory()` to `UE4SSProgram` class to centralize motor-path logic.

## Architecture Changes
- **`UE4SSProgram`**: New public method `get_logic_mods_directory()`.
- **`LuaMod`**: Thread-safe event pause flag and optimized async loop.
- **Global Scope**: New mutex for hook data management.

## Verification Plan
### Automated Tests
- Since this is a C++ mod for a game engine, automated unit tests are limited. Verification will rely on:
    - **Compilation Check**: Ensure all changes compile across supported toolchains (xwin-clang-cl).
    - **Manual Log Review**: Verify that "Error processing directory entry" no longer floods the logs on Linux.
    - **Performance Check**: Monitor CPU usage when `m_pause_events_processing` is toggled.

### Manual Verification
- Test `ipairs()` on `__files` table in a directory containing subdirectories to ensure no `nil` gaps exist.
- Verify that non-ASCII filenames are correctly displayed and accessible in the Lua API.
