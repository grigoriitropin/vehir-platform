# ARCHITECTURE.md

## 1. Goals and design model

### Problem

Operating systems and development tools are designed for people.
They produce free-form text, rely on learned conventions, and
expect manual intervention when something fails. An AI agent
needs interfaces it can parse, actions it can predict the outcome
of, and errors it can act on.

### Decision

Vehir is built for agents to use directly. Its interfaces are
structured and machine-readable — not free-form text meant for a
human reading a terminal.

Rules that hold across the system:

- **Declarative over imperative.** The agent declares desired state;
  the reconciler builds it.
- **Immutable generations.** Every successful reconcile produces a new generation.
  Build outputs are assembled in a new generation instead of modifying the
  active generation in place. After successful verification, `current`
  switches atomically.
- **Everything is a package.** Every capability, tool, and library is
  a named IPM package in a dependency graph.
- **Structured I/O.** Every tool has a generated request contract.
  Errors carry machine-readable codes and remediation hints.

### Current behavior

- A single MCP entry point (vehir-shell) exposes the system's
  capabilities. The set of available tools is discoverable at
  runtime.
- The build process is declared: artifacts are registered in a
  build state, the reconciler computes the dependency closure,
  compiles, and produces a new immutable generation.
- The source tree contains IPM packages covering the compiler, build
  system, runtime, tools, service supervision, experiments, and legacy
  components. The active build closure is derived from the declared
  artifacts and their dependencies for each generation.
- The compiler compiles itself. Running the reconciler multiple
  times with unchanged sources produces an identical toolchain —
  verified through fixpoint checks.
- Content-addressed storage deduplicates objects by digest.
  Writes are atomic (write-to-temp-then-rename).
- The runtime is a user-space process: loader → kernel →
  supervisor → managed services. All components run as the
  same user, without root.

### Enforced invariant

- Before dispatch, the boundary checks the requested operation and parameter
  names against a contract derived from the tool's code. Unknown operations
  and unknown fields are rejected. Value types and operation-specific
  constraints are validated by the tool itself.
- Error responses must reference the centralized error catalog.
- Verification checks run before promotion. Some are enforced
  as hard failures; others start as warnings and move to hard
  failures as the enforcement threshold tightens over time.

### Known limitation

- Linux x86-64 only.
- Tool request contracts are derived from code analysis but the derivation
  does not yet recognize all parameter forms.
- Installed generations contain absolute paths from the build
  machine and require manual correction after extraction.
- Some tools (e.g. the directory tree renderer) may crash on
  large inputs or edge cases.
- Not all tools have been migrated to the MCP launch contract.
  Some still write errors to stdout; checking journalctl may
  be necessary to see what happened.
- Many functions are stubbed out and return placeholder results
  because their implementation is not yet complete.
- Memory isolation between components and the full reclaim ring
  model for arena chunk recycling are still in progress.
- Tool timeouts or crashes can bring down the kernel due to
  incomplete memory isolation and lack of stronger kernel-side
  error recovery.
- The journal system does not yet cover all use cases and lacks
  convenient tools for querying and filtering.
- Many tools are at MVP level — some have stubs, others are
  awaiting migration to the current tool contract, and edge
  case handling is often incomplete.
- No multi-core execution. Everything runs on a single core
  despite the worker infrastructure being present.
- Arena chunk reuse is not fully implemented. Under frequent
  reconciles, memory usage grows over time and a kernel
  restart may be needed to reclaim it.
- The source tree carries significant technical debt: inconsistent
  patterns across packages, legacy code paths, and lack of
  standardization. These will be addressed through automated
  enforcement checks.
- The compiler, tools, and enforcers share a common backend but
  its unification is not yet complete.

## 2. Agent–computer interaction

### Problem

Conventional tools are built for humans: they output free-form text,
hide their capabilities behind man pages and `--help` flags, and
signal failure through exit codes that carry no explanation. It is
often unclear whether an empty output means success with nothing to
report or a silent failure. An agent must be told what each tool
can do before using it — and even then, parsing human-readable
output is error-prone.

### Decision

Vehir tools are structured end-to-end:

