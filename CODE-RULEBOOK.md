<!--
Copyright (c) 2026 Albert Mirakhmedov. All rights reserved.
Licensed under the PolyForm Noncommercial License 1.0.0.
See LICENSE file in the project root for details.
-->

# Kinnector Coding Rules & Codebase Guidelines (CODE-RULEBOOK.md)

This document establishes the official coding standards, architectural constraints, and development practices for the **Kinnector** project. All contributors and AI assistants must strictly adhere to these rules when writing or modifying code.

---

## 1. Names, Paths, and Executables Agnosticism (Agnostic Code)

### Principle
We enforce strict agnosticism regarding file system paths, process names, executable names, registry keys, and other system identifiers. **Do not hardcode specific names or paths in detection logic, utility functions, or filtering mechanisms.** 

Hardcoding specific OS processes or paths limits portability, makes the codebase fragile, and invites bypasses.

### Guidelines
* **No Hardcoded Process/Binary Lists**: Do not hardcode lists of system processes (e.g., `svchost.exe`, `csrss.exe`, `lsass.exe`, `explorer.exe`).
* **Dynamic Resolution**: If you need to verify whether a process is a legitimate OS process, query the system dynamically. For example, discover the standard system paths (via APIs like `GetSystemDirectory` or environmental variables) and locate/verify binaries dynamically at startup or run-time.
* **Consent on Embedding**: If embedding static resource names, configuration keys, or default paths is absolutely unavoidable, you **must ask the user for explicit permission** beforehand, or redesign the logic to use a configuration-driven or dynamic approach.

### Anti-Pattern Example (DO NOT DO)
```cpp
// BAD: Hardcoded list of system processes
bool IsOSProcess(const std::wstring& processName) {
    std::wstring lower = processName;
    std::transform(lower.begin(), lower.end(), lower.begin(), ::towlower);
    return (lower == L"svchost.exe" ||
            lower == L"csrss.exe" ||
            lower == L"lsass.exe" ||
            lower == L"services.exe" ||
            lower == L"system");
}
```

### Preferred Pattern Example
```cpp
// GOOD: Dynamic resolution at startup, zero-allocation lookups in telemetry callbacks.
// Avoids heap allocation, std::wstring copies, and dynamic object initialization in the hot path.
namespace SystemBinaryResolver {
    // Array of pre-resolved system binaries, populated once at startup
    extern std::wstring_view* g_SystemBinaries; 
    extern size_t g_SystemBinariesCount;

    // Called ONCE at startup (outside the telemetry callback hot path)
    void Initialize(); 

    // Called on telemetry hot path: Zero allocations, inlineable binary search
    inline bool IsLegitimateSystemProcess(std::wstring_view processPath) noexcept {
        return std::binary_search(g_SystemBinaries, g_SystemBinaries + g_SystemBinariesCount, processPath);
    }
}
```

---

## 2. Platform-Specific Directory and Code Isolation

### Principle
Kinnector supports cross-platform components or may be ported/structured for multiple OS targets. Platform-specific code must be strictly isolated to dedicated directories or modules. Do not mix Windows-specific code into generic cross-platform code.

### Guidelines
* **Directory Structure**: If a directory is dedicated to a specific operating system (e.g., `windows`, `linux`, `macos`), its contents must strictly use the APIs, naming conventions, and libraries characteristic of that platform.
* **Naming Conventions**:
  * Platform-specific modules or source files should contain appropriate suffixes (e.g., `_win.cpp`, `_linux.cpp`, or Rust modules nested under `#[cfg(target_os = "windows")]`).
  * Follow the idiomatic casing and code structure of the host language/platform in these files.
* **API Separation**: Cross-platform code should interact with platform-specific code solely through clean interfaces (abstract base classes in C++, traits in Rust, or abstraction layers).

---

## 3. Performance-First Telemetry Architecture (Core Module Constraints)

