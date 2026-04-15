# PLAN.md

## 1. Purpose

This document translates and operationalizes issue `#2` into an English research plan.

The goal is not only to describe why **insllun** looks promising, but to define a concrete evaluation framework with:

* conformance tests
* reproducible experiments
* explicit claims
* comparison baselines
* detailed acceptance criteria

Every central claim in this document should be stated in a way that allows a finite experiment to disconfirm it.
The plan should prefer claims that can fail clearly over claims that can only be defended rhetorically.

The document is intended to be used as the working plan for evaluation, implementation prioritization, and eventual paper-style argumentation.

The contribution of this project should not be framed as “a new OCI runtime implementation” alone.
The intended research contribution is the combination of:

* a deterministic lowering from strict OCI subset bundles to a canonical execution plan
* a machine-verifiable execution contract connecting `inspect`, `validate`, `diff`, and `run`
* a reproducible benchmark-and-evaluation methodology for plan determinism, rejection correctness, and execution consistency

## 2. Core Research Question

Compared with raw `config.json` handling and existing runtime-centered bundle workflows, can insllun compile a strict OCI subset into a canonical execution plan that provides measurable advantages in:

* deterministic pre-execution plan generation
* explicit unsupported-surface rejection
* semantic consistency between `inspect` and `run`
* machine-detectable semantic change classification

### 2.1 Scope and Boundaries

This research plan is intentionally scoped to:

* Linux only
* rootful execution only
* cgroup v2 only
* a strict OCI runtime-spec subset
* inspect-first review workflows

The plan does not claim generality for:

* full OCI runtime compatibility
* rootless execution
* non-Linux platforms
* orchestration-heavy or network-management-heavy runtime workflows

All claims, metrics, and conclusions in this document should be interpreted within that scope.

## 3. Main Research Themes

### 3.1 Diff Noise Reduction

Question:
How much does insllun reduce review noise compared with raw `config.json` diffs and conventional runtime-adjacent workflows?

What to observe:

* how well syntactic changes are separated from semantic changes
* how stable diffs remain under semantically equivalent bundle edits
* how usable the diff output is for CI and structured inspection

Candidate metrics:

* diff line count
* diff density per semantic change
* false-positive-style change count
* number of review checkpoints needed to understand a change

### 3.2 Pre-execution Detection Power

Question:
How well can insllun detect risky changes and unsupported surface area before execution on real bundle corpora?

What to observe:

* rejection rate for unsupported OCI fields
* rate of risky-change exposure before execution
* visibility of host dependencies and privileged operations
* fraction of problems stopped during `validate` or `inspect` rather than at execution time

Candidate metrics:

* supported / unsupported classification accuracy
* pre-execution detection rate
* number of inconsistencies deferred to runtime
* quality of warnings and errors

### 3.3 Reduction of `inspect` / `run` Semantic Drift

Question:
How much can insllun reduce the meaning gap between what `inspect` shows and what `run` actually applies?

What to observe:

* the effect of using a single planner for both `inspect` and `run`
* semantic agreement between pre-execution plans and applied runtime state
* the usefulness of `rootfs.digest` and `planSha256` for post-review integrity

Candidate metrics:

* plan-to-application agreement rate
* number and type of drift incidents
* reproducibility of `run --plan`
* number of detected hash mismatches and digest mismatches

### 3.4 Losses and Gains of a Strict Subset

Question:
How much OCI compatibility is lost by adopting a strict subset, and what is gained in return?

What to observe:

* fraction of real bundles accepted by the v1 subset
* distribution of rejected feature categories
* gains in explainability, maintainability, and machine inspectability
* security impact of rejecting unsupported surface instead of silently ignoring it

Candidate metrics:

* acceptance rate on the corpus
* distribution of rejection reasons
* reduction in implementation surface or branching complexity
* measured improvement in machine inspectability

### 3.5 Promising Research Problems for Contribution

The following are strong candidate research problems for this OCI runtime.
They are good topics because they are:

* aligned with the core design of insllun
* novel enough to produce publishable or at least clearly defensible claims
* measurable through conformance and comparative evaluation
* useful both to systems research and to practical runtime engineering

#### 3.5.1 Deterministic Execution Plans as a Machine-verifiable Intermediate Representation

Problem:
Can an inspect-first execution plan serve as a deterministic semantic intermediate representation that is stable, machine-verifiable, and faithfully executable from raw OCI configuration?

Why it matters:

* this is the central design idea of insllun
* it turns the runtime from a black-box executor into an explicit lowering pipeline
* it creates a clear research contribution beyond “another OCI runtime”

Contribution opportunity:

* define the execution plan as a canonical lowering target for supported OCI input
* show that plan generation is stable, unsupported input is rejected explicitly, and plan-level changes can be machine-scored more accurately than raw `config.json` changes

How to evaluate:

