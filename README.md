# Cloud-Platform

A small AWS infrastructure-as-code project that checks the same security controls **twice**: once before deployment against the planned change, and once after deployment against the live resources — using the same control IDs on both sides.

Built with [OpenTofu](https://opentofu.org/), [Conftest/OPA](https://www.conftest.dev/) for custom policy, [Trivy](https://trivy.dev/) for a baseline misconfiguration scan, and a small Python checker for post-deployment verification.

---

## Status

**Implementation in progress.** This README currently documents the design; code is being added incrementally.

| Component | State |
|---|---|
| Design | Complete |
| `infra/` — OpenTofu configuration | Not yet committed |
| `policy/` — Rego controls + their unit tests | Not yet committed |
| `compliance/` — post-deploy checker | Not yet committed |
| `Makefile`, CI workflow | Not yet committed |
| ADRs, control matrix | Not yet committed |

Every figure and command output that eventually appears below will be pasted from a real run, not estimated. No live AWS deployment has been performed — see [Scope](#scope) for why, and what that means.

---

## The idea

Most "compliance as code" setups pick one of two places to check:

- **Scan the code** before it merges. Catches violations early, but cannot see anything changed outside the IaC — someone editing a resource by hand in the console is invisible.
- **Scan the live account** after deployment. Sees reality, but only tells you after the problem is already running.

This project does both, and makes them comparable by giving every control a stable ID that appears in both layers:

| Layer | Reads | Answers |
|---|---|---|
| Pre-deploy (`policy/`) | the `tofu plan` JSON | what is **about to** exist |
| Post-deploy (`compliance/`) | live AWS APIs | what **actually** exists |

Because both layers report against the same control IDs, a disagreement between them is meaningful: if `CTL-004` passes in the plan but fails against the live account, something changed outside the IaC. **That disagreement is configuration drift**, and it only becomes visible when both layers exist and speak the same vocabulary.

### Why the policy layer reads plan JSON rather than the HCL source

Writing policy against raw `.tf` text is easy to bypass, because values can come from variables, locals, `for_each`, or modules:

```hcl
variable "allowed_cidr" {
  default = "0.0.0.0/0"
}

resource "aws_security_group_rule" "ssh" {
  from_port   = 22
  cidr_blocks = [var.allowed_cidr] # the literal never appears in the source
}
```

A text search for `0.0.0.0/0` finds nothing here. The plan JSON, by contrast, contains the resolved value, so the policy sees what will really be created. Generating that plan JSON **without AWS credentials** is therefore a requirement of the pipeline, not an optimisation.

---

## Controls

| ID | Control |
|---|---|
| `CTL-001` | S3 buckets must have all four Block Public Access settings enabled, and no public bucket policy or ACL |
| `CTL-002` | No ingress rule may allow `0.0.0.0/0` or `::/0` on an administrative or data-plane port |
| `CTL-003` | No IAM policy may combine `Action: "*"` with `Resource: "*"` |
| `CTL-004` | Storage and logs must be encrypted at rest, and every CloudWatch log group must set an explicit retention period |
| `CTL-005` | Every taggable resource must carry `Environment`, `Owner`, `DataClassification`, and `CostCentre` |

`CTL-002` is kept even though the current configuration creates no security groups. A control exists to constrain future changes, not only to describe the present state — and the negative tests below demonstrate that it genuinely fires.

Each control is mapped to the public framework clauses it serves (CIS AWS Foundations Benchmark, AWS Well-Architected, and the relevant APRA CPS 234 / CPS 230 themes) in `docs/COMPLIANCE.md`.

### Proving the controls actually work

A policy suite that has never been observed to fail is not evidence of anything. Verification therefore includes **negative tests**: one deliberate violation is introduced per control, each is shown failing with its control ID in the message, and then reverted. The failing output is recorded as part of the verification report.

---

## Planned structure

```
Makefile                 single entry point for every check; CI calls these targets
infra/
  main.tf                providers (pinned), state encryption, common tags, AWS Budgets guardrail
  s3_kms.tf              S3 bucket + customer-managed KMS key + Block Public Access + TLS-only policy
  lambda.tf              one small Lambda + least-privilege IAM role + log group with retention
  monitoring.tf          CloudWatch alarms + SNS, including CIS-style CloudTrail metric-filter alarms
  cloudtrail.tf          one multi-region trail, log file validation, KMS-encrypted, management events only
  variables.tf
  outputs.tf
policy/
  controls.rego          the five controls
  controls_test.rego     unit tests for all five
compliance/
  check.py               post-deploy verification, same control IDs
  test_check.py
.github/workflows/ci.yml one credential-free workflow that calls make
docs/
  COMPLIANCE.md          control matrix with public framework mappings
  adr/                   architecture decision records
```

The CI workflow deliberately contains no checking logic of its own — it installs pinned tool versions and calls `make`. That keeps what runs locally identical to what runs in CI, and keeps CI changes small enough to review.

---

## Scope

What this is:

- **One** environment, **one** small workload, local state, no live deployment performed.
- A flat `infra/` root rather than a `modules/` + `envs/` split. At this size a flat layout is the honest structure; module extraction is a later step, not a thing to simulate.
- Verified entirely offline: formatting, validation, a baseline scan, custom policy against credential-free plan JSON, unit tests, and the negative tests described above.

What this is not, and what I would add next:

- **No live deployment.** I deliberately did not deploy this into any account I did not own outright, so the post-deploy checker is demonstrated against its unit tests rather than a running environment. Wiring it to a real account is a configuration change, not a code change.
- **No remote state backend.** Production would use remote state with locking; the state-encryption ADR covers what changes.
- **No deploy or drift-detection pipeline.** Production would add a manually approved deploy job authenticating via OIDC (never static keys), a post-deploy compliance job publishing its report as an artifact, and a scheduled `tofu plan -detailed-exitcode` drift check.
- **No VPC.** The Lambda needs no VPC attachment, and a VPC with no workload in it is decorative. The reasoning, including the cost of the private-subnet egress options, is recorded as an ADR.

---

## Cost

The configuration is deliberately shaped to have close to no idle cost: the workload is a Lambda outside any VPC, so it needs neither a NAT Gateway nor interface VPC endpoints, and the only always-on charges are a single customer-managed KMS key and a small number of CloudWatch alarms. An AWS Budgets guardrail with alerts is applied before anything else, so a mistake surfaces as an email rather than as an invoice.

A per-component cost table with a source URL and region for every figure will be added here once the configuration is committed.

---

## Running it

```bash
make ci          # the full offline gate: fmt, validate, scan, policy, tests
make compliance  # post-deploy checker (requires a deployed target)
```

`make ci` requires no AWS credentials and no secrets, so it runs identically on a laptop and on a clean CI runner.

---

## Licence

MIT
