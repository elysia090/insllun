# MODEL.md

## Purpose

This document defines the Go type model for **insllun**.

The model is intentionally strict. It follows the **version 1 boundary** defined in `SPEC.md` and does **not** include broader OCI fields that insllun version 1 rejects.

The model is split into four stable domains:

* **plan**: the inspect-first execution contract
* **state**: the OCI-style runtime state
* **diff**: semantic plan comparison output
* **validation**: machine-readable validation results

`plan` and `state` are distinct.

* `plan` is an insllun-specific pre-execution artifact
* `state` is the OCI-style runtime state of a container instance

All types below are intended to live under:

```text id="j5t9p4"
internal/model/
```

---

## Design rules

1. The exported model reflects the **supported insllun v1 subset**, not the full OCI schema.
2. Unsupported OCI fields must be handled in the raw OCI input layer, not in this model.
3. `inspect` and `run` must share the same `Plan`.
4. `declared` and `effective` remain distinct concepts.
5. `Plan` must be stable and diff-friendly.

---

## `internal/model/plan.go`

```go id="q1t4c7"
package model

const (
	PlanAPIVersionV1Alpha1 = "v1alpha1"
)

type Plan struct {
	APIVersion  string            `json:"apiVersion"`
	OCIVersion  string            `json:"ociVersion"`
	Bundle      BundleRef         `json:"bundle"`
	RootFS      RootFSPlan        `json:"rootfs"`
	Process     ProcessPlan       `json:"process"`
	Linux       LinuxPlan         `json:"linux"`
	Mounts      []MountPlan       `json:"mounts"`
	Cgroups     CgroupPlan        `json:"cgroups"`
	Seccomp     *SeccompPlan      `json:"seccomp,omitempty"`
	Annotations map[string]string `json:"annotations,omitempty"`
	Declared    *PlanSnapshot     `json:"declared,omitempty"`
	Effective   *PlanSnapshot     `json:"effective,omitempty"`
	Derived     DerivedPlan       `json:"derived"`
}

type BundleRef struct {
	Path       string `json:"path"`
	ConfigPath string `json:"configPath"`
}

type RootFSPlan struct {
	Path     string `json:"path"`
	Readonly bool   `json:"readonly"`
}

type PlanSnapshot struct {
	RootFS      *RootFSPlan        `json:"rootfs,omitempty"`
	Process     *ProcessPlan       `json:"process,omitempty"`
	Linux       *LinuxPlan         `json:"linux,omitempty"`
	Mounts      []MountPlan        `json:"mounts,omitempty"`
	Cgroups     *CgroupPlan        `json:"cgroups,omitempty"`
	Seccomp     *SeccompPlan       `json:"seccomp,omitempty"`
	Annotations map[string]string  `json:"annotations,omitempty"`
}

type ProcessPlan struct {
	Terminal        bool              `json:"terminal"`
	ConsoleSize     *ConsoleSize      `json:"consoleSize,omitempty"`
	Cwd             string            `json:"cwd"`
	Args            []string          `json:"args"`
	Env             map[string]string `json:"env"`
	User            UserPlan          `json:"user"`
	NoNewPrivileges bool              `json:"noNewPrivileges"`
	Capabilities    CapabilitySets    `json:"capabilities"`
	RLimits         []RLimit          `json:"rlimits"`
}

type ConsoleSize struct {
	Height uint `json:"height"`
	Width  uint `json:"width"`
}

type UserPlan struct {
	UID            uint32   `json:"uid"`
	GID            uint32   `json:"gid"`
	AdditionalGIDs []uint32 `json:"additionalGids"`
}

type CapabilitySets struct {
	Bounding    []string `json:"bounding"`
	Effective   []string `json:"effective"`
	Inheritable []string `json:"inheritable"`
	Permitted   []string `json:"permitted"`
	Ambient     []string `json:"ambient"`
}

type RLimit struct {
	Type string `json:"type"`
	Hard uint64 `json:"hard"`
	Soft uint64 `json:"soft"`
}

type LinuxPlan struct {
	UIDMappings       []IDMapping          `json:"uidMappings"`
	GIDMappings       []IDMapping          `json:"gidMappings"`
	Sysctl            map[string]string    `json:"sysctl"`
	Resources         LinuxResourcesPlan   `json:"resources"`
	CgroupsPath       string               `json:"cgroupsPath,omitempty"`
	Namespaces        []NamespacePlan      `json:"namespaces"`
	MaskedPaths       []string             `json:"maskedPaths"`
	ReadonlyPaths     []string             `json:"readonlyPaths"`
	MountLabel        string               `json:"mountLabel,omitempty"`
	RootFSPropagation RootFSPropagation    `json:"rootfsPropagation,omitempty"`
}

type IDMapping struct {
	ContainerID uint32 `json:"containerID"`
	HostID      uint32 `json:"hostID"`
	Size        uint32 `json:"size"`
}

type LinuxResourcesPlan struct {
	Memory  *MemoryResources    `json:"memory,omitempty"`
	CPU     *CPUResources       `json:"cpu,omitempty"`
	PIDs    *PIDsResources      `json:"pids,omitempty"`
	Unified map[string]string   `json:"unified,omitempty"`
}

type MemoryResources struct {
	Limit *int64 `json:"limit,omitempty"`
	Swap  *int64 `json:"swap,omitempty"`
}

type CPUResources struct {
	Shares *uint64 `json:"shares,omitempty"`
	Quota  *int64  `json:"quota,omitempty"`
	Period *uint64 `json:"period,omitempty"`
}

type PIDsResources struct {
	Limit int64 `json:"limit"`
}

type NamespaceType string

const (
	NamespacePID     NamespaceType = "pid"
	NamespaceNetwork NamespaceType = "network"
	NamespaceMount   NamespaceType = "mount"
	NamespaceIPC     NamespaceType = "ipc"
	NamespaceUTS     NamespaceType = "uts"
	NamespaceUser    NamespaceType = "user"
	NamespaceCgroup  NamespaceType = "cgroup"
)

type NamespaceMode string

const (
	NamespaceModeNew     NamespaceMode = "new"
	NamespaceModeInherit NamespaceMode = "inherit"
	NamespaceModePath    NamespaceMode = "path"
)

type NamespacePlan struct {
	Type NamespaceType `json:"type"`
	Mode NamespaceMode `json:"mode"`
	Path string        `json:"path,omitempty"`
}

type RootFSPropagation string

const (
	RootFSPropagationPrivate    RootFSPropagation = "private"
	RootFSPropagationShared     RootFSPropagation = "shared"
	RootFSPropagationSlave      RootFSPropagation = "slave"
	RootFSPropagationUnbindable RootFSPropagation = "unbindable"
)

type MountPlan struct {
	Destination string           `json:"destination"`
	Type        string           `json:"type"`
	Source      string           `json:"source"`
	Options     []string         `json:"options"`
	Readonly    bool             `json:"readonly"`
	Bind        bool             `json:"bind"`
	Recursive   bool             `json:"recursive"`
	Propagation MountPropagation `json:"propagation,omitempty"`
}

type MountPropagation string

const (
	MountPropagationPrivate     MountPropagation = "private"
	MountPropagationRPrivate    MountPropagation = "rprivate"
	MountPropagationShared      MountPropagation = "shared"
	MountPropagationRShared     MountPropagation = "rshared"
	MountPropagationSlave       MountPropagation = "slave"
	MountPropagationRSlave      MountPropagation = "rslave"
	MountPropagationUnbindable  MountPropagation = "unbindable"
	MountPropagationRUnbindable MountPropagation = "runbindable"
)

type CgroupPlan struct {
	Path string            `json:"path,omitempty"`
	V2   map[string]string `json:"v2"`
}

type SeccompPlan struct {
	DefaultAction   SeccompAction    `json:"defaultAction"`
	DefaultErrnoRet *uint32          `json:"defaultErrnoRet,omitempty"`
	Architectures   []SeccompArch    `json:"architectures"`
	Flags           []SeccompFlag    `json:"flags"`
	Syscalls        []SeccompSyscall `json:"syscalls"`
}

type SeccompAction string

const (
	SeccompActKillProcess SeccompAction = "SCMP_ACT_KILL_PROCESS"
	SeccompActKillThread  SeccompAction = "SCMP_ACT_KILL_THREAD"
	SeccompActTrap        SeccompAction = "SCMP_ACT_TRAP"
	SeccompActErrno       SeccompAction = "SCMP_ACT_ERRNO"
	SeccompActTrace       SeccompAction = "SCMP_ACT_TRACE"
	SeccompActAllow       SeccompAction = "SCMP_ACT_ALLOW"
	SeccompActLog         SeccompAction = "SCMP_ACT_LOG"
	SeccompActNotify      SeccompAction = "SCMP_ACT_NOTIFY"
)

type SeccompArch string

const (
	SeccompArchX86             SeccompArch = "SCMP_ARCH_X86"
	SeccompArchX86_64          SeccompArch = "SCMP_ARCH_X86_64"
	SeccompArchX32             SeccompArch = "SCMP_ARCH_X32"
	SeccompArchARM             SeccompArch = "SCMP_ARCH_ARM"
	SeccompArchAARCH64         SeccompArch = "SCMP_ARCH_AARCH64"
	SeccompArchMIPS            SeccompArch = "SCMP_ARCH_MIPS"
	SeccompArchMIPS64          SeccompArch = "SCMP_ARCH_MIPS64"
	SeccompArchMIPS64N32       SeccompArch = "SCMP_ARCH_MIPS64N32"
	SeccompArchMIPSEL          SeccompArch = "SCMP_ARCH_MIPSEL"
	SeccompArchMIPSEL64        SeccompArch = "SCMP_ARCH_MIPSEL64"
	SeccompArchMIPSEL64N32     SeccompArch = "SCMP_ARCH_MIPSEL64N32"
	SeccompArchPPC             SeccompArch = "SCMP_ARCH_PPC"
	SeccompArchPPC64           SeccompArch = "SCMP_ARCH_PPC64"
	SeccompArchPPC64LE         SeccompArch = "SCMP_ARCH_PPC64LE"
	SeccompArchS390            SeccompArch = "SCMP_ARCH_S390"
	SeccompArchS390X           SeccompArch = "SCMP_ARCH_S390X"
	SeccompArchRISCV64         SeccompArch = "SCMP_ARCH_RISCV64"
)

type SeccompFlag string

const (
	SeccompFlagTSync       SeccompFlag = "SECCOMP_FILTER_FLAG_TSYNC"
	SeccompFlagLog         SeccompFlag = "SECCOMP_FILTER_FLAG_LOG"
	SeccompFlagSpecAllow   SeccompFlag = "SECCOMP_FILTER_FLAG_SPEC_ALLOW"
	SeccompFlagNewListener SeccompFlag = "SECCOMP_FILTER_FLAG_NEW_LISTENER"
)

type SeccompSyscall struct {
	Names    []string      `json:"names"`
	Action   SeccompAction `json:"action"`
	ErrnoRet *uint32       `json:"errnoRet,omitempty"`
	Args     []SeccompArg  `json:"args"`
}

type SeccompArg struct {
	Index    uint             `json:"index"`
	Value    uint64           `json:"value"`
	ValueTwo *uint64          `json:"valueTwo,omitempty"`
	Op       SeccompCompareOp `json:"op"`
}

type SeccompCompareOp string

const (
	SeccompCompareNotEqual        SeccompCompareOp = "SCMP_CMP_NE"
	SeccompCompareLess            SeccompCompareOp = "SCMP_CMP_LT"
	SeccompCompareLessOrEqual     SeccompCompareOp = "SCMP_CMP_LE"
	SeccompCompareEqual           SeccompCompareOp = "SCMP_CMP_EQ"
	SeccompCompareGreaterOrEqual  SeccompCompareOp = "SCMP_CMP_GE"
	SeccompCompareGreater         SeccompCompareOp = "SCMP_CMP_GT"
	SeccompCompareMaskedEqual     SeccompCompareOp = "SCMP_CMP_MASKED_EQ"
)

type DerivedPlan struct {
	PlanSHA256                   string                `json:"planSha256"`
	PrivilegedOperationsRequired []PrivilegedOperation `json:"privilegedOperationsRequired"`
	HostDependencies             []HostDependency      `json:"hostDependencies"`
	Warnings                     []PlanWarning         `json:"warnings"`
}

type PrivilegedOperation struct {
	Name    string `json:"name"`
	Reason  string `json:"reason"`
	Details string `json:"details,omitempty"`
}

type HostDependency struct {
	Kind     HostDependencyKind `json:"kind"`
	Name     string             `json:"name"`
	Required bool               `json:"required"`
	Details  string             `json:"details,omitempty"`
}

type HostDependencyKind string

const (
	HostDependencyNamespace  HostDependencyKind = "namespace"
	HostDependencyCgroup     HostDependencyKind = "cgroup"
	HostDependencyMount      HostDependencyKind = "mount"
	HostDependencyPath       HostDependencyKind = "path"
	HostDependencyKernel     HostDependencyKind = "kernel"
	HostDependencyCapability HostDependencyKind = "capability"
)

type PlanWarning struct {
	Code    string `json:"code"`
	Field   string `json:"field,omitempty"`
	Message string `json:"message"`
}
```

