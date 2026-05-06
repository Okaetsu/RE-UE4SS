# Bug Fixes and Stability Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix critical recursion bugs, high CPU usage, data races, and indexing errors to improve stability on Linux and general robustness.

**Architecture:** Surgical application of standard C++ synchronization primitives (atomic, mutex), hardening of filesystem iterators with error handling, and refactoring path logic into the core program class.

**Tech Stack:** C++, std::filesystem, std::atomic, std::mutex, LuaMadeSimple.

---

### Task 1: Filesystem Robustness (Iterators & Blacklist)

**Files:**
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Harden IterateGameDirectories iterator**
Wrap the `directory_iterator` construction in a try-catch and ensure `skip_permission_denied` is used.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 2629
try {
    for (const auto& item : std::filesystem::directory_iterator(directory, std::filesystem::directory_options::skip_permission_denied, ec)) {
        // ... existing logic ...
    }
} catch (const std::exception& e) {
    Output::send<LogLevel::Error>(STR("Exception constructing directory_iterator for {}: {}\n"), directory.wstring(), to_wstring(e.what()));
}
```

- [ ] **Step 2: Expand Linux directory blacklist**
Add common system directories that can cause issues or are irrelevant for modding.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 2647
if (path_str == "dosdevices" || path_str == "drive_c" || path_str == "proc" || path_str == "sys" || 
    path_str == ".snapshots" || path_str == "dev" || path_str == "run" || path_str == "tmp" || 
    path_str == "boot" || path_str == "lost+found")
{
    continue;
}
```

- [ ] **Step 3: Harden __files handler iterator**
Apply same robustness to the file listing handler.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 2799
try {
    for (const auto& item : std::filesystem::directory_iterator(path_wstr, std::filesystem::directory_options::skip_permission_denied, ec)) {
        // ...
    }
} catch (const std::exception& e) {
    Output::send<LogLevel::Error>(STR("Exception constructing file iterator for {}: {}\n"), path_wstr, to_wstring(e.what()));
}
```

- [ ] **Step 4: Commit**
`git commit -m "fix(linux): harden directory iterators and expand blacklist"`

---

### Task 2: Fix Lua Table Indexing in __files

**Files:**
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Move index increment**
Ensure `index` only increments when a file is actually added to the table, avoiding gaps.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 2817
if (!item.is_directory())
{
    files_table.add_key(index);
    // ... setup file_table ...
    files_table.fuse_pair();
    ++index; // Move here from outside the if
}
```

- [ ] **Step 2: Commit**
`git commit -m "fix(lua): avoid nil gaps in __files table by only incrementing index for files"`

---

### Task 3: Thread-safe Event Pause (Atomic + Sleep)

**Files:**
- Modify: `UE4SS/include/Mod/LuaMod.hpp`
- Modify: `UE4SS/include/UE4SSProgram.hpp`
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Update headers to use std::atomic**
Change `m_pause_events_processing` to `std::atomic<bool>`.

```cpp
// UE4SS/include/Mod/LuaMod.hpp
#include <atomic>
// ...
std::atomic<bool> m_pause_events_processing{};

// UE4SS/include/UE4SSProgram.hpp
#include <atomic>
// ...
std::atomic<bool> m_pause_events_processing{};
```

- [ ] **Step 2: Fix busy-wait in update_async**
Add sleep to prevent 100% CPU usage when paused.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 6458
if (m_pause_events_processing)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(5));
    continue;
}
```

- [ ] **Step 3: Commit**
`git commit -m "perf: fix busy-wait in update_async and use atomic for pause flag"`

---

### Task 4: Thread-safe Hook Unregistration

**Files:**
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Add mutex and capture logic**
Implement the safe unregistration pattern with a mutex and local copies.

```cpp
// UE4SS/src/Mod/LuaMod.cpp
static std::mutex g_hook_data_mutex; // Global or static member

// Inside lua_unreal_script_function_hook_post callback lambda remove_if_scheduled:
auto remove_if_scheduled = [&] -> bool {
    if (lua_data.scheduled_for_removal)
    {
        std::lock_guard<std::mutex> guard(g_hook_data_mutex);
        
        // Capture everything before erasure
        const auto pre_id = lua_data.pre_callback_id;
        const auto post_id = lua_data.post_callback_id;
        const auto cb_ref = lua_data.lua_callback_ref;
        const auto post_cb_ref = lua_data.lua_post_callback_ref;
        const auto thread_ref = lua_data.lua_thread_ref;
        void* const target_ptr = &lua_data;

        // Perform unregistration using captured IDs/refs
        lua_data.unreal_function->UnregisterHook(pre_id);
        luaL_unref(lua_data.lua.get_lua_state(), LUA_REGISTRYINDEX, cb_ref);
        // ... repeat for post and thread ...

        std::erase_if(g_hooked_script_function_data, [&](const auto& elem) {
            return elem.get() == target_ptr;
        });
        return true;
    }
    return false;
};
```

- [ ] **Step 2: Commit**
`git commit -m "fix(hook): use mutex and safe capture to avoid UAF during hook unregistration"`

---

### Task 5: Robust String Conversion

**Files:**
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Replace manual conversion loop**
Use `RC::to_wstring` which handles UTF-8 correctly.

```cpp
// UE4SS/src/Mod/LuaMod.cpp around line 2744
try
{
    path_wstr = RC::to_wstring(path_str);
}
catch (const std::exception& e)
{
    Output::send<LogLevel::Error>(STR("Failed to convert path to wstring: {}\n"), to_wstring(e.what()));
    // Fallback or handle error
}
```

- [ ] **Step 2: Commit**
`git commit -m "fix(string): use RC::to_wstring for robust UTF-8 to wide string conversion"`

---

### Task 6: UE4SSProgram Path Abstraction

**Files:**
- Modify: `UE4SS/include/UE4SSProgram.hpp`
- Modify: `UE4SS/src/UE4SSProgram.cpp`
- Modify: `UE4SS/src/Mod/LuaMod.cpp`

- [ ] **Step 1: Add get_logic_mods_directory to UE4SSProgram**
Implement the centralized logic for locating LogicMods.

```cpp
// UE4SS/include/UE4SSProgram.hpp
RC_UE4SS_API auto get_logic_mods_directory() -> std::filesystem::path;

// UE4SS/src/UE4SSProgram.cpp
auto UE4SSProgram::get_logic_mods_directory() -> std::filesystem::path {
    // Navigate from m_game_executable_directory to Content/Paks/LogicMods
    // ... implementation ...
}
```

- [ ] **Step 2: Refactor LuaMod.cpp to use the new method**
Replace the hardcoded "LogicMods" string search and path reconstruction.

```cpp
// UE4SS/src/Mod/LuaMod.cpp
path_wstr = UE4SSProgram::get_program().get_logic_mods_directory().wstring();
```

- [ ] **Step 3: Commit**
`git commit -m "refactor: centralize LogicMods path logic in UE4SSProgram"`