* compare raw `config.json`, canonicalized `config.json`, and plan-level outputs
* measure exact match, unsupported detection precision/recall/F1, semantic change detection F1, and plan/run consistency

#### 3.5.2 Preventing `inspect` / `run` Drift with Verifiable Execution Contracts

Problem:
Can `planSha256` and `rootfs.digest` make the contract between pre-execution inspection and actual execution measurably stronger?

Why it matters:

* many runtime workflows implicitly trust that the reviewed bundle is the one that gets executed
* insllun can make this contract explicit and testable

Contribution opportunity:

* formalize plan integrity and rootfs integrity as part of runtime semantics
* demonstrate how often drift is prevented or exposed before host mutation

How to evaluate:

* inject plan tampering and rootfs drift
* measure pre-mutation detection rate and semantic agreement between `inspect` and `run`

#### 3.5.3 Low-noise Semantic Diffing for Human and CI Audits

Problem:
Can semantic diffing over execution plans produce lower-noise and more actionable audits than structural JSON diffs?

Why it matters:

* container configuration review is often dominated by noisy structural changes
* reducing review noise is directly useful in CI, policy review, and security auditing

Contribution opportunity:

* define a semantic diff model for namespaces, mounts, cgroups, seccomp, and process settings
* demonstrate that canonical plan ordering and semantic grouping improve machine-level interpretability

How to evaluate:

* compare raw JSON diffs, normalized JSON diffs, and insllun semantic diffs
* measure diff size, semantic precision, and semantic change detection quality

#### 3.5.4 Safety and Explainability Benefits of a Strict OCI Subset

Problem:
What is the practical trade-off between compatibility and explicit machine-verifiable semantics when a runtime rejects unsupported surface instead of tolerating or silently ignoring it?

Why it matters:

* this is a recurring systems design question beyond OCI runtimes
* insllun provides a concrete setting in which the trade-off is visible and measurable

Contribution opportunity:

* characterize the “safety frontier” of a strict subset
* show what proportion of real bundles is lost, and what explainability or security benefits are gained

How to evaluate:

* measure acceptance rates on a real bundle corpus
* classify rejection reasons and compare them with review and audit gains

#### 3.5.5 Pre-execution Exposure of Host Dependencies and Privileged Operations

Problem:
Can a runtime expose host dependencies, privilege requirements, and unsupported kernel surface early enough to improve portability and safety?

Why it matters:

* many container failures come from hidden host assumptions rather than malformed configs
* surfacing these assumptions before execution is operationally valuable

Contribution opportunity:

* model host dependencies as first-class plan outputs
* study whether early exposure reduces runtime surprises and improves portability diagnostics

How to evaluate:

* collect bundles that fail or vary across hosts
* compare runtime-only failure discovery against plan-time exposure

#### 3.5.6 A Benchmark Corpus for Inspect-first OCI Runtime Research

Problem:
Can insllun contribute not only a runtime, but also a reusable benchmark corpus for evaluating inspect-first runtime behavior?

Why it matters:

* research value increases significantly if the artifacts are reusable by others
* benchmarks and corpora often outlive individual implementations

Contribution opportunity:

* build a public corpus of valid, unsupported, risky, semantically equivalent, and host-sensitive bundles
* define reusable evaluation scripts and metrics for other runtimes or tools

How to evaluate:

* show that the corpus supports repeated comparative experiments
* demonstrate coverage across normal, failure, and boundary cases

### 3.6 Selected Primary Research Theme

The project should narrow its primary research contribution to a single main theme:

**Deterministic lowering of OCI subset bundles into a machine-verifiable execution-plan IR.**

In other words, the main question is:

**Can an inspect-first planner compile a strict OCI subset into a canonical execution plan that is stable under repeated inspection, explicit about unsupported surface, and faithfully executable by `run`?**

This should be the main theme because:

* it is the most central idea in insllun's design
* it is more mechanically precise and falsifiable than a general “better reviewability” claim
* it naturally subsumes the other promising themes as supporting evidence
* it is measurable through exact match, classification, consistency, and overhead metrics

The primary thesis should therefore be:

**insllun contributes a deterministic, machine-verifiable execution-plan layer between OCI subset bundles and runtime execution, making execution semantics explicit, stable, and testable before host mutation.**

The other candidate themes should be treated as supporting questions for this main thesis:

* `inspect` / `run` drift prevention supports the claim that the IR is faithful to execution
* semantic diffing supports the claim that the IR is a stable machine-level object rather than a formatting artifact
* strict subset analysis supports the claim that lowering boundaries are explicit and enforceable
* host dependency exposure supports the claim that execution preconditions are externalized before mutation
* the benchmark corpus supports reproducible evaluation of the IR itself

### 3.6.1 Targeted Mechanical Contribution

The main contribution should be described in machine terms, not only workflow terms.

At the mechanism level, insllun should aim to contribute:

* a canonical plan schema that is the lowering target for the supported OCI subset
* a deterministic `bundle -> plan` lowering procedure
* explicit rejection semantics for unsupported, ambiguous, or host-incompatible input
* a `plan -> run` execution contract that does not introduce hidden runtime semantics
* a semantic diff model over plans that can be scored with standard classification metrics

The default evaluation order should be:

1. show that supported bundles deterministically lower to stable canonical plans
2. show that unsupported or ambiguous surface is rejected explicitly before execution
3. show that plan diffs track semantic changes better than raw `config.json` diffs
4. show that `run` faithfully applies the reviewed plan
5. show what compatibility is lost and what explicit semantics are gained
6. package the corpus and scripts so the claim becomes reproducible

### 3.7 Fixed Primary Metrics

The plan should use standard machine-facing metrics to test the mechanical properties of the planner contract: determinism, explicit rejection, semantic change detection, execution consistency, and overhead.

The primary metrics for the paper should therefore be fixed to established metrics such as exact match, precision, recall, F1, false accept rate, false reject rate, and runtime overhead.
Thresholds and decision rules for the main hypotheses should be fixed before interpreting the real-world corpus results.

#### M1. Inspect Exact Match Rate (I-EMR)

Definition:

* `I-EMR = exact repeated inspect matches / total repeated inspect trials`

Operationalization:

* run `insllun inspect <bundle>` repeatedly on the same valid bundle
* compare canonical JSON byte-for-byte after canonicalization
* also require identical `planSha256`

Success threshold:

* `I-EMR = 1.00` on the synthetic valid corpus

#### M2. Unsupported Detection Precision (UDP)

Definition:

* `UDP = true unsupported rejections / all unsupported rejections`

Operationalization:

* treat unsupported detection as a binary classification task
* count a case as positive only if the tool rejects it as unsupported before execution

Goal:

* maximize precision while preserving perfect or near-perfect recall

#### M3. Unsupported Detection Recall (UDR)

Definition:

* `UDR = true unsupported rejections / total labeled unsupported cases`

Operationalization:

* run `validate` and `inspect` on the unsupported corpus
* count whether unsupported cases are rejected before execution

Success threshold:

* `UDR = 1.00`

#### M4. Unsupported Detection F1 (UDF1)

Definition:

* harmonic mean of `UDP` and `UDR`

Operationalization:

* compute on the same unsupported classification task used for `UDP` and `UDR`

Purpose:

* provide a single standard summary metric for unsupported detection quality

#### M5. Plan/Run Consistency Exact Match Rate (PR-EMR)

Definition:

* `PR-EMR = accepted cases with no observable plan/run mismatch / total accepted cases`

Operationalization:

* compare execution-relevant observables against the reviewed plan
* exclude only pre-declared host-dependent fields from mismatch counting

Success threshold:

* `PR-EMR = 1.00` on the synthetic accepted corpus

#### M6. Semantic Change Detection F1 (SCD-F1)

Definition:

* F1 score for detecting labeled semantic changes from a diff output

Operationalization:

* use the mutation corpus with oracle change labels
* compare each workflow's diff output against those oracle labels
* score change-category detection as a multi-label classification task

Purpose:

* replace ad hoc “looks lower-noise” arguments with a standard classification metric

#### M7. False Accept Rate (FAR)

Definition:

* `FAR = unsupported cases incorrectly accepted / total unsupported cases`

Purpose:

* make safety regressions visible even when recall or F1 is summarized elsewhere

Target:

* `FAR = 0`

#### M8. False Reject Rate (FRR)

Definition:

* `FRR = supported cases incorrectly rejected / total labeled supported cases`

Purpose:

* quantify the compatibility cost of the strict subset and implementation behavior separately

#### M9. Runtime Overhead Ratio (ROR)

Definition:

* `ROR = treatment workflow runtime / baseline runtime`

Operationalization:

* measure end-to-end time for inspect, validate, and diff tasks
* report median and tail latency ratios

Purpose:

* ensure gains in explicit semantic analyzability are not achieved with unreasonable execution overhead

#### Secondary Metrics

The following remain secondary metrics:

* compatibility acceptance rate on the real-world corpus
* distribution of rejection reasons
* median diff line count
* implementation surface reduction or branching reduction
* storage size of plans and diff artifacts

## 4. Three Required Pillars

### 4.1 Conformance Testing

Purpose:
Verify that insllun behaves according to the v1 subset specification.

Minimum command surface to test:

* `insllun inspect <bundle>`
* `insllun validate <bundle>`
* `insllun validate <plan>`
* `insllun run <bundle>`
* `insllun run --plan <file>`
* `insllun run --plan <file> --require-sha256 <hash>`
* `insllun diff <old-plan> <new-plan> --format text`
* `insllun diff <old-plan> <new-plan> --format json`

Minimum corpus categories:

* valid OCI subset bundles
* bundles containing unsupported fields
* bundles that are syntactically different but semantically equivalent
* bundles containing semantically meaningful changes
* bundles containing risky changes
* bundles with host-dependent behavior
* plans with tampered `planSha256` or `rootfs.digest`

Each category must verify:

* cases expected to succeed do so reliably
* cases expected to fail are always rejected
* rejection never degenerates into silent ignore or partial application
* warning and error boundaries match the specification

### 4.2 Reproducible Evaluation

Purpose:
Ensure the results are not one-off observations and can be reproduced by a third party.

Minimum requirements:

* fix the corpus
* script the execution procedure
* fix the comparison baselines
* store output artifacts
* record the experiment environment fingerprint

Environment data to record:

* kernel version
* cgroup v2 state
* runtime and libseccomp versions
* privilege assumptions and execution user
* host conditions that may affect results

Artifacts to save:

* input bundles
* plan output from `inspect`
* `validate` results
* `run` results and failure reasons
* text and JSON diff outputs
* baseline outputs
* aggregated metrics

Required release artifacts:

* the corpus itself
* experiment scripts
* metric-computation scripts
* human-study task sheets, scoring rubrics, and anonymized aggregate results only if the optional human evaluation is run

### 4.3 Explicit Claims

Purpose:
Turn evaluation results into research claims rather than a collection of logs.
The claims should be falsifiable, not merely suggestive.

Each claim must specify:

* what baseline it compares against
* what corpus it is based on
* what metric supports it
* what exact prediction it makes
* what threshold or decision rule counts as support
* what result would disconfirm it
* what scope it covers
* what limitation or counterexample remains

Claim template:

* `baseline`: what is being compared
* `claim`: what improved or changed
* `prediction`: the concrete measurable expectation
* `evidence`: which metrics support it
* `decision_rule`: which outcome counts as support, qualification, or rejection
* `disconfirming_result`: which result would count against the claim
* `scope`: which bundles and environments it applies to
* `limitation`: where the claim weakens or fails

### 4.3.1 Falsifiability Rules

The paper should follow these rules for all central claims:

* no claim should rely only on qualitative impressions
* no central claim should be made without a named baseline that could plausibly outperform insllun
* no central claim should be made without predeclared metrics and scoring scripts
* no central claim should be reported as “supported” unless its decision rule was fixed before result interpretation
* no negative or mixed result should be omitted if it bears on the primary claim

Allowed conclusion labels:

* `supported`: the predeclared metric thresholds are met
* `qualified`: some thresholds are met, but one or more boundaries or counterexamples materially weaken the claim
* `rejected`: the predeclared disconfirming result occurs, or the baseline matches or outperforms insllun on the relevant metric without losing safety

### 4.4 Optional Human-facing Evaluation

Human evaluation is optional and should not be the primary evidence for the paper.

If included at all, it should serve only as a supporting sanity check for the user-facing interpretation of the machine results.

Rules:

* do not make the paper depend on a human study for acceptance
* do not use human metrics as the main result
* only run a small study if the machine-facing results are already strong

If performed, the study should be:

* within-subject and counterbalanced
* limited in scope
* reported as supporting evidence rather than the core contribution

The main evaluation remains machine-centered.

## 5. Shared Evaluation Design

The minimum evaluation set should include:

* supported bundle families
* OCI-valid but unsupported bundles
* syntactically different but semantically equivalent bundles
* bundles containing risky changes
* host-sensitive bundles

Comparison baselines:

* direct review of raw `config.json`
* runtime-centered bundle review and validation workflows
* if feasible, established flows around `runc` and `crun`

Expected outputs:

* a fixed corpus
* reproducible evaluation scripts
* a metric definition document
* a claim-to-evidence mapping
* observed traces for `bundle -> inspect -> validate -> run`

The evaluation must include both positive and negative cases.
The final artifact set must preserve:

* successful supported cases
* rejected unsupported cases
* host-sensitive cases
* counterexamples where the proposed workflow is weaker than a baseline

### 5.1 Concrete Corpus Specification

The corpus should be concrete enough to run, version, and release.

Use the following minimum structure.

#### A. Synthetic Conformance Corpus

Minimum size:

* `10` valid bundles
* `10` unsupported bundles
* `6` risky-but-valid bundles
* `6` host-sensitive bundles

Minimum valid bundle cases:

* minimal process-only bundle
* readonly rootfs bundle
* env normalization bundle
* mount normalization bundle
* namespace creation bundle
* external namespace path bundle
* CPU resource bundle
* memory resource bundle
* seccomp basic bundle
* annotations-preserving bundle

Minimum unsupported bundle cases:

* hooks present
* `mountLabel`
* `selinuxLabel`
* time namespace
* unsupported seccomp listener fields
* scheduler fields
* ioPriority fields
* unsupported mount feature such as idmapped mount
* unsupported project-policy path case
* any other non-v1 Linux surface chosen as a sentinel reject case