### Principle
The telemetry library (`kinnect-core`) operates inside the Windows kernel ETW callback threads. Latency in these callbacks blocks kernel telemetry dispatch, leading to event loss, system instability, or visible CPU overhead for the user. Therefore, **performance is our highest priority**. Any structural or library overhead on the hot path is strictly restricted.

### Guidelines
* **Zero Allocations in the Hot Path**: 
  * Never invoke `malloc`, `new`, or standard library structures that trigger dynamic memory allocation (e.g., resizing `std::vector`, constructing `std::wstring`/`std::string`, or using dynamic node-based maps like `std::unordered_map` without custom pre-allocated buckets) inside ETW callbacks.
  * Use `std::wstring_view` or raw character pointers to reference paths/names rather than copying them.
* **Restrict Object-Oriented & Vtable Overhead**:
  * Avoid deep inheritance hierarchies and virtual function table (`vtable`) lookups in the callback loop. 
  * Prefer inline functions, static templates, or simple namespaces with flat functions.
* **Pre-Allocate & Cache at Startup**:
  * Any complex initialization, file-system lookup, environment resolving, or memory layout sizing must occur once during startup/initialization.
  * Store results in contiguous buffers (arrays or pre-allocated pools) optimized for zero-allocation cache-friendly lookups.
* **FFI Callback Safety**:
  * Keep C-compatible FFI callbacks between `kinnect-core` and the Rust agent (`kinnect-agent`) extremely lightweight. 
  * Instantly copy necessary raw telemetry data into a thread-safe, lock-free ring buffer/channel and return control to the operating system immediately. Do not perform evaluation or heavy IO inside the telemetry thread.

---

## 4. Cross-Platform Code Readability, Language, and OS-Specific Rules

