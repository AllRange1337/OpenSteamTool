# OpenSteamTool Persistent Memory

## 1. Project Directory & Environment
- **Repository Root**: `D:\Development\git-Projects\OpenSteamTool`
- **Fork Remote**: `origin` -> `https://github.com/AllRange1337/OpenSteamTool.git`
- **Upstream Remote**: `upstream` -> `https://github.com/OpenSteam001/OpenSteamTool.git`
- **Steam Installation**: `C:\Program Files (x86)\Steam`
- **Config File**: `C:\Program Files (x86)\Steam\opensteamtool.toml`
- **Local Cache Directory**: `C:\Program Files (x86)\Steam\opensteamtool\`
  - `pattern\steamclient\<sha256>.toml`
  - `pattern\steamui\<sha256>.toml`
  - `ipc\steamclient\<sha256>.toml`

---

## 2. Architecture & File Index
- **`src/Steam/Structs.h`**:
  - Defines binary layout for internal Steam objects: `CNetPacket`, `IKeyValuesSystem`, `MsgHdr`, etc.
  - **CRITICAL x64 ABI NOTE**: `CNetPacket` is a polymorphic C++ object in 64-bit Steam.
    - `0x00`: `vptr` (`void* m_pVTable`)
    - `0x08`: `HCONNECTION m_hConnection` (8 bytes, value `-1` / `0xFFFFFFFF`)
    - `0x10`: `uint8* m_pubData` (pointer to packet payload)
    - `0x18`: `uint32 m_cubData` (payload length)
    - If `m_pVTable` is missing, `m_pubData` reads `0xFFFFFFFF`, causing an instant `0xC0000005` access violation upon packet arrival.
- **`src/Hook/Hooks_NetPacket.cpp`**:
  - Intercepts Steam networking packets (`RecvPkt` and `BBuildAndAsyncSendFrame`).
  - Contains `UnpackRaw` for unpacking protobuf / raw message envelopes. Includes defensive bounds and pointer validation.
- **`src/Utils/SteamMetadata/RemoteToml.cpp`**:
  - Handles remote fetching of pattern and IPC TOML specifications when Steam updates.
  - Multi-mirror fallback chain:
    1. Custom mirror template from `opensteamtool.toml` (if configured)
    2. Upstream GitHub: `OpenSteam001/steam-monitor`
    3. Upstream jsDelivr CDN
    4. Community GitHub: `madoiscool/steam-monitor`
    5. Community jsDelivr CDN
    6. Community Git: `git.lua.tools/luatools/steam-monitor`
  - Does NOT abort on HTTP 404 so downstream community mirrors can supply signatures when upstream lags behind.
- **`src/Utils/SteamMetadata/PatternLoader.cpp`**:
  - Computes SHA-256 of `steamclient64.dll` and `steamui.dll`, checks local cache, and fetches remote TOML.
- **`src/Utils/SteamMetadata/IPCLoader.cpp`**:
  - Computes SHA-256 of `steamclient64.dll`, checks local IPC cache, and fetches remote TOML.
- **`src/dwmapi/dwmapi.cpp` & `src/xinput1_4/`**:
  - DLL hijacking proxy wrappers placed in `C:\Program Files (x86)\Steam\` that load `OpenSteamTool.dll` upon `steam.exe` startup.
- **`src/dllmain.cpp`**:
  - Spawns detached worker thread `InitThread` to avoid executing file I/O or hooking inside `DllMain` (loader lock).

---

## 3. History of Critical Issues & Fixes

### A. Missing Signatures & IPC Spec on Steam Updates (Issue #209)
- **Symptom**: Three popups on startup: "signature file not found for steamui.dll", "signature file not found for steamclient64.dll", "IPC spec file not found".
- **Root Cause**: Official `OpenSteam001/steam-monitor` does not release TOMLs immediately when Valve updates Steam.
- **Fix**:
  - Configured community mirrors in `opensteamtool.toml` and built-in fallback chain in `RemoteToml.cpp`.
  - Removed early `break;` on HTTP 404 in `RemoteToml.cpp`.

### B. Instant Silent Crash on Start / Home Page Load (`RecvPkt` 0x5BC990)
- **Context**: Reported by `jelinet` in `issuecomment-6174921564`. Jelinet noted disabling `RecvPkt` stopped the crash.
- **Third-party Zip**: Comments `6174725589` and `6177072812` discussed `popup_fix_v1.zip`. **Never use it**; it introduces broken unreleased header dependencies (`Hooks_Branches.h`).
- **Root Cause**:
  `RecvPkt` hook target `0x5BC990` was correct, but `struct CNetPacket` in `src/Steam/Structs.h` lacked the virtual method table pointer (`vptr`) at offset 0 on x64. `m_pubData` loaded `0xFFFFFFFF` (`m_hConnection`), leading to a crash inside `UnpackRaw` when dereferencing the payload.
- **Fix**:
  - Added `#if defined(_WIN64) || defined(__x86_64__) void* m_pVTable; #endif` in `src/Steam/Structs.h`.
  - Added pointer sanity guards (`reinterpret_cast<uintptr_t>(data) < 0x10000 || == ~0ULL`) in `Hooks_NetPacket.cpp`.

---

## 4. Build, Deployment, and Testing Runbook
- **Build**:
  ```cmd
  cmake --build build --config Release
  ```
- **Deploy**:
  ```powershell
  Copy-Item "build\Release\OpenSteamTool.dll" "C:\Program Files (x86)\Steam\OpenSteamTool.dll" -Force
  Copy-Item "build\Release\OpenSteamTool.pdb" "C:\Program Files (x86)\Steam\OpenSteamTool.pdb" -Force
  ```
- **Launch Test**:
  When testing Steam startup from CLI runners, decouple from Job Objects using WMI or start detached to prevent termination on script exit:
  ```powershell
  Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine = '"C:\Program Files (x86)\Steam\steam.exe"'}
  ```
