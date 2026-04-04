# SPEC.md

## 1. Overview

**insllun** is a minimal OCI Runtime Spec–compliant runtime for Linux, implemented in Go.

Its primary design principle is **inspect-first**. Before `run`, insllun must be able to emit, as JSON, the fully resolved execution plan that describes the isolation boundaries and resource limits that will actually be applied at runtime.

OCI defines configuration, execution environment, and lifecycle semantics for containers, but it does not prescribe a public CLI. Therefore, insllun follows OCI lifecycle and configuration semantics internally while exposing a small, modern, and explicit CLI.

This specification does not attempt to implement the entire OCI surface. It defines a strict Linux subset that is small enough to stay maintainable over time. Features outside that subset are treated as **unsupported** and must be rejected explicitly rather than silently ignored.

## 2. Goals

insllun is designed to:

* read an OCI bundle and emit a stable, effective execution plan as JSON
* apply only that exact plan during execution
* make namespace, cgroup, seccomp, mounts, environment, and capabilities reviewable before execution
* make container isolation diffable in CI
* keep implementation scope small and explicit
* preserve loose coupling between CLI, planner, validator, applier, and diff engine
* minimize long-term technical debt

## 3. Non-goals

Version 1 does not aim to support:

* hooks
* rootless execution
* non-Linux platforms
* the full OCI feature surface
* high-level network setup
* FreeBSD, Windows, VM, z/OS, or Solaris support
* AppArmor, SELinux, Intel RDT, RDMA, memory policy, and other broad peripheral features
* a runtime-specific configuration file
* large sets of runtime override flags on `run`

## 4. Compliance Model

insllun is a **Linux-focused OCI Runtime Spec subset implementation**.

Compliance means:

1. OCI bundle and `config.json` are interpreted according to the OCI Runtime Spec.
2. Lifecycle semantics follow OCI `state`, `create`, `start`, `kill`, and `delete`.
3. The public CLI is project-defined and intentionally minimal.
4. Unsupported OCI fields are rejected by validation rather than ignored.
5. OCI `state` and insllun `plan` are distinct objects and must not be conflated.

## 5. Target Platform

Version 1 is limited to:

* OS: Linux
* runtime model: container process
* cgroup model: cgroup v2 only
* privilege model: rootful only
* bundle format: OCI bundle
* config file: `config.json` at the bundle root

## 6. Terms

### 6.1 bundle

An OCI bundle directory containing `config.json` and a root filesystem.

### 6.2 config

The `config.json` file at the bundle root.

### 6.3 state

The OCI container state object. It includes at least:

* `ociVersion`
* `id`
* `status`
* `pid`
* `bundle`
* `annotations`

### 6.4 plan

A runtime-specific JSON document produced by insllun. It represents the normalized and effective runtime configuration that will actually be applied: namespaces, mounts, cgroups, seccomp, environment, process settings, capabilities, warnings, and hash.

`plan` is **not** an OCI-standard object.

### 6.5 declared

The value as written in `config.json`.

### 6.6 effective

The value after normalization and resolution by insllun’s planner.

## 7. Public CLI

The public CLI is fixed as follows:

```text
insllun inspect <bundle> [--output <file>]
insllun run <bundle>
insllun run --plan <file> [--require-sha256 <hash>]
insllun diff <old-plan> <new-plan> [--format text|json]
insllun validate <bundle-or-plan>
insllun version
insllun help [command]
```

### 7.1 CLI Principles

* `insllun` with no subcommand shows top-level help
* `insllun help` and `insllun --help` are equivalent
* invalid commands print a short error plus top-level help summary
* stdout contains only the main artifact
* stderr contains warnings and errors only
* `inspect` writes only plan JSON to stdout
* `diff --format json` writes only diff JSON to stdout
* `run` does not print unnecessary success text

### 7.2 Not Supported in the CLI

Version 1 intentionally avoids:

* command aliases
* hidden flags
* implicit bundle discovery
* runtime config override flags
* a runtime-specific config file
* plugin systems

## 8. Core Architecture

The runtime is split into the following layers:

* `oci`: load and decode `config.json`
* `plan`: build the effective execution plan
* `validate`: validate config and plan
* `apply`: apply a plan to the host
* `diff`: compare plans
* `cli`: argument parsing only