### Principle
To support robust engineering across multiple host development environments, all code must remain clean, legible, and compatible regardless of the platform where it is written or built. Platform-specific execution must strictly align with native detection boundaries defined in the [detection-engine](file:///home/user/Documents/kinnector/kinnector-context/detection-engine/).

### Guidelines

* **Code Readability & Cross-Platform Compatibility**:
  * **Encoding & Line Endings**: All source code files must be saved with UTF-8 encoding and standard LF (`\n`) line endings.
  * **Path Representation**: In shared, non-platform-isolated modules, always use the forward slash (`/`) as the directory separator, or leverage cross-platform abstractions (such as C++ `std::filesystem::path` or Rust `std::path::Path`). Never hardcode backslash (`\`) characters in common source code.
  * **Fixed-Width Types**: Use standard fixed-width integer types (e.g., `uint32_t`, `int64_t` from `<cstdint>` in C++, and native primitive types in Rust) inside telemetry messages, configurations, and FFI mappings to ensure size consistency across compilers and target architectures.

* **Language-Specific Telemetry Rules**:
  * **C++ (`kinnect-core`)**: Adhere to C++20/C++23 standards. Avoid manual raw pointer resource tracking; manage allocations using standard smart pointers (`std::unique_ptr` and `std::shared_ptr`) outside the telemetry callback path. Structs exchanged across FFI must utilize explicit alignment structures (e.g., `#pragma pack(push, 1)`) to avoid layout packing bugs.
  * **Rust (`kinnect-agent`)**: Maintain a clean, Clippy-compliant codebase. In production telemetry workers, never use panicking macros such as `unwrap()` or `expect()`; propagate errors cleanly through `Result` or `Option` types. Limit the use of `unsafe` blocks to FFI boundary validation.

* **Pattern Matching Engines**:
  * **YARA & Sigma Matching**: All rule evaluation engines must operate asynchronously inside the agent to safeguard core performance. Follow file hashing caches, memory boundaries, and Directed Acyclic Graph (DAG) structures specified in [yara.md](file:///home/user/Documents/kinnector/kinnector-context/detection-engine/yara.md) and [sigma.md](file:///home/user/Documents/kinnector/kinnector-context/detection-engine/sigma.md).

* **OS-Specific Rules (Detection Engine Alignment)**:
  * **Windows**: Telemetry pipelines must consume ETW and minifilter driver logs. Enforce the direct rule-based prevention model (Rule 1: Credential/Wallet Access Ownership & Tamper Check; Rule 2: PKM Isolation with 2-second detached process killer; Rule 3: Clipboard Hijack Prevention; Rule 4: Same-Vendor updates). Validate the parent process ID by comparing the reported PPID against `RealParentPid`. BYOVD driver checks are explicitly out-of-scope for the Windows client to avoid system false positives.
  * **macOS**: Target macOS-specific file-driven threats (Keychain `.keychain-db` extraction, browser databases) using Endpoint Security Framework (ESF) synchronous hooks (`ES_EVENT_TYPE_AUTH_*`) and auditpipe fallback. Use `audit_token_t` for process validation. Background pasteboard checking is prohibited.
  * **Linux**: Integrate with BPF LSM hooks (`security_file_open`, `security_socket_connect`) for inline access blocking. Support container awareness via mount/PID namespaces. Keep process containment strictly centered around tree suspension (`SIGSTOP`); do not issue `SIGKILL` automatically to avoid file corruption and package manager lockouts.
  * **Hybrid Trust Pipeline**: On Linux, implement dynamic package database queries (`dpkg -V`, `rpm -V`, `pacman -Qk`), squashfs read-only loop mounts validation, and `InstallContext` tracking to auto-trust package upgrades and minimize user interruption.

---

## 5. Dynamic Configurations & Business Logic Isolation

### Principle
Keep commercial features, subscription checks, and deployment orchestration entirely separate from the core telemetry collection and heuristic detection loops. Do not hardcode lists of detection files, wallets, browser targets, or exfiltration criteria in code.

### Guidelines

* **No Hardcoded Configurations**:
  * Never hardcode lists of target browser files, sensitive folders, executable allowlists, or threat signature categories.
  * All telemetry evaluation matrices, exclusion files, and detection flags must be read dynamically from structured configurations.

* **Fallback Configuration Loading Chain**:
  * Implement the following configuration parsing cascade at startup:
    1. **Remote Cloud API**: Retrieve updated rule databases over authenticated TLS connections (Paid instances only).
    2. **Local Configuration overrides**: Merge rules from host-local config paths (Linux/macOS `/etc/kinnector/rules.json`, Windows `%ProgramData%\Kinnector\rules.json`) if present.
    3. **Compile-Time Defaults**: Use embedded static configurations (via `include_str!` or compiler-linked assets) solely as a final fallback.

* **Paid Feature Rule Isolation**:
  * **Strict In-Memory Storage**: Security rules, proprietary YARA signatures, and real-time feeds downloaded from the cloud API must be stored strictly in-memory and never written to disk, preventing indicator extraction.
  * **Atomic Hot-Reloading**: Apply lock-free atomic pointers (like `ArcSwap` in Rust) to swap active configuration references dynamically without interrupting detection workers.

* **Auditing and Alert Routing**:
  * **Open Source (OSS)**: Write local, standard structured JSON events to `/var/log/kinnector/audit.log` or `%ProgramData%\Kinnector\logs\audit.log` without cloud dependencies. Support local webhook routing (Slack, Discord, Telegram) directly from the endpoint.
  * **Paid (Enterprise)**: Stream compressed (`zstd`), buffered telemetry over gRPC channels. Buffer low-priority telemetry locally in a circular, in-memory ring buffer (e.g., 50 MB), flushing it on-demand to the cloud only when triggered by security events.

* **Monetizable Cloud Boundaries**:
  * Cloud dashboard capabilities, email/session credential leak scanning, crowdsourced binary reputation lookups, and console-initiated host isolation/quarantine are out-of-scope for the local OSS agent. Keep these modules isolated behind mockable network services.