Minimum risky-but-valid cases:

* broad capability set
* external namespace join
* writable bind mount from host
* permissive seccomp default action
* high-impact cgroup limit change
* host dependency exposed in plan but still valid

Minimum host-sensitive cases:

* missing external namespace path
* controller unavailable on host
* mount source missing
* insufficient privilege for apply
* rootfs permission variance
* host-dependent seccomp or kernel-surface variance

#### B. Mutation and Diff Corpus

Minimum size:

* `6` semantically equivalent bundle pairs
* `6` semantically meaningful change pairs

Minimum semantically equivalent pairs:

* env order only
* capability order only
* namespace order only
* mount option order only
* annotation order only
* formatting-only JSON change

Minimum meaningful change pairs:

* changed namespace mode
* changed mount readonly semantics
* changed seccomp rule set
* changed cgroup CPU setting
* changed cgroup memory setting
* changed process args or cwd

#### C. Tampered Plan Corpus

Minimum size:

* `4` tampered plans

Required plan tamper cases:

* stale `planSha256`
* stale `rootfs.digest`
* semantic plan mutation with unchanged hash
* derived-field mutation after inspect

#### D. Real-world Corpus

Minimum size:

* `20-40` real bundles or image-derived bundles

Requirements:

* label each case as subset-valid, subset-invalid, or host-sensitive
* keep the provenance of each bundle
* include more than one bundle family
* retain the labeling rationale
* record which cases are expected to favor or not favor the insllun workflow

### 5.2 Recommended Corpus Layout

The repository or artifact package should eventually expose a stable layout such as:

```text
corpus/
  synthetic/
    valid/
    unsupported/
    risky/
    host-sensitive/
  mutations/
    equivalent/
    changed/
  tampered-plans/
  real/
  labels/
  expected/
```

`labels/` should store human-readable labels and machine-readable metadata.
`expected/` should store expected outcomes for conformance runs.

### 5.3 Oracle Labels and Scoring Schema

The corpus should include explicit oracle labels so that the main claims are mechanically testable.

Each case should define at least:

* `support_status`: supported, unsupported, or host-sensitive
* `expected_stage`: inspect, validate, run, or not-applicable
* `expected_outcome`: accept, reject, mismatch, or host-dependent
* `expected_change_categories`: the semantic change labels used for diff scoring
* `expected_rejection_reason`: the field or rule category responsible for rejection
* `host_sensitive_fields`: fields that are allowed to vary by host

All primary metrics should be computed from these labels through versioned scoring code.
If a case cannot be labeled precisely enough for scoring, it should not be used as primary evidence for a central claim.

## 6. Requirements for Comparative Evaluation

Comparative evaluation must show more than “insllun seems good in isolation”.

Minimum baselines:

* raw `config.json` diff review
* validate / inspect-like flows from existing runtimes
* workflows without an explicit intermediate execution plan

Minimum comparison dimensions:

* diff noise
* pre-execution unsupported detection
* semantic consistency between `inspect` and `run`
* false accept / false reject behavior
* runtime overhead
* compatibility loss due to subset boundaries

The evaluation should reveal:

* where insllun wins
* where insllun loses
* whether reduced compatibility actually buys more explicit and machine-verifiable semantics
* whether results generalize across bundle families
* where the approach does not help enough to justify its added machinery

### 6.1 Required Baseline Workflows

The paper should make the baselines explicit and reproducible instead of referring to “existing workflows” abstractly.

#### Baseline A. Raw Bundle Review

Workflow:

1. review raw `config.json`
2. compute a standard structural diff on bundle changes
3. extract machine-scored change signals from the diff
4. compare outputs against oracle labels

Purpose:

* represent the common direct structural-review workflow

#### Baseline B. Canonicalized Config Review

Workflow:

1. canonicalize `config.json` by key sorting and whitespace normalization
2. compute the canonicalized structural diff
3. extract machine-scored change signals
4. compare outputs against oracle labels

Purpose:

* separate the benefit of semantic IR from the simpler benefit of canonical formatting

#### Baseline C. Validator-augmented Runtime Workflow

Workflow:

1. run a reference validation or runtime-acceptance toolchain
2. inspect raw `config.json`
3. record validator or runtime diagnostic output
4. compare acceptance, rejection, and change signals against oracle labels

Purpose:

* represent tool-assisted runtime-centered practice without an explicit semantic execution plan

#### Treatment Workflow. insllun Plan Review

Workflow:

1. run `insllun inspect <bundle>`
2. compute the plan JSON and semantic diff
3. run `insllun validate` where relevant
4. compare acceptance, rejection, and change signals against oracle labels

Purpose:

* test the actual proposed workflow enabled by insllun

### 6.2 Comparison Matrix

The evaluation matrix should be fixed in advance.

Use at least the following mapping:

* Baselines A, B, C, and Treatment for diff and change-classification comparisons
* Baseline C and Treatment for unsupported detection comparisons
* Treatment for inspect exact-match and plan/run consistency measurements
* optional human evaluation only after the machine metrics are complete

### 6.3 Required Ablation

The paper should include at least one ablation:

* compare raw config review against canonicalized config review against semantic plan review

This is important because otherwise reviewers may argue that the gain comes only from normalization, not from the semantic IR itself.

### 6.4 Decision Rules for Comparative Results

The plan should specify in advance what counts as a supported, qualified, or rejected primary result.

For paired corpus comparisons, report:

* the point estimate for each workflow
* the paired delta between Treatment and the baseline
* a paired bootstrap `95%` confidence interval on that delta

Use the following interpretation:

* `strict win`: the predeclared margin is met and the paired-delta `95%` confidence interval is entirely favorable
* `non-inferior`: the hard safety gates are met and the Treatment point estimate is not lower than the baseline point estimate
* `tie`: the point estimates are close enough that no predeclared win margin is met
* `loss`: the Treatment point estimate is lower on a decisive metric, or a hard safety gate fails

### 6.4.1 Confirmatory Win Conditions Against Baseline B

Baseline B is the decisive comparator for the claim that the semantic plan adds value beyond canonicalized formatting alone.

The confirmatory metric against Baseline B is:

* `M6 = SCD-F1` on the mutation corpus

Treatment counts as a strict win over Baseline B only if all of the following hold:

* `SCD-F1(Treatment) - SCD-F1(Baseline B) >= 0.05`
* the paired-delta `95%` confidence interval for `SCD-F1` is entirely `> 0`
* Treatment does not introduce more false positive semantic changes than Baseline B on the semantically equivalent subset

If Treatment merely matches Baseline B, the project may still claim that the plan is executable and explicit, but it may not claim that the semantic IR is better than canonicalized formatting for change detection.

If Baseline B matches or exceeds Treatment on `SCD-F1`, the claim that the semantic IR adds value beyond normalization is rejected.

### 6.4.2 Confirmatory Win Conditions Against Baseline C

Baseline C is the decisive comparator for the claim that inspect-first planning does not lose unsupported-surface detection quality relative to runtime-centered validation workflows.

The confirmatory metrics against Baseline C are:

* `M2 = UDP`
* `M3 = UDR`
* `M4 = UDF1`
* pre-execution detection rate on the unsupported corpus

For this plan, pre-execution detection rate means:

* unsupported cases rejected at `inspect` or `validate` before any `run` attempt
* divided by the total number of unsupported cases in the unsupported corpus

Treatment satisfies the minimum claim against Baseline C only if all of the following hold:

* `UDR = 1.00`
* `FAR = 0`
* `UDP(Treatment) >= UDP(Baseline C)`
* `UDF1(Treatment) >= UDF1(Baseline C)`
* Treatment pre-execution detection rate is not lower than Baseline C

Treatment counts as a strict win over Baseline C only if, in addition:

* `UDF1(Treatment) - UDF1(Baseline C) > 0`
* the paired-delta `95%` confidence interval for `UDF1` is entirely `> 0`
* Treatment pre-execution detection rate is strictly higher than Baseline C

If Treatment only matches Baseline C on unsupported detection, the paper may claim that the explicit execution-plan workflow preserves validator-grade rejection quality while adding plan-level execution guarantees.
It may not claim that Treatment outperforms Baseline C on unsupported detection.

If Baseline C exceeds Treatment on `UDP`, `UDR`, `UDF1`, or pre-execution detection rate, the stronger claim against runtime-centered validation workflows is rejected.

### 6.4.3 Primary-claim Reading Rule

The primary claim requires different comparative outcomes against the two baselines:

* Against Baseline B, Treatment must achieve a strict win on `SCD-F1`
* Against Baseline C, Treatment must be at least non-inferior on unsupported detection, with `UDR = 1.00` and `FAR = 0`

Therefore, the primary claim is not supported by “doing reasonably well overall.”
It is supported only if the plan clearly beats canonicalized formatting on semantic change detection and does not lose validator-grade unsupported detection quality.

The primary claim is supported only if all of the following hold within the declared scope:

* `I-EMR = 1.00` on the synthetic valid corpus
* `PR-EMR = 1.00` on the synthetic accepted corpus
* `UDR = 1.00`
* `FAR = 0`
* Treatment achieves a strict win over Baseline B under Section `6.4.1`
* Treatment satisfies the minimum claim against Baseline C under Section `6.4.2`

The primary claim is qualified if:

* determinism and rejection guarantees hold, but semantic diff gains are small or limited to some bundle families
* semantic diff gains hold, but compatibility cost is materially higher than expected
* results are strong on synthetic corpora but mixed on the real-world corpus
* Treatment matches Baseline C on unsupported detection but does not strictly beat it

The primary claim is rejected if any of the following occur:

* repeated `inspect` runs disagree on supported inputs
* unsupported inputs are accepted, silently ignored, or only fail later at `run`
* `run` introduces semantics that are not represented by the reviewed plan
* Treatment fails to achieve a strict win over Baseline B on semantic change detection, implying that normalization alone explains the result
* Baseline C exceeds Treatment on unsupported detection quality

## 7. Generalizable Knowledge We Want to Extract

The final result should not stop at bundle-specific observations.
It should support broader conclusions such as:

* whether a strict runtime can expose a canonical semantic object for machine reasoning before execution
* whether deterministic lowering reduces ambiguity between declared configuration and applied execution semantics
* whether rejecting unsupported surface can be treated as a standard classification problem with strong precision and recall
* whether `planSha256` and `rootfs.digest` create an enforceable execution contract between `inspect` and `run`
* whether canonical semantic diffs improve machine-scored change detection compared with structural configuration diffs

Conditions required for generalization:

* do not evaluate only one bundle family
* include normal, error, and boundary cases
* separate and record host-sensitive cases
* retain both successful and unsuccessful examples

## 8. Detailed Acceptance Criteria

### 8.1 Stable and Side-effect-free Plan Generation

A valid OCI subset bundle must support stable execution-plan generation without host-side side effects.

At minimum, this means:

* `insllun inspect <bundle>` succeeds for valid subset bundles
* `inspect` does not perform mounts, create cgroups, load seccomp, or create namespaces as a host-side mutation
* repeated `inspect` runs on the same bundle produce stable canonical JSON and stable `planSha256`
* `rootfs.digest` is computed stably
* the difference between `declared` and `effective` is explainable
* host dependencies and privileged operations are visible in the plan

### 8.2 Strict Plan-only Application

Execution must apply only the reviewed plan.

At minimum, this means:

* `insllun run <bundle>` internally executes only through the planner result
* `insllun run --plan <file>` treats the plan as the sole execution contract
* `run --plan` verifies `rootfs.digest` before mutating host state
* `--require-sha256` enforces `planSha256` equality when provided
* digest mismatch and hash mismatch fail before host mutation
* `run` does not introduce extra runtime semantics that were not visible in `inspect`
* `insllun run <bundle>` and `insllun run --plan <file>` are semantically equivalent except for explicitly allowed host-dependent areas

### 8.3 Mandatory Rejection of Unsupported Surface

Unsupported surface must always be rejected.

At minimum, this means:

* non-v1 OCI fields become validation errors
* unsupported values are errors, not warnings
* unknown or unsupported input is never silently ignored
* bundles containing unsupported surface never partially succeed
* rejection reasons can be traced back to specific fields
* subset boundaries are consistent across the corpus

### 8.4 Low-noise Semantic Change Detection for CI and Review

Diffs must be low-noise and semantically meaningful for automated pipelines, while remaining readable enough for review.

At minimum, this means:

* semantically equivalent bundle changes do not produce inflated diffs
* semantically meaningful changes appear clearly in the diff
* text diff is readable and summarized at a review-useful level
* JSON diff is structured for machine processing in CI
* canonical ordering makes equivalent plans diff-stable
* the diff is not just a raw structural JSON delta, but remains interpretable in terms of resources, namespaces, mounts, and seccomp

### 8.5 Conformance, Reproducibility, and Explicit Claims Are All Present

The project is not complete from a research perspective unless all three exist together.

At minimum, this means:

* a conformance corpus is defined
* experiment procedures are scripted
* comparison procedures against baselines are fixed
* metrics are defined for each major claim
* evidence is retained for each claim
* a third party can rerun the same corpus and procedure

### 8.6 Generalizable Findings and Comparative Results

The evaluation must produce comparative and generalizable findings, not only local examples.

At minimum, this means:

* comparisons span multiple bundle families
* gains and losses versus baselines are shown
* the scope of generalization is stated explicitly
* counterexamples and failure conditions are recorded
* the practical trade-off of the strict subset is explained quantitatively or systematically

## 9. Expected Claims and Falsifiable Hypotheses

The study should aim to support, qualify, or reject one primary claim and several supporting claims.

Primary claim:

* insllun deterministically lowers OCI subset bundles into a machine-verifiable execution-plan IR that is more explicit and less semantically ambiguous than raw `config.json`

Supporting claims:

* insllun deterministically produces stable canonical plans for supported bundles
* insllun rejects unsupported surface early without silent fallback
* insllun preserves strong semantic consistency between `inspect` and `run`
* insllun trades compatibility for more explicit machine-verifiable semantics and explainability
* insllun semantic diffs improve machine-scored change detection over raw configuration diffs

Each claim must have:

* a baseline
* a corpus
* a metric
* a result
* a limitation or counterexample