- **Self-discoverable.** A help tool returns every tool's name and
  request contract. An agent can enumerate what the system can do
  without reading external documentation.
- **Structured input.** Every tool exposes a request contract derived
  from its own code. The agent sends structured objects,
  not command-line strings.
- **Structured output.** Tools return JSON documents. Errors are
  not exit codes — they are JSON objects with a machine-readable
  code, a message, and a remediation hint.
- **No silent success.** Every tool call returns an explicit
  result. Empty output is a defect.

### Current behavior

- The MCP entry point exposes a single `vehir-shell` tool. All
  available operations are dispatched through it by name.
- `help` with `{help: null}` returns the list of available tool names.
  Requesting help for a specific tool returns that tool's generated request
  contract.
- The boundary validates the requested operation and parameter names before
  the tool runs. Value types and operation-specific constraints remain the
  responsibility of the tool.
- Errors reference the centralized catalog. Each error includes
  a code, a human-readable message, and a remediation string.
- Some tools still use the legacy argv/stdout contract and are not
  reachable through MCP. Their migration is in progress.

### Enforced invariant

- Every tool request must carry a recognized operation identifier.
  Unknown operations are rejected before any work begins.
- Request validation is not advisory — unknown parameters cause
  the request to fail, not silently ignored.
- Error responses must use a code from the error catalog. Ad-hoc
  error strings are rejected by the error reporting mechanism.

### Known limitation

- Request-contract derivation does not yet infer every value shape correctly.
  For example, a field used as an array may be described as a string. The
  boundary validates field names, while the tool remains responsible for
  validating value shapes.

## 3. Packages and IPM

Vehir code is organized as IPM packages. Each package is a directory
containing an `.ipm` source document. The same representation is used
throughout the compiler, runtime, build system, services, and tools,
allowing the build system and analyzers to process them uniformly.

## 4. Declared build state

### Problem

A build system needs to know what to build. In conventional
systems this is split across Makefiles, build scripts, and
implicit conventions. For an agent-driven system, there must be
a single place that declares what artifacts should exist — and
that declaration must be machine-readable and machine-writable.

### Decision

`declared-build-state.json` is the authoritative declaration of the
requested top-level artifacts, compiler target, and source roots. The
reconciler combines it with the source graph, current generation, build
policies, and content-addressed store to produce the next generation.
Dependencies do not need to be listed as top-level artifacts; they are
derived from the package graph.

### Current behavior

- The file lives at `etc/reconcile/reconcile.d/declared-build-state.json`.
- Agents modify it through the standard file editing tools.
- The dedicated `register-artifact-inside-declared-build-state`
  tool is stubbed and not yet functional.

### Enforced invariant

- Every top-level artifact included in a generation must originate from the
  declared build state. Its transitive dependencies are derived automatically.

### Known limitation

- The declared state contains absolute paths from the build
  machine and must be corrected after extraction.

## 5. Native compiler

### Problem

Vehir is written in its own source language (IPM). To run on
real hardware, IPM source must be translated to x86-64 machine
code and packed into ELF binaries that Linux can execute.

### Decision

Vehir has its own native compiler that translates IPM source
directly to x86-64 machine code:

1. IPM packages are parsed into an internal representation.
2. The lowering pass translates IPM instructions into native
   operations (register moves, memory accesses, syscall
   invocations).
3. Native objects are emitted and stored in the objlib.
4. An ELF image is assembled from the objects.

### Current behavior

- The compiler is itself an IPM package
  (`compile-process-starting-stdin-inside-native-elf-image`).
- Compilation happens during reconcile, not as a separate step.
- The compiler processes one function at a time from a work
  item queue.
- Fixpoint verification confirms that compiling the compiler
  with itself produces identical binaries across successive
  generations.

### Enforced invariant

- The compiler target in the declared build state must point to
  a valid compiler package. Without it, the reconciler cannot
  build anything.

### Known limitation

- The compiler can crash on certain inputs.

## 6. Compiler-backed validation

### Problem

There must be a single authority on what IPM instructions exist
and how they behave. If this information is maintained separately
from the compiler, the two drift apart — documentation, request contracts,
and analyzers end up describing a different language than what
actually compiles.

### Decision

