# Security

We intend to provide comprehensive security analysis and risk review for the protocol.

## Current approach

Our current approach has 2 main areas:

1. **Leaked data**: First, we're going to assess the impact of leaking each piece of data appeared in the protocol. This is currently under development and the work in progress file can be reached at [leaked_data_analysis(wip).xlsx](security/leaked_data_analysis(wip).xlsx). Results, alongside an entity-data table, will be used to help us analyze possible passive attack scenarios by compromised entities.

2. **Integrity chain and guarantees**: We will then represent which entity, channel or assumption each entity relies on for the integrity of the data provided, i.e. the chain of trust and authority in the protocol will be shown.

Using results from the above approaches, we will try to find possible attack vectors for an active attacker holding one or more adversary entities that try to undermine Boomerang's protection of the assets involved.

## Roadmap

- [ ] Impact analysis of leaked data **[in progress]**
- [ ] Passive attack analysis
- [ ] Data integrity audit **[in progress]**
- [ ] Authentication audit
- [ ] Trust assumptions audit
- [ ] Active attack analysis