---

## `internal/model/state.go`

```go id="0ddm0u"
package model

type State struct {
	OCIVersion  string            `json:"ociVersion"`
	ID          string            `json:"id"`
	Status      ContainerStatus   `json:"status"`
	PID         int               `json:"pid,omitempty"`
	Bundle      string            `json:"bundle"`
	Annotations map[string]string `json:"annotations,omitempty"`
}

type ContainerStatus string

const (
	ContainerStateCreating ContainerStatus = "creating"
	ContainerStateCreated  ContainerStatus = "created"
	ContainerStateRunning  ContainerStatus = "running"
	ContainerStateStopped  ContainerStatus = "stopped"
)
```

---

## `internal/model/diff.go`

```go id="m5vlw0"
package model

type PlanDiff struct {
	APIVersion string       `json:"apiVersion"`
	OldSHA256  string       `json:"oldSha256"`
	NewSHA256  string       `json:"newSha256"`
	Summary    DiffSummary  `json:"summary"`
	Changes    []DiffChange `json:"changes"`
}

type DiffSummary struct {
	Added      int                 `json:"added"`
	Removed    int                 `json:"removed"`
	Changed    int                 `json:"changed"`
	ByCategory map[DiffCategory]int `json:"byCategory,omitempty"`
}

type DiffChange struct {
	Category DiffCategory `json:"category"`
	Path     string       `json:"path"`
	Kind     ChangeKind   `json:"kind"`
	Before   any          `json:"before,omitempty"`
	After    any          `json:"after,omitempty"`
	Message  string       `json:"message,omitempty"`
}

type DiffCategory string

const (
	DiffCategoryProcess      DiffCategory = "process"
	DiffCategoryUser         DiffCategory = "user"
	DiffCategoryEnv          DiffCategory = "env"
	DiffCategoryCapabilities DiffCategory = "capabilities"
	DiffCategoryNamespace    DiffCategory = "namespace"
	DiffCategoryMount        DiffCategory = "mount"
	DiffCategoryRootFS       DiffCategory = "rootfs"
	DiffCategoryCgroup       DiffCategory = "cgroup"
	DiffCategorySeccomp      DiffCategory = "seccomp"
	DiffCategoryReadonlyPath DiffCategory = "readonlyPath"
	DiffCategoryMaskedPath   DiffCategory = "maskedPath"
	DiffCategoryAnnotation   DiffCategory = "annotation"
	DiffCategoryMetadata     DiffCategory = "metadata"
)

type ChangeKind string

const (
	ChangeAdded   ChangeKind = "added"
	ChangeRemoved ChangeKind = "removed"
	ChangeChanged ChangeKind = "changed"
)
```