The compiler backend defines what instructions exist and how
they work. Other parts of the system — tool help, request-contract
derivation, enforcers — are being moved to query the compiler
for this information rather than maintaining separate
definition lists. The goal is that the compiler is the only
place an instruction is defined.

### Current behavior

- The compiler defines instructions. Other consumers still use
  separate lists; unification is in progress.

## 7. Content-addressed storage

### Problem

Filesystem paths are fragile. Two identical files stored in
different places waste space and create ambiguity about which
copy is authoritative. Renames break references. For a build
system that produces objects from source, deduplication and
content integrity matter — you need to know that the object
you load matches the source it was built from.

### Decision

Content is addressed by SHA-256 digest. Sources and build
artifacts are stored in CAS directories, referenced by
digest rather than path. Writes are atomic
(write-to-temp-then-rename).

### Current behavior

- Source packages are stored as IPM files in CAS. The
  `sources/` tree contains symlinks into CAS.
- Build artifacts land in CAS by digest and are loaded
  through symlinks in `bin/` and `libexec/`.

### Enforced invariant

- Every stored object has a verifiable digest.

## 8. Immutable generations

### Problem

Mutating a running system in place is risky. A failed update
can leave the system in an inconsistent state — half new code,
half old, with no way back. For a self-compiling system, this
is fatal: if the compiler breaks during an in-place update,
the system can no longer rebuild itself.

### Decision

Every successful reconcile produces a new, complete generation directory
under `generations/`. The previous generation is untouched.
On success, the `current` symlink switches atomically to the
new generation. If anything fails, `current` stays on the
last known-good generation.

### Current behavior

- Generations are numbered. The reconciler finds the highest
  number to determine the baseline.
- Promotion records a content-addressed snapshot of the source state,
  assembles and verifies the generation, then flips `current`.
- The rollback tool is stubbed. To revert to a previous
  generation, `current` must be repointed manually with `ln`.

### Enforced invariant

- A generation that fails verification is never promoted to
  `current`.

### Known limitation

- Automatic rollback on runtime failure is not implemented.
  Recovery requires manual `ln` of `current` to a previous
  generation.

## 9. Reconciliation

### Problem

Building a self-compiling system is not a linear compile step.
The system must: compute what changed, build new artifacts in
dependency order, verify the result against enforced checks,
and switch to the new output without corrupting the running
state. Doing this by hand is error-prone; it needs to be a
single deterministic operation.

### Decision

Reconciliation is the single operation that turns declared
state into a new generation:

1. Read the declared build state.
2. Compute the dependency closure across all referenced
   packages.
3. Compile artifacts.
4. Run verification checks.
5. If all checks pass, promote and flip `current`.

### Current behavior

- Reconciliation is triggered through the MCP tool
  `reconcile-declared-build-state-record`.
- The preview mode is stubbed and not functional.
- Verification checks include both hard failures and warnings
  that will become hard failures as enforcement tightens.

### Enforced invariant

- Reconcile failure leaves the running generation untouched.
  No partial state is promoted.

### Known limitation

- Reconcile can hang if the build logic is malformed.
- There is no timeout or cancellation once started.
- There is no lock preventing concurrent reconciles. Running
  two at once will corrupt state.

## 10. Native loading

### Problem

The reconciler produces ELF objects in the objlib. To run a
binary, the system must load these objects, resolve their
dependencies, apply relocations, and transfer control — all
at runtime, without an external dynamic linker.

### Decision

Vehir uses its own ELF loader rather than the system dynamic
linker. Binaries are loaded from the objlib by resolving
their dependency closure, mapping objects, and applying
relocations at runtime.

### Current behavior

- The loader reads manifests to discover dependencies, walks
  the closure, maps objects from the objlib, patches
  relocations, and jumps to the entry function.
- Loading happens at boot and when spawning worker binaries
  during reconcile.

### Enforced invariant

- A binary whose dependencies cannot be resolved will not
  load. The loader fails rather than guessing.

### Known limitation

- All three loading paths still coexist. Migration to a
  single loader is not complete.

## 11. Tool boundary

### Problem

