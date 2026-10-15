# Persistent Memory Protocol

You must maintain a persistent memory file named `memory.md` in the root directory. This file serves as your indexed, long-term memory to keep conversation history short and avoid redundant full-repo searches.

## Execution Rules:
1. **Bootstrap Phase:** At the very start of a session or when a new task/bug/question is introduced, your first action must be to read `memory.md` using your file-reading tool.
2. **Context Restriction:** Use the indexes inside `memory.md` to pinpoint specific files. Do not run grep or full-directory listings if the memory file tells you exactly where the feature or past bug lives.
3. **The Commit Gate:** Before you declare a task finished, you *must* update `memory.md` to reflect architectural changes, new features, fixed bugs, or resolved questions.
4. **No Chat Bloat:** Assume the user will clear the CLI conversation history frequently. Rely entirely on `memory.md` for project continuity across sessions.

## Project Guidelines & Operational Rules:
- **Build Target**: Always compile in `Release` configuration (`cmake --build build --config Release`) unless debugging specific memory faults.
- **Local Deployment**: The active Steam client is located at `C:\Program Files (x86)\Steam`. Always deploy built DLLs and PDBs directly to this directory for testing.
- **Crash Dump Analysis**: When Steam terminates unexpectedly, inspect minidumps under `C:\Program Files (x86)\Steam\dumps\crash_steam.exe_*.dmp`.
- **x64 ABI Integrity**: Valve's 64-bit Steam client uses polymorphic objects for internal packet structures (e.g., `CNetPacket` requires `vptr` at offset `0x00`). Always verify structure alignment against disassembly before modifying packet hooks.
- **Mirror Resiliency**: Upstream metadata repos often lag behind weekly Steam client updates. Always maintain community mirror fallback chains (`madoiscool`, `lua.tools`) in `RemoteToml.cpp` and `opensteamtool.toml`.