---

## `internal/model/validation.go`

```go id="ms25eg"
package model

type ValidationReport struct {
	Valid    bool              `json:"valid"`
	Subject  ValidationSubject `json:"subject"`
	Errors   []ValidationIssue `json:"errors"`
	Warnings []ValidationIssue `json:"warnings"`
}

type ValidationSubject string

const (
	ValidationSubjectBundle ValidationSubject = "bundle"
	ValidationSubjectPlan   ValidationSubject = "plan"
)

type ValidationIssue struct {
	Code     string        `json:"code"`
	Severity IssueSeverity `json:"severity"`
	Field    string        `json:"field,omitempty"`
	Message  string        `json:"message"`
	Rule     string        `json:"rule,omitempty"`
}

type IssueSeverity string

const (
	IssueSeverityError   IssueSeverity = "error"
	IssueSeverityWarning IssueSeverity = "warning"
)
```

---

## `internal/model/request.go`

```go id="p2ik43"
package model

type InspectRequest struct {
	BundlePath string `json:"bundlePath"`
	OutputPath string `json:"outputPath,omitempty"`
}

type RunRequest struct {
	BundlePath    string `json:"bundlePath,omitempty"`
	PlanPath      string `json:"planPath,omitempty"`
	RequireSHA256 string `json:"requireSha256,omitempty"`
}

type DiffRequest struct {
	OldPlanPath string     `json:"oldPlanPath"`
	NewPlanPath string     `json:"newPlanPath"`
	Format      DiffFormat `json:"format"`
}

type ValidateRequest struct {
	TargetPath string `json:"targetPath"`
}

type DiffFormat string

const (
	DiffFormatText DiffFormat = "text"
	DiffFormatJSON DiffFormat = "json"
)
```

