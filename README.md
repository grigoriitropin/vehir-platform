# Vehir Platform

Vehir Platform is an experimental AI-native computing environment designed around agent–computer interaction. In the same way that conventional systems use human–computer interaction principles to make computing accessible to people, Vehir designs the entire software environment around the needs of AI agents.

Its interfaces and tooling are designed around structured, discoverable input and output that AI agents can use directly. Vehir explores how the software environment itself can be made more convenient for machine reasoning, rather than treating agent support only as an additional interface over existing workflows.

The current implementation provides the foundations of this environment, including structured agent tools, IPM packages, native compilation and loading, content-addressed generations, reconciliation, and supervised services. The platform is still under active development.

## System requirements

- Linux
- x86-64 processor
- 4 GB of RAM or more recommended

## Installation

Copy the following and send it to your AI agent:

```
Read the complete installation guide at
https://github.com/grigoriitropin/vehir-platform/blob/main/INSTALL.md
before taking any action. Follow it from beginning to end, and ask the
user whenever it requires a choice or information about the host system.
```

## Why AI-native

Vehir is built for machine-to-machine interaction. Instead of wrapping
existing human-facing tools with AI interfaces, it designs every layer
to be directly usable by agents:

- Structured input and output reduce reliance on free-form text parsing.
- Self-discoverable tools — the system tells the agent what it can do.
- Machine-readable errors with remediation hints.
- The source language (IPM) is designed for machine writing and verification, not manual editing.
- Declarative build — the agent declares desired state; the system reconciles.

Some features are still under development and may not yet be complete.

## Core systems

Vehir is more than an agent tool server. It combines several layers into
one self-hosting platform:

- **User-space microkernel and worker runtime.** The kernel launches and
  manages workers using arenas, shared memory, and doorbell-based
  coordination. Managed services use the same worker model.
- **Self-hosting native compiler.** IPM packages are compiled directly to
  x86-64 native code. The compiler builds itself, with fixpoint verification
  across generations.
- **Native loading.** Vehir resolves object closures, applies relocations,
  and loads its own compiled artifacts without relying on a conventional
  application runtime.
- **Content-addressed storage.** Sources and compiled objects are addressed
  by digest, deduplicated, and recorded in immutable generations.
- **Declarative reconciliation.** A declared build state and package graph
  are transformed into a verified generation, followed by an atomic
  `current` switch.
- **Code-derived agent tools.** MCP operations, request contracts, help, and
  request validation are derived from the tool implementations rather than
  maintained as separate handwritten interfaces.
- **Whole-universe enforcement.** Cross-package analysis checks structural
  and policy invariants before a generation is promoted.

## Source availability

This GitHub repository contains documentation and a convenience mirror of
the latest generation's IPM source tree. Release archives are the
authoritative distribution: every published release (`N.tar.gz`) contains
the complete IPM source tree together with native artifacts. The same
sources are mirrored in the repo under `sources-N/` for browsing without
downloading the archive, but the repo mirror may lag behind the latest
release.

## Project status

Vehir is experimental and under active development. Interfaces and
formats may change between releases. All source code is AI-generated.

## Documentation

- [INSTALL.md](INSTALL.md) — installation guide for AI agents
- [ARCHITECTURE.md](ARCHITECTURE.md) — technical architecture
- [LICENSE](LICENSE) — license

## Uninstallation

Before removing anything, identify the actual Vehir installation location
and ask the user to confirm it. Stop and remove the user service:

```bash
systemctl --user disable --now vehir-kernel.service
rm ~/.config/systemd/user/vehir-kernel.service
systemctl --user daemon-reload
```

Remove only the Vehir MCP entry from the agent client's configuration and
the persistent Vehir instruction added during installation, preserving all
unrelated configuration.

The installation directory is the complete Vehir content-addressed store.
It contains all generations, compiled objects, and source objects. After the
user confirms that none of this state is needed, remove the confirmed absolute
installation location:

```bash
rm -rf -- <confirmed-absolute-installation-location>
```