No claim should be reported without an explicit failure boundary.
The paper should include at least one negative result table or subsection summarizing where insllun does not improve over the baseline.

### 9.1 Primary Hypothesis

`H1`: A strict inspect-first planner can deterministically lower supported OCI subset bundles into a canonical execution plan that is both executable and measurably better for machine-scored semantic change detection than raw or merely canonicalized `config.json`.

Support requires:

* `I-EMR = 1.00`
* `PR-EMR = 1.00`
* `UDR = 1.00`
* `FAR = 0`
* Treatment achieves a strict win over Baseline B under Section `6.4.1`
* Treatment satisfies the minimum claim against Baseline C under Section `6.4.2`

Disconfirming results:

* repeated `inspect` mismatch on supported inputs
* any unsupported case accepted or silently ignored
* any plan/run mismatch on non-host-sensitive fields
* no strict semantic change detection win over Baseline B
* unsupported detection quality lower than Baseline C

### 9.2 Supporting Hypotheses

`H2`: Unsupported-surface handling is a strong pre-execution classification problem, not a runtime-only failure mode.

Support requires:

* `UDR = 1.00`
* `FAR = 0`
* `UDP(Treatment) >= UDP(Baseline C)`
* `UDF1(Treatment) >= UDF1(Baseline C)`
* Treatment pre-execution detection rate is not lower than Baseline C

Disconfirming results:

* unsupported cases survive into `run`
* rejection reasons cannot be mapped back to concrete fields or rules
* Baseline C exceeds Treatment on unsupported detection quality

`H3`: The execution contract between `inspect` and `run` is enforceable rather than advisory.

Support requires:

* `PR-EMR = 1.00` on accepted synthetic cases
* tampered plans and stale digests are rejected before host mutation

Disconfirming results:

* `run` applies semantics not represented in the plan
* plan tampering or rootfs drift is detected only after mutation

`H4`: The semantic plan provides value beyond canonicalized formatting alone.

Support requires:

* `SCD-F1(Treatment) - SCD-F1(Baseline B) >= 0.05`
* the paired-delta `95%` confidence interval for `SCD-F1` is entirely `> 0`
* Treatment diff outputs remain stable on semantically equivalent pairs

Disconfirming results:

* Baseline B matches or exceeds Treatment on semantic change detection
* the apparent advantage disappears after canonicalization controls are applied

`H5`: The strict subset has a measurable and explainable trade-off surface.

Support requires:

* `FRR` and real-world acceptance rate are reported together with rejection-reason distributions
* reduced compatibility can be tied to explicit semantics, simpler boundaries, or earlier rejection

Disconfirming results:

* compatibility loss is high but the main safety or change-detection gains are not observed
* rejection behavior is inconsistent across cases in the same category

## 10. Threats to Validity

The paper should discuss threats to validity directly rather than leaving them implicit.

### 10.1 Construct Validity

Main risks:

* the chosen metrics may capture formatting quality rather than semantic normalization correctness
* diff labels may fail to distinguish structural churn from execution-relevant change
* plan/run faithfulness may be under-measured if observables are too narrow

Mitigations:

* keep the raw-review, canonicalized-review, and semantic-plan conditions separate
* define oracle labels and mismatch observables before running the experiments
* compute exact-match and classification metrics from fixed schemas rather than ad hoc interpretation

### 10.2 Internal Validity

Main risks:

* corpus-labeling mistakes
* baseline implementation bias
* untracked host variation affecting outcomes
* leakage between corpus design and metric definitions

Mitigations:

* review labels and expected outcomes against a fixed schema before scoring
* keep baseline scripts separate from treatment implementation
* version environment fingerprints together with artifacts

### 10.3 External Validity

Main risks:

* the strict subset may not generalize to full OCI workflows
* the selected real-world corpus may overrepresent certain bundle families
* the label schema may overfit the kinds of semantic changes favored by the plan format

Mitigations:

* scope claims explicitly to Linux, rootful, cgroup v2, and the chosen subset
* include more than one bundle family in the real-world corpus
* publish the label schema and include counterexamples that do not favor insllun

### 10.4 Artifact Validity

Main risks:

* experiments may not be reusable by others
* released artifacts may be incomplete or underspecified

Mitigations:

* release corpus, scripts, outputs, scoring rubrics, and metric computation code
* ensure core results can be rerun from a clean checkout with documented steps
* publish the exact decision rules used to label hypotheses as supported, qualified, or rejected

## 11. Immediate Next Steps

The next concrete planning tasks are:

* lock `M1-M9` and their success thresholds
* instantiate the concrete corpus layout and label schema
* script Baselines A, B, C, and the insllun treatment workflow
* define the experiment runner and artifact formats
* build the claim-to-evidence mapping
* predeclare the decision rules and disconfirming results for `H1-H5`
* prepare the threats-to-validity section alongside the first evaluation pass
* only decide on a small human-facing study after the core mechanical claims are already supported