---

## `internal/model/constants.go`

```go id="9nvryz"
package model

const (
	DefaultPlanFileName = "plan.json"
)
```

---

## Notes

### 1. Why unsupported OCI fields are not in `Plan`

`Plan` is the stable runtime contract for insllun version 1.

It should describe only what insllun version 1 is willing to validate, inspect, diff, and apply.
Fields such as AppArmor, SELinux, Intel RDT, seccomp listener metadata, time namespace, and hooks belong in the **raw OCI decoding layer**, not here.

That keeps the public execution model small and makes long-term maintenance easier.

### 2. Why `Linux.Resources` and `Cgroups.V2` both exist

They serve different purposes.

* `Linux.Resources` preserves the logical resource declaration in the supported OCI subset
* `Cgroups.V2` preserves the effective write plan that will be applied to cgroup v2 files

This separation makes `inspect` more useful and keeps `diff` readable.

### 3. Why `PlanSnapshot` exists

`Declared` and `Effective` are both useful, but the top-level `Plan` must remain stable.

`PlanSnapshot` allows:

* richer audit output
* more precise diffing
* side-by-side raw and normalized views

without requiring two unrelated schemas.

### 4. Why `State` is separate

`State` answers:

* what container instance exists now?

`Plan` answers:

* what exactly will be applied before execution?

Merging them would make the model ambiguous and would weaken the inspect-first design.

### 5. Why `DiffChange.Before` and `DiffChange.After` use `any`

The semantic diff output is intentionally heterogeneous.

A namespace change, a cgroup value change, and a seccomp rule change do not fit one rigid payload shape well.
Using `any` keeps the diff model small while preserving stable JSON output.

### 6. Recommended package boundary

Use this model package as the stable contract layer only.

* `internal/oci`: raw OCI decoding
* `internal/plan`: raw OCI → `model.Plan`
* `internal/validate`: validation → `model.ValidationReport`
* `internal/diff`: plan comparison → `model.PlanDiff`
* `internal/state`: state persistence → `model.State`

Do not place loader logic or syscall logic inside `model`.

---

## Minimal extension point

If version 2 later expands support, extend the model in this order:

1. add raw OCI decoding support first
2. add validation support
3. add `inspect` representation
4. add `diff` representation
5. only then add `run` support

No new feature should exist in `run` unless it already exists in `inspect`.

---

## Final rule

The model exists to make the execution contract explicit.

If a runtime behavior cannot be represented here in a stable way, it should not be part of insllun version 1.