An agent needs to invoke system capabilities. Each capability
must be reachable through a single, well-defined entry point.
Without one, the agent must know about multiple interfaces,
protocols, and access patterns — which creates confusion and
security gaps.

### Decision

MCP is the primary agent-facing capability boundary. MCP-exposed
operations are invoked through the single `vehir-shell` tool. Internal
components communicate through the loader, portals, worker calls, and
service supervision; these mechanisms are not part of the public
agent-facing tool interface.

The agent sends a JSON request naming an operation and its parameters.
The boundary validates the request, checks capability permissions, and
dispatches to the handler.

### Current behavior

- The MCP server listens on 127.0.0.1, port from
  `etc/mcp/listen.json`.
- The shell dispatches by operation name. The operation
  registry (`tool-policy.json`) is generated from source,
  not hand-written.
- Each tool's operation and parameter names are checked against its generated
  request contract before the handler runs.

### Enforced invariant

- Every request passes through capability validation before
  dispatch.

## 12. Code-derived tool contracts

### Problem

If tool request contracts are written by hand, they drift from the
actual code. A parameter gets added to the implementation
but forgotten in the contract — or the contract promises a
parameter the code ignores. The contract must come from the
same source as the behavior.

### Decision

Tool request contracts are derived from the implementation code,
not written separately. The derivation traces how the tool
parses its input — which fields it reads, which are required,
which are optional — and produces a request contract from that
analysis. When the code changes, the contract follows
automatically.

### Current behavior

- Request contracts are derived from IPM source and written to
  `tool-policy.json`. Tools are registered there with their
  parameter declarations.

### Enforced invariant

- Unknown parameters in a tool request are rejected.

### Known limitation

- Not all parameter forms are recognized by the derivation.

## 13. Error catalog

Vehir uses a centralized catalog of stable error identifiers, messages,
and remediation instructions. The standard reporting path accepts a
catalog identifier and produces a structured error response; identifiers
absent from the catalog are rejected.

Whole-universe checks detect missing catalog entries and code paths that
bypass the standard mechanism. Some legacy error paths remain, and catalog
completeness is not yet enforced as a hard build failure.

## 14. Whole-universe analysis

Before promotion, the reconciler runs checks across the full
package graph. These catch cross-package issues that are
invisible when looking at individual packages in isolation.
Checks start as warnings and move to hard failures as
enforcement tightens.

Not all checks are hard failures yet.

## 15. Runtime and concurrency

The runtime is a user-space kernel — not to be confused with
the Linux kernel. It manages arenas (pre-allocated memory
regions) and workers (execution units). Workers communicate
through shared memory and doorbell descriptors.

The loader enters the kernel, which initializes its arena and launches the
root worker. Managed services are launched and supervised through the same
worker model.

Despite the worker infrastructure, everything currently runs
single-core. Multi-core execution is not yet active.

## 16. Capability isolation

MCP is the primary agent-facing capability boundary.

Agent-facing filesystem writes are mediated by capability-specific tools.
Some tools currently apply operation-specific path guards that restrict
which locations or target classes they may modify. Other capabilities
explicitly authorize destinations outside the generation tree, such as
export targets or user service configuration. There is no single global
rule limiting all writes to the generation tree.

A unified filesystem authorization backend is planned. It will provide a
shared policy model for deciding which capabilities may read, create,
modify, rename, or remove particular targets. Existing tool-specific guards
will be migrated to that backend without broadening their privileges.

The tool policy controls which operations are available.
Capability checks are not advisory.

The kernel launches and manages workers. Complete memory isolation between
workers is not yet implemented, so a faulting worker can affect other
workers and may terminate the kernel.

## 17. Service supervision and observability

The supervisor starts declared services at boot from
configuration files in `etc/vsm/`. Services are loaded
through the runtime linker as workers.

## 18. Export and distribution

The `export-generation-toward-relocatable-target` tool materializes the
selected generation, including its runtime artifacts and IPM source tree,
into a directory suitable for archiving. The exported generation does not
include a `current` symlink, and host-specific absolute paths must still be
corrected during installation.

GitHub releases carry the generation archive (`N.tar.gz`) as
the primary artifact. The Git repository itself contains
only documentation — the full IPM source tree is inside the
archive.
