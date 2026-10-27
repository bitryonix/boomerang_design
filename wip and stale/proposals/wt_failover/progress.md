# WT failover and Boomletwo recovery progress

The current recovery candidate uses self-contained ping checkpoints with an
offline designated Boomletwo. It has no continuously online ST or backup
requirement. An independent mystery draw needs a conservative floor; a checkpoint
alone does not establish latest state or exclude a returning original device.

## Deliverables

| Deliverable | Files |
| --- | --- |
| WT failover rules | [Proposal](README.md), diagrams 01 through 08 |
| Checkpoint and mystery policies | [Boomletwo candidate](boomlet_rollover.md), [sequence](09_boomlet_rollover.puml) |
| Candidate calculations and counterexamples | [Checks](boomlet_rollover_model.md) |
| WT wire draft | [Contract](wire_contract.md), [catalog](wire_catalog.json), [vectors](canonical_vectors.json) |
| Security requirements and limits | [Review](security_verification.md), [issues](security_issues.md), [requirements](security_requirements.md), [resources](resource_limits.md) |
| Separate WT and ordering experiments | [Existing model](security_model.md) |

## Evidence and limits

The checkpoint checks cover 1,152,921 conservative-threshold cases, including
stale counters. They reproduce immediate completion for 80 of 100 raw redraws
at counter 80, and an independent parallel-trial example increasing completion
probability from 1/4 to 7/16. They also exhibit indistinguishable checkpoint
observations with different later duties, votes and source availability.

These checks count valid progress units; they do not prove elapsed block time,
latest-state recovery, source exclusion or complete activation. Existing WT
models retain their separate scopes. The wire catalog does not yet allocate
checkpoint fields or a five-ping pong format.

## Required work

- [ ] Choose an enforceable old-source exclusion mechanism that remains safe
  when an attacker falsely claims the source is lost.
- [ ] Preserve later or concealed rescue duties, WT votes and replay state
  without a continuously online ST or Boomletwo.
- [ ] Select the mystery policy and define recovery progress and incarnation
  rules, including already-reached state and fresh local signing sessions.
- [ ] Finish the bounded checkpoint schema, fixed padding, detached bundle,
  distribution and retention policy, and version-selected pong changes.
- [ ] Resolve authenticated chain observation and quantify fallback exposure
  from any additional recovery delay.
- [ ] Compose the complete mechanism with WT recovery, duress and signing;
  verify cryptographic encodings, device lifecycle and measured resource costs.
