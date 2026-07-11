# Kinnector Warden — Package Manager & Supply Chain Security (PM.md)

This document outlines the list of supported package managers (both application-level and system-level) and detail the mitigation logic used in **Kinnector Warden** to prevent supply chain attacks during installation context.

---

## 1. Supported Package Managers

The EDR Heuristics engine dynamically classifies the execution of any of the following package manager binaries as an **Installation Context** (`is_install_context`).

### Application-Level Package Managers & Build Tools
* **Node.js / JS / TS**: `npm`, `yarn`, `pnpm`, `bun`
* **Python**: `pip`, `poetry`, `pipenv`
* **PHP**: `composer`
* **Rust**: `cargo`
* **Ruby**: `gem`, `bundle`
* **.NET**: `nuget`, `dotnet`
* **Go**: `go`
* **Java / JVM**: `gradle`, `gradlew`, `mvn`, `sbt`
* **C / C++**: `conan`, `vcpkg`
* **Perl**: `cpan`, `cpanm`
* **Lua**: `luarocks`
* **Julia**: `julia`
* **Haskell**: `cabal`, `stack`
* **macOS / Linux Utilities**: `brew`

### System-Level Package Managers
* **Debian / Ubuntu**: `apt`, `apt-get`, `dpkg`
* **RHEL / CentOS / Fedora**: `yum`, `dnf`, `rpm`
* **Arch Linux**: `pacman`
* **Alpine Linux**: `apk`
* **Universal Linux Packages**: `snap`


---

## 2. Supply Chain Mitigation Logic

When a process is flagged in the **Installation Context**, the Warden daemon dynamically enforces three strict real-time mitigation rules to block malicious package scripts (e.g. `preinstall`/`postinstall` hooks) from executing exploits:

### Rule A: No Credential Reading (Exfiltration Prevention)
If an installation process (or any of its spawned child processes/scripts) attempts to read sensitive system credentials, the daemon immediately blocks the access and **terminates the process tree** (`SIGKILL`). 

Monitored sensitive credentials include:
* **SSH Keys**: Paths containing `/.ssh/`, `/id_rsa`, `/id_dsa`, `/id_ecdsa`, `/id_ed25519`
* **Docker Configurations**: `/.docker/config.json`
* **NPM Configurations**: `/.npmrc`
* **AWS Credentials**: `/.aws/credentials`
* **Git Configurations**: `/.gitconfig`, `/.git-credentials`
* **Python/PyPI Credentials**: `/.pypirc`, `/.pip/`
* **System Passwords**: `/etc/shadow`

*Event Triggered*: `Vulnerability.SupplyChain.CredentialAccessAttempt` (Remediation: `SIGKILL`)

### Rule B: No Persistence Mechanisms
Package installation scripts must never alter the host system's persistence files or services. If a process in the installation context writes or modifies paths in the following persistence categories, it is instantly **terminated** (`SIGKILL`):
* **Systemd Services**: `/etc/systemd/system`, `/usr/lib/systemd/system`
* **Cron Jobs**: `/etc/cron*`, `/var/spool/cron`
* **Shell Profiles**: `/etc/profile`, `/etc/profile.d`, `/etc/bash.bashrc`, `~/.bashrc`, `~/.profile`

*Event Triggered*: `Vulnerability.SupplyChain.PersistenceAttempt` (Remediation: `SIGKILL`)

### Rule C: Persistent Process Containment (Leftover Process Cleanup)
Post-install scripts often spawn detached background processes (e.g., reverse shell listeners, crypto miners, backdoors) that continue running after the main installation command completes. 

* **Tracking**: Warden monitors the life cycle of the top-level installation process (e.g. `npm install`). 
* **Eviction**: As soon as the top-level installer terminates, Warden scans for any orphaned background processes spawned within that install session.
* **Remediation**: All remaining background processes linked to that installation run-context are immediately **terminated** (`SIGKILL`).

*Event Triggered*: `Vulnerability.SupplyChain.PersistentProcess` (Remediation: `SIGKILL`)

---

## 3. Heuristic Verification & Alignment

The implementation in [heuristics.rs](file:///home/user/Documents/kinnector/warden/src/heuristics.rs) has been validated against all targets:
1. **Precise Binary Matching**: The classification does not use loose string matching (which causes false positives, e.g. matching `google-chrome` due to `go`). It checks for exact filename equality or ends-with boundary matches (e.g. `/usr/bin/go` matches exactly `go`).
2. **Instant Termination**: The `SIGKILL` signals are sent immediately to the offending PIDs, keeping the parent OS and package manager alive, but completely stopping the malicious execution.
3. **Open-Source JSON Alerts**: Structured JSON alert payloads are written directly to `/var/log/kinnector/alerts.log` and dispatched to webhooks.
