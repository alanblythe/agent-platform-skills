# Skills

| Skill | Use it for |
| :--- | :--- |
| [`agent-platform-architecture`](agent-platform-architecture/SKILL.md) | **Designing.** The constraints that govern how agents fit together, and the six architectures they permit. One agent or several, Agent Identity or not, whether Agent Gateway is required, and how to expose an agent in Gemini Enterprise. |
| [`agent-platform-a2a`](agent-platform-a2a/SKILL.md) | **Debugging.** 401 vs 403, identifying the real caller from audit logs, Agent Identity principal forms, reading permissions out of errors, and the conditions under which a broken deployment reports success. |
| [`agent-platform-implementation`](agent-platform-implementation/SKILL.md) | **Building and verifying.** The deployment runs and is quietly wrong: what actually crosses an agent-to-agent hop, why a caller invents data a sub-agent holds, guards a model cannot game, how to build an A2UI surface that survives the round trip, and the verification traps that report a score unrelated to your agent. |
| [`agent-platform-state`](agent-platform-state/SKILL.md) | **Persisting.** Where data lives and what survives: the three stores and their lifetimes, why `app:`-prefixed state does not scope on Agent Runtime, memory scopes and sharing a collection across users, and which store fits which workload. |
