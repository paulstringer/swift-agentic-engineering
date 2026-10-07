# Roadmap

## Phase 1 — Foundations
- [ ] Write the thesis (`docs/thesis.md`)
- [ ] Survey existing Swift tooling (architecture, duplication, complexity, mutation) — identify actual gaps, not assumed ones
- [ ] Run SwarmForge on a small real Swift task — identify where its workflow lacks Swift-specific quality feedback (two-pack done: no Swift quality gates ran, see [findings](research/findings/exp-02-two-pack.md); four- and six-pack outstanding)
- [ ] Investigate the dependency-checker model
- [x] Smallest SwiftSyntax dependency-graph experiment (in `swift-agentic-quality-engineering`) — see [research/experiments/dependency-graph.md](research/experiments/dependency-graph.md)

## Phase 2 — Verification feedback loops
- [ ] Dependency/architecture checking
- [ ] Duplication detection
- [ ] Complexity + coverage (CRAP)
- [ ] Mutation testing
- [ ] Agent workflow integration (SwarmForge)

## Phase 3 — Evidence
- [ ] Benchmark agents under varying verification regimes
- [ ] Publish findings
- [ ] Distill into public technical writing

## Phase 4 — Publish SME frameworks
- [ ] Agentic Mobile Engineering Assessment (framework)
- [ ] Agent Readiness Assessment (framework)
- [ ] Agent Quality Framework
- [ ] Mobile Agent Workflow Architecture
- [ ] Agent Effectiveness Programme
