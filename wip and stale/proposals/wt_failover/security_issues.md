# WT failover security review

The findings apply to the proposed `WT_FAILOVER_V1` profile. The proposal,
[wire contract](wire_contract.md), [resource limits](resource_limits.md), and
sequence diagrams specify its behavior. The base single-WT specification keeps
its own profile; WT dispatch is version-bound while designated backup
authorization and setup export boundaries are inherited. Protocol and device implementation evidence remains necessary.

| Finding | Severity | Required control | Proposal location |
| --- | --- | --- | --- |
| 1. Pre-DIGGING closure loses a rescue signal | Critical | Discharge every frozen placeholder before initial activation PREPARE; bind exact closure scope and all five obligation records | README 4.2, 5.2, 5.3, 7.3 |
| 2. Cloned or stale backup equivocates | Critical | Preserve designated inactive backup; require exclusive authority and complete history before activation; lost-source activation remains open | README 3.3; rollover 6 |
| 3. Deactivated WT retains wire authority | High | Separate setup-selected version, exact head wrapper and channel dispatch, current-index validation, explicit historical-evidence handler | Wire contract; README 3.2, 7.4 |
| 4. Fragment export destroys recovery evidence | High | Persist exact fragment before export; keep public progress until complete transaction verification and own discharge; retain own fragment until setup retirement | README 9 |
| 5. Timeout races deliberate SAR delay | Medium | Both block-height silence and elapsed time exceeding SAR, processing, routing and observation budgets; exact in-flight discharge | Resource limits; README 4.1, 5.2 |
| 6. Historical freshness exception is too broad | Medium | Enumerate the two historical commit and gate age exceptions; retain every other authorization, chain, head and SAR check | README 7.4 |
| 7. Parsing, ballot and storage exhaustion | Medium | Fixed ceilings, trusted admission, one obligation slot, fixed journal and archives, successor-only ballots, protected duplicate results | Resource limits |
| 8. Threat-model integration is required | Medium | Incorporate profile-specific assumptions, attack paths, audit mappings and operational duties during adoption | security_requirements.md |
| 9. Host substitutes recovery bytes after authentication | High | Authenticate the exact retained ciphertext before decryption and bind it to the signed ping and designated target | Rollover 1; verification SV-01 |
| 10. Oversized or repeatedly replayed checkpoints exhaust resources | Medium | Fixed schema, padding, byte limits and one-time recovery admission; exact encoding remains open | Rollover 1, 7; verification SV-02 |
| 11. Colluding observers fabricate chain progress | High | Specify authenticated chain observation for the honest Boomlet; current real-chain delay guarantee remains blocked | README 1; verification SV-03 |
| 12. Conflicting intent locks prevent recovery | Medium | Preserve first locks and document fallback; later cooperation alone cannot reconcile them | README 6.6; verification SV-04 |
| 13. Independent redraw lowers delay or creates a second trial | Critical | Preserve an original-threshold floor or use upper-bound plus fresh draw; prevent parallel authority; no raw redraw against restored counter | Rollover 3, 6; verification SV-06 |

## Review boundaries

The [verification report](security_verification.md) contains the security argument
and remaining blockers. The [integration requirements](security_requirements.md)
define assumptions and audit work. Availability still depends on all five peers,
profile limits, required services, safe chain evidence and device integrity.