`inspect` and `run` must use the **same planner**. There must not be a separate explanatory model for `inspect`.

## 9. Mapping to OCI Lifecycle

insllun exposes a small CLI, but internally follows OCI lifecycle semantics.

### 9.1 `inspect`

Not an OCI standard operation.
Reads a bundle and emits an effective execution plan.
Does not create a container.

### 9.2 `run`

A high-level command that internally performs:

1. input resolution from bundle or plan
2. validation
3. OCI `create`-equivalent application of non-process execution state
4. OCI `start`-equivalent execution of the configured process
5. cleanup after process exit
6. deletion of runtime-created resources when applicable

### 9.3 `validate`

Performs static validation of a bundle or a plan.
Does not execute anything.

### 9.4 `diff`

Compares two plans.
Not an OCI standard operation.

### 9.5 `state`

Not exposed publicly in version 1, but retained as an internal concept.

### 9.6 `kill` / `delete`

Not exposed as public commands in version 1, but their semantics are preserved internally and reserved for future extension.

## 10. Bundle Input Rules

### 10.1 Required Conditions

A bundle must satisfy:

* `config.json` exists at the bundle root
* `root` is present for Linux
* `root.path` resolves to an existing directory

### 10.2 Project Policy

For version 1, `root.path` must be one of:

* `rootfs`
* a relative path within the bundle directory

Absolute `root.path` values may be rejected in version 1 as a project policy, even if OCI permits them. This keeps review and portability simple.

## 11. Supported OCI Fields

Version 1 supports only the following OCI fields.

### 11.1 Top-level

* `ociVersion`
* `process`
* `root`
* `mounts`
* `annotations`
* `linux`

### 11.2 `process`

* `terminal`
* `consoleSize`
* `user.uid`
* `user.gid`
* `user.additionalGids`
* `cwd`
* `env`
* `args`
* `capabilities`
* `rlimits`
* `noNewPrivileges`

### 11.3 `root`

* `path`
* `readonly`

### 11.4 `mounts`

* `destination`
* `type`
* `source`
* `options`

Version 1 requires absolute mount destinations. Relative destinations are rejected even though OCI has compatibility behavior for them.

### 11.5 `linux`

* `uidMappings`
* `gidMappings`
* `sysctl`
* `resources`
* `cgroupsPath`
* `namespaces`
* `maskedPaths`
* `readonlyPaths`
* `mountLabel`
* `seccomp`
* `rootfsPropagation`

### 11.6 `linux.resources`

Version 1 supports:

* `memory.limit`
* `memory.swap`
* `cpu.shares`
* `cpu.quota`
* `cpu.period`
* `pids.limit`
* `unified`

### 11.7 `linux.namespaces`

Version 1 supports:

* `pid`
* `network`
* `mount`
* `ipc`
* `uts`
* `user`
* `cgroup`

`time` namespace is unsupported in version 1.

### 11.8 `linux.seccomp`

Version 1 supports:

* `defaultAction`
* `architectures`
* `flags`
* `syscalls[].names`
* `syscalls[].action`
* `syscalls[].errnoRet`
* `syscalls[].args`

### 11.9 `maskedPaths` / `readonlyPaths`

Both are supported and must use absolute container paths.

## 12. Unsupported and Rejected OCI Fields

Version 1 rejects the following.

### 12.1 hooks

All hooks are rejected:

* `prestart`
* `createRuntime`
* `createContainer`
* `startContainer`
* `poststart`
* `poststop`

### 12.2 Linux Peripheral Features

Rejected in version 1:

* `apparmorProfile`
* `selinuxLabel`
* `intelRdt`
* `rdma`
* `personality`
* `timeOffsets`
* `scheduler`
* `ioPriority`
* `seccomp.listenerPath`
* `seccomp.listenerMetadata`

### 12.3 Non-Linux Platform Fields

All non-Linux platform-specific fields are rejected.

### 12.4 Advanced Mount Features

The following may be rejected in version 1:

* idmapped mounts
* complex propagation combinations
* filesystem-specific opaque option sets

Unknown mount options are rejected by default.

## 13. Planner

The planner is the only path from bundle to executable runtime plan.

### 13.1 Planner Responsibilities

The planner must:

* read `config.json`
* enforce supported-subset boundaries
* resolve and normalize paths
* normalize mounts and environment
* compute effective namespace mode
* translate cgroup settings into cgroup v2 write targets
* construct the effective seccomp model
* record warnings, host dependencies, and privileged operations
* compute `planSha256`

### 13.2 Single Source of Truth

`inspect` and `run` must use the same planner implementation.

## 14. Plan JSON

`inspect` outputs a JSON document with the following shape:

```json
{
  "apiVersion": "v1alpha1",
  "ociVersion": "1.3.0",
  "bundle": {
    "path": "/abs/path/to/bundle",
    "configPath": "/abs/path/to/bundle/config.json"
  },
  "rootfs": {
    "path": "/abs/path/to/bundle/rootfs",
    "readonly": true
  },
  "process": {
    "terminal": false,
    "cwd": "/",
    "args": ["/bin/sh"],
    "env": {
      "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM": "xterm"
    },
    "user": {
      "uid": 0,
      "gid": 0,
      "additionalGids": []
    },
    "noNewPrivileges": true,
    "capabilities": {
      "bounding": [],
      "effective": [],
      "inheritable": [],
      "permitted": [],
      "ambient": []
    },
    "rlimits": []
  },
  "linux": {
    "namespaces": [
      { "type": "pid", "mode": "new" },
      { "type": "mount", "mode": "new" },
      { "type": "ipc", "mode": "inherit" },
      { "type": "uts", "mode": "inherit" },
      { "type": "network", "mode": "path", "path": "/proc/123/ns/net" }
    ],
    "uidMappings": [],
    "gidMappings": [],
    "maskedPaths": ["/proc/kcore"],
    "readonlyPaths": ["/proc/sys"],
    "rootfsPropagation": "slave",
    "sysctl": {},
    "mountLabel": null
  },
  "mounts": [
    {
      "destination": "/proc",
      "type": "proc",
      "source": "proc",
      "options": ["nosuid", "nodev", "noexec"]
    }
  ],
  "cgroups": {
    "path": "/sys/fs/cgroup/my-container",
    "v2": {
      "cpu.max": "50000 100000",
      "memory.max": "268435456",
      "memory.swap.max": "268435456",
      "pids.max": "64"
    }
  },
  "seccomp": {
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64"],
    "flags": [],
    "syscalls": [
      {
        "names": ["read", "write", "exit", "rt_sigreturn"],
        "action": "SCMP_ACT_ALLOW",
        "args": []
      }
    ]
  },
  "declared": {},
  "effective": {},
  "derived": {
    "planSha256": "…",
    "privilegedOperationsRequired": [],
    "hostDependencies": [],
    "warnings": []
  }
}
```

### 14.1 `plan` vs `state`

OCI `state` describes a created or running container instance.
insllun `plan` describes a resolved execution contract before execution.
They are separate schemas.

### 14.2 `declared` and `effective`

The plan model must preserve both concepts:

* what was declared
* what will actually be applied

### 14.3 `derived`

The `derived` section must include at least:

* `planSha256`
* `privilegedOperationsRequired`
* `hostDependencies`
* `warnings`

## 15. `inspect`

### 15.1 Input

* bundle path

### 15.2 Output

* writes plan JSON to stdout
* returns exit code `0` on success
* writes errors to stderr and exits non-zero on failure

### 15.3 Restrictions

`inspect` must not:

* create a container
* create cgroups
* perform mounts
* load seccomp
* mutate host state

### 15.4 Normalization Rules

`inspect` must normalize output into a stable form:

* all paths resolved to absolute paths
* env normalized into a sorted key-value object
* capability lists sorted
* mount options normalized and sorted
* namespace entries sorted by type
* seccomp syscall names sorted

### 15.5 Linux Default Filesystems

Although OCI Linux suggests availability of standard filesystems such as `/proc`, `/sys`, `/dev/pts`, and `/dev/shm`, insllun version 1 does **not** add them implicitly. Only mounts explicitly declared in the bundle appear in the effective plan.

## 16. `validate`

### 16.1 Targets

* bundle
* plan

### 16.2 Validation Layers

Validation consists of:

1. **OCI subset validation**
2. **project policy validation**

### 16.3 Key Checks

Validation must include at least:

* presence of `config.json`
* valid `ociVersion`
* presence of `root` for Linux
* valid existing `root.path`
* absolute `cwd`
* non-empty `args` on non-Windows
* absolute mount destinations
* namespace type uniqueness
* absolute namespace paths when present
* absolute `maskedPaths` and `readonlyPaths`
* absence of hooks
* absence of unsupported fields
* valid `unified` keys
* non-empty seccomp `names`

### 16.4 Unsupported Handling

Unsupported features are errors, not warnings.

The only exception is where host behavior may legitimately prevent full realization of a valid config, such as capabilities that cannot be granted on the current host. Those may be surfaced as warnings.

## 17. `run`

### 17.1 Inputs

* `insllun run <bundle>`
* `insllun run --plan <file>`

### 17.2 Core Rule

`run` must apply only the planner result.

Even when a bundle is passed directly, insllun must first construct a plan internally.

### 17.3 `--plan`

If `--plan` is used, `run` reads the plan file directly. The plan must match the insllun plan schema.

### 17.4 `--require-sha256`

If provided, the supplied hash must match `plan.derived.planSha256`. Otherwise execution fails.

### 17.5 Internal Create/Start Mapping

Internally, `run` performs:

* rootfs preparation
* namespace setup
* mount application
* cgroup configuration
* masked and readonly path application
* seccomp preparation
* execution of `process.args` as the final step

### 17.6 State Transitions

The internal state model includes:

* `creating`
* `created`
* `running`
* `stopped`

## 18. `diff`

### 18.1 Purpose

`diff` compares two plans in a form suitable for both humans and CI.

### 18.2 Diff Scope

The diff must cover at least:

* process args
* cwd
* env
* user
* capabilities
* `noNewPrivileges`
* namespaces
* mounts
* `maskedPaths`
* `readonlyPaths`
* cgroup values
* seccomp default action
* seccomp syscall allow/deny deltas
* annotations

### 18.3 Output Formats

* `--format text`
* `--format json`

### 18.4 Text Diff Principle

Text diff must present semantic changes rather than raw JSON deltas.

Example:

```text
Namespaces:
  added: network
  removed: none
  changed: mount -> inherit to new

Mounts:
  added: /sys (sysfs, ro,nosuid,nodev,noexec)
  changed: /proc options

Cgroups:
  memory.max: 268435456 -> 536870912

Seccomp:
  defaultAction unchanged: SCMP_ACT_ERRNO
  allowed syscalls added: bpf, perf_event_open
```

### 18.5 Canonical Stability

The planner must emit canonical ordering so that `diff` stays stable and low-noise.

## 19. Namespaces

### 19.1 Interpretation

Each namespace entry must have a `type`.

* if `path` is present, insllun joins that namespace
* if `path` is absent, insllun creates a new namespace
* if a namespace type is omitted entirely, the runtime namespace is inherited
* duplicate namespace types are errors

### 19.2 Plan Representation

Each namespace in the plan has one of these modes:

* `new`
* `inherit`
* `path`

### 19.3 Version 1 Policy

* `time` namespace is rejected
* `user` namespace is supported conceptually, but rootless mode is not supported
* `network` namespace support is limited to namespace participation, not external network orchestration

## 20. Mounts

### 20.1 Interpretation

Mounts must be applied in the order listed.

### 20.2 Version 1 Policy

* `destination` must be absolute
* `source` must not be empty
* `type` must not be empty
* unknown options are rejected
* bind and recursive bind may be supported
* propagation support is intentionally limited

### 20.3 Plan Representation

Each mount should expose:

* `destination`
* `type`
* `source`
* `options`
* `readonly`
* `bind`
* `recursive`
* `propagation`

## 21. Process

### 21.1 Interpretation

* `cwd` must be absolute
* input `env` is an array of strings
* plan `env` is normalized to a key-value map
* `args` must be non-empty
* `noNewPrivileges` is preserved
* capability sets are preserved separately

### 21.2 Environment Normalization

Input form:

```text
["K=V", ...]
```

Plan form:

```json
{
  "K": "V"
}
```

If duplicate keys are present, the last value wins. The original array may be kept under `declared`.

## 22. Cgroups

### 22.1 Version 1 Rule

Version 1 targets cgroup v2 only.

### 22.2 `unified`

