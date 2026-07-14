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

- Structured input and output — no free-form text parsing needed.
- Self-discoverable tools — the system tells the agent what it can do.
- Machine-readable errors with remediation hints.
- The source language (IPM) is designed for machine writing and verification, not manual editing.
- Declarative build — the agent declares desired state; the system reconciles.

Some features are still under development and may not yet be complete.

## Source availability

This GitHub repository contains documentation only. It does not contain
the Vehir source code.

The full IPM source tree is included inside every published generation
archive (N.tar.gz).

## Project status

Vehir is experimental and under active development. Interfaces and
formats may change between releases. Detailed limitations are listed in
[STATUS.md](STATUS.md).

## Documentation

- [INSTALL.md](INSTALL.md) — installation guide for AI agents
- [ARCHITECTURE.md](ARCHITECTURE.md) — technical architecture
- [STATUS.md](STATUS.md) — known limitations and current state
- [LICENSE](LICENSE) — license
