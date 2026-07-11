# Kinnector Docs

Kinnector Docs is the central documentation repository for architectural blueprints, developer manuals, rulebooks, and coding standards within the Kinnector project.

---

## Why this exists

A distributed security platform requires unified development guidelines, system blueprints, and policy compilation standards. Without a central reference, engineers write inconsistent code, and policy writers build incompatible schemas.

Kinnector Docs solves this by providing version-controlled technical references and developer specifications.

---

## Key Documentation Index

* **[CODE-RULEBOOK.md](file:///home/user/Documents/kinnector/kinnector-docs/CODE-RULEBOOK.md)**: Coding standards, formatting rules, Git branching practices, pull request workflows, and codebase structures.
* **[POLICY-MANAGEMENT.md](file:///home/user/Documents/kinnector/kinnector-docs/POLICY-MANAGEMENT.md)**: EDR policy parameters, audit mode logging, distribution rules, and FlatBuffers rules compiler instructions.
* **[PM.md](file:///home/user/Documents/kinnector/kinnector-docs/PM.md)**: Package manager supply chain security heuristics.

---

## Topics and Blueprints Covered

1. **Architecture & IPC Blueprints**: Core specifications detailing Unix domain socket communication and binary payload casting between the native collection layer (`kinnector-core`) and the analyzer daemon (`kinnector-agent`).
2. **Rule Authoring Guidelines**: Procedures for writing, testing, and cryptographically signing behavioral rules evaluated by `warden` and validated by `kinnector-config`.
3. **Deployment Specs**: Standards for packaging agents (DEB, RPM, MSI) and running sidecars in Docker environments.