`unified` keys are treated as direct cgroup v2 file mappings. Required controllers must be enabled. Missing or unavailable controllers are errors.

### 22.3 Plan Representation

The plan translates declared resources into cgroup v2 write targets such as:

```json
{
  "cpu.max": "50000 100000",
  "memory.max": "268435456",
  "memory.swap.max": "268435456",
  "pids.max": "64"
}
```

## 23. Seccomp

### 23.1 Supported Scope

Version 1 supports ordinary seccomp action rules.
Notify and listener-based modes are rejected.

### 23.2 Plan Representation

The plan must emit:

* `defaultAction`
* `architectures`
* `flags`
* `syscalls[]`

Each syscall rule may contain:

* `names`
* `action`
* `errnoRet`
* `args`

### 23.3 Diff Summary

`diff` must summarize seccomp changes at least by:

* change in `defaultAction`
* additions/removals in architectures
* additions/removals in allowed syscalls
* additions/removals of conditional rules

## 24. `maskedPaths` and `readonlyPaths`

Both are supported in version 1.
Both must be absolute paths in the container namespace.

## 25. Annotations

Annotations are preserved transparently.

Version 1 does not define annotations that alter runtime behavior.
Annotations are diffable metadata, not a secondary source of execution semantics.

## 26. OCI State

insllun may maintain OCI-style state internally.
That state includes at least:

* `ociVersion`
* `id`
* `status`
* `pid`
* `bundle`
* `annotations`

## 27. Errors and Warnings

### 27.1 Principles

* unsupported means error
* ambiguity means error
* host limitations may produce warnings in narrow cases
* silent fallback is forbidden

### 27.2 Suggested Exit Codes

* `0` success
* `1` runtime error
* `2` usage error
* `3` validation failure
* `4` diff detected changes
* `5` plan hash mismatch

### 27.3 Warning Examples

* a capability cannot be granted on the host
* a host-dependent resource cannot be fully verified during `inspect`
* a rootless-oriented bundle is detected in a rootful-only runtime

## 28. Output Stability

Plan JSON must be diff-friendly.

This requires:

* stable field order
* stable sorting
* stable path normalization
* stable env normalization
* stable hash computation

`planSha256` must be computed from canonical JSON.

## 29. Security Principles

* `inspect` must be side-effect free
* `run` must not alter execution semantics outside the plan
* unsupported features must never be silently disabled
* bundle-external rootfs paths are not allowed in version 1
* relative mount destinations are not allowed
* plan hash must support post-review integrity checks

## 30. Implementation Notes

### 30.1 Recommended Package Layout

```text
cmd/insllun/
internal/cli/
internal/oci/
internal/plan/
internal/validate/
internal/apply/
internal/diff/
internal/state/
internal/jsoncanon/
```

### 30.2 Dependency Direction

* `cli` → `plan`, `validate`, `apply`, `diff`
* `apply` → `plan`
* `diff` → `plan`
* `validate` → `oci`, `plan`
* `plan` → `oci`

Reverse dependencies should be avoided.

## 31. Version 1 Completion Criteria

Version 1 is complete when all of the following are true:

1. `inspect` emits plan JSON from a bundle
2. `validate` enforces the supported subset and project policy
3. `diff` emits semantic differences in text and JSON
4. `run` executes only through the planner result
5. OCI `create/start/kill/delete` semantics are preserved internally
6. hooks are rejected
7. cgroup v2, namespaces, mounts, seccomp, environment, and process settings all appear consistently in the plan

## 32. Future Extensions

Possible future work includes:

* public `state` command
* public `kill` and `delete` commands
* rootless support
* limited hooks support
* AppArmor and SELinux
* seccomp notify
* default Linux filesystem profiles
* richer policy engine
* machine-readable validation codes
* published JSON schema

No future extension may violate the inspect-first principle.
If a feature cannot be represented in `inspect`, it must not be added.

## 33. Final Principle

The primary artifact of insllun is not container startup.

Its primary artifact is a **reviewable execution plan** that shows, before execution, exactly:

* which isolation boundaries will exist
* which resource limits will apply
* which syscall restrictions will apply
* which mounts will exist
* which environment will be present

`run` is not the center of the design.
`inspect` is the center.
`run` exists to apply the contract that `inspect` made visible.



