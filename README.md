# Cloud-Platform

A small AWS project, built with the [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html) in TypeScript, that checks the same security controls **twice**: once before deployment while the template is being synthesized, and once after deployment against live resources — using the same control IDs on both sides.

Policy is enforced by five custom [CDK Aspects](https://docs.aws.amazon.com/cdk/v2/guide/aspects.html) plus [cdk-nag](https://github.com/cdklabs/cdk-nag) for an AWS-maintained baseline.

---

## Status

**Implementation in progress.** This README documents the design; code is being added incrementally.

| Component | State |
|---|---|
| Design | Complete |
| `lib/` — CDK stack | Not yet committed |
| `lib/aspects/` — the five control Aspects | Not yet committed |
| `compliance/` — post-deploy checker | Not yet committed |
| `test/` — stack assertions and negative tests | Not yet committed |
| `Makefile`, CI workflow | Not yet committed |
| ADRs, control matrix | Not yet committed |

Every figure and command output that eventually appears below will be pasted from a real run, not estimated. **No AWS deployment has been performed** — see [Scope](#scope) for what that means in practice.

---

## The idea

Most "compliance as code" setups pick one of two places to check:

- **Check the definition** before it merges. Catches violations early, but cannot see anything changed outside the IaC — someone editing a resource by hand in the console is invisible to it.
- **Check the live account** afterwards. Sees reality, but only reports once the problem is already running.

This project does both, and makes the two results comparable by giving every control a stable ID that appears in both layers:

| Layer | Runs | Inspects | Answers |
|---|---|---|---|
| Pre-deploy | during `cdk synth` | the synthesized CloudFormation properties | what is **about to** exist |
| Post-deploy | on demand | live AWS APIs | what **actually** exists |

Because both layers report against the same control IDs, a disagreement between them carries information: if `CTL-004` passes at synth but fails against the live account, something changed outside the CDK app. **That disagreement is configuration drift**, and it is only detectable because both layers exist and share a vocabulary.

### Why the Aspects inspect synthesized properties, not construct types

A check written against the L2 construct type (`s3.Bucket`) is easy to slip past, because the same resource can be declared at L1 (`s3.CfnBucket`) or arrive from a construct the check has never heard of. The Aspects therefore resolve the **synthesized CloudFormation properties** of each node, which is what will actually be submitted to CloudFormation. A test asserts that an L1 resource violating a control is caught, not only the L2 form.

### A useful property of this approach

`cdk synth` makes no AWS API calls, so the entire pre-deploy layer — synthesis, cdk-nag, and all five custom controls — runs with **no AWS credentials at all**. That is what lets the whole gate run identically on a laptop and on a clean CI runner with no secrets configured.

---

## Controls

| ID | Control |
|---|---|
| `CTL-001` | S3 buckets must block all public access and carry no public policy or ACL |
| `CTL-002` | No ingress rule may allow `0.0.0.0/0` or `::/0` on an administrative or data-plane port |
| `CTL-003` | No IAM policy may combine `Action: "*"` with `Resource: "*"` |
| `CTL-004` | Storage and logs must be encrypted at rest, and every log group must set an explicit retention period |
| `CTL-005` | Every taggable resource must carry `Environment`, `Owner`, `DataClassification`, and `CostCentre` |

`CTL-002` is kept deliberately even though this stack creates no security groups. A control exists to constrain **future** changes, not only to describe the present state — and the negative test below proves it fires.

Each control is mapped to the public framework clauses it serves (CIS AWS Foundations Benchmark, AWS Well-Architected, and the relevant APRA CPS 234 / CPS 230 themes) in `docs/COMPLIANCE.md`.

### Proving the controls work

A policy suite that has never been observed to fail is not evidence of anything. For each of the five controls there is a test that builds a stack genuinely violating it, synthesizes it, and asserts that an error was raised **carrying that control's ID**. A further test asserts the real stack raises none.

These are ordinary unit tests, so they run in CI on every change rather than being a manual procedure someone has to remember.

---

## What it deploys

Deliberately small, and shaped to have close to no idle cost:

- One S3 bucket and one customer-managed KMS key — public access blocked, encrypted, versioned, non-TLS access denied.
- One small Lambda with a least-privilege role and a log group with an explicit retention period.
- CloudWatch alarms on the Lambda, plus CIS-style CloudTrail metric-filter alarms for root account usage, IAM policy changes, and console sign-in without MFA, delivered to an SNS topic.
- One CloudTrail trail: multi-region, log file validation on, KMS-encrypted, **management events only**.
- An AWS Budgets guardrail with alerts, so a mistake surfaces as an email rather than an invoice.

There is **no VPC**. The Lambda needs no VPC attachment, and a VPC with no workload in it is decorative; the reasoning, including the cost of the private-subnet egress options, is recorded as an ADR.

---

## Layout

```
bin/app.ts                   CDK app entry; applies cdk-nag and the custom Aspects
lib/cloud-platform-stack.ts  the stack
lib/aspects/controls.ts      the five control Aspects
lambda/handler.js            trivial handler
compliance/check.ts          post-deploy checker (AWS SDK v3), same control IDs
test/stack.test.ts           assertions on the synthesized template
test/aspects.test.ts         negative tests: each control fires
test/check.test.ts           checker logic against stubbed SDK responses
Makefile                     single entry point for every check
.github/workflows/ci.yml     one credential-free workflow that calls make
docs/COMPLIANCE.md           control matrix with public framework mappings
docs/adr/                    architecture decision records
docs/MULTI-CLOUD.md          honest note on the CDK being AWS-only
```

The CI workflow contains no checking logic of its own — it installs pinned versions and calls `make`. That keeps what runs locally identical to what runs in CI, and keeps CI changes small enough to review.

---

## Scope

What this is:

- One stack, one small workload, verified entirely offline: build, lint, credential-free synth (which runs cdk-nag and all five controls), and the unit tests including the negative tests above.
- A flat single-stack layout. At this size that is the honest structure; multi-stack and multi-environment separation is a later step, not something to simulate.

What this is not, and what I would add next:

- **No deployment has been performed.** I did not deploy this into any account I do not own outright, so the post-deploy checker's correctness is demonstrated against stubbed SDK responses rather than a live environment. Pointing it at a real account is configuration, not a code change — but it has not been done, and the README will say so until it has.
- **No deployment pipeline.** Production would add a manually approved deploy job authenticating via OIDC rather than static keys, a post-deploy compliance job publishing its report as an artifact, and a scheduled drift check.
- **The CDK is AWS-only.** For an estate spanning more than one cloud this is a real limitation, not a detail. `docs/MULTI-CLOUD.md` and ADR 0001 discuss the trade-off and when a different tool would be the better choice.
- **No AWS Config.** The hand-written checker replaces it; the ADR covers what that costs as well as what it saves.

---

## Cost

The stack has close to no idle cost by design: the workload is a Lambda outside any VPC, so it needs neither a NAT Gateway nor interface VPC endpoints, and the only always-on charges are a single customer-managed KMS key and a small number of CloudWatch alarms.

A per-component cost table with a source URL and region for every figure will be added here once the stack is committed.

---

## Running it

```bash
make install   # pinned node version + dependencies
make ci        # the full offline gate: build, lint, synth (cdk-nag + Aspects), tests
make synth     # just synthesize and run the pre-deploy controls
make test      # just the unit tests, including the negative tests
```

`make ci` requires no AWS credentials and no secrets, so it runs identically on a laptop and on a clean CI runner.

---

## Licence

MIT
