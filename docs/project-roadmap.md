# Memobase Project Roadmap

## Vision Statement

Establish Memobase as the industry-leading user-centric memory platform for LLM applications, balancing performance, cost, and latency while maintaining ease of integration and customization.

## Current State (v0.0.40)

**Version**: 0.0.40 (July 2025)  
**Status**: Stable, production-ready

### Completed Milestones

| Feature | Version | Status |
|---------|---------|--------|
| Core profile + event APIs | v0.0.1+ | Complete |
| Multi-language SDKs (Python, TS, Go) | v0.0.15+ | Complete |
| Buffer + async flush | v0.0.30+ | Complete |
| Event gist + fine-grained search | v0.0.37+ | Complete |
| YOLO profile merge (cost reduction) | v0.0.40+ | Complete |
| MCP server for agents | v0.0.33+ | Beta (tools incomplete) |
| OpenTelemetry + structured logging | v0.0.35+ | Complete |
| Roleplay features (experimental) | v0.0.34+ | Alpha |
| LOCOMO benchmark validation | v0.0.32+ | Complete |

### Known Issues & Gaps

- Event gist/tag search parity: TypeScript + Python verified; Go needs confirmation
- MCP resources/prompts: Not yet exposed (planned)
- Social graph: Design-phase only
- Typed profile slots: Prototype needed

## Q3 2025 (Current Quarter)

**Focus**: Stabilization, benchmark validation, MCP completeness

### Tasks (In Progress)

- [ ] Confirm event search parity in Go SDK
- [ ] Complete MCP tool coverage (all memory operations)
- [ ] Publish LOCOMO benchmark results publicly
- [ ] Fix known bugs (random Chinese profile, UUID validation)
- [ ] Update SDKs for consistency

### Milestones

- Benchmark results published
- MCP tools feature-complete
- All SDKs at parity for search operations

**Target completion**: End Q3 2025

## Q4 2025 (Next Quarter)

**Focus**: Feature expansion, cost optimization, user experience

### Planned Features

#### 1. Multi-Profile Schema (Mid-priority)
Allow users to maintain multiple distinct profiles within one project (e.g., work profile vs. personal profile).

**Rationale**: Different LLM applications may need isolated contexts; reduce noise in a single merged profile.

**Technical approach**:
- Add `profile_type` field to UserProfile table
- Config specifies active profiles per project
- Context API: can filter by profile_type
- Buffer flush: routes facts to correct profile based on metadata

**Estimated effort**: 2-3 weeks  
**Risk**: Schema migration complexity

#### 2. Social Graph (Lower-priority, future)
Model relationships between users (e.g., "Alice is colleagues with Bob").

**Rationale**: Enable shared memory, relationship tracking, group dynamics.

**Technical approach**:
- New `UserRelationship` table (id, project_id, user_id_a, user_id_b, relation_type)
- New `SharedContext` API to retrieve relationship-based memory
- LLM feature: detect relationships from conversations

**Estimated effort**: 4-5 weeks  
**Risk**: Increased complexity, privacy implications

#### 3. Typed Profile Slots (Lower-priority)
Add typed fields to profiles (number, boolean, date) instead of free-text only.

**Rationale**: Enable structured querying, filtering, analytics on profile attributes.

**Technical approach**:
- Extend profile_config to specify field types
- Validation in profile update
- New query operators (e.g., "age > 30")

**Estimated effort**: 2 weeks  
**Risk**: Breaking change for existing profiles

#### 4. MCP Streaming & Real-time Updates (Future)
WebSocket support for real-time memory synchronization in MCP agents.

**Rationale**: Enable live agent memory updates without polling.

**Technical approach**:
- Extend MCP server with streaming resources
- WebSocket endpoint for buffer updates
- Agent-initiated subscriptions

**Estimated effort**: 3-4 weeks  
**Risk**: Connection management complexity

### Optimization Priorities

- **LLM cost**: Further reduce tokens (compression, selective facts)
- **Latency**: Optimize context assembly (<50ms target for simple cases)
- **Search**: Hybrid search (text + semantic) for better recall

### Target Completion

End Q4 2025: Multi-profile schema + partial social graph prototype

## 2026 (Outlook)

### Strategic Goals

1. **Adoption**: 10K+ active users, 100K+ total users
2. **Performance**: <50ms latency for 95% of operations
3. **Cost**: <$0.10 per user per month (LLM costs only)
4. **Ecosystem**: SDKs in Rust, Java; integrations with popular LLM frameworks

### Potential Features

| Feature | Priority | Effort | Impact |
|---------|----------|--------|--------|
| Rate limiting + quotas | High | 1 week | Production hardening |
| Data export/import APIs | High | 2 weeks | Compliance, portability |
| Admin dashboard | Medium | 4 weeks | Operational visibility |
| Relationship detection | Medium | 3 weeks | Social graph |
| Temporal memory queries | Medium | 2 weeks | Advanced search |
| Regional deployments | Medium | 4 weeks | Data residency |
| Batch profile update | Low | 1 week | Performance |
| Profile versioning | Low | 2 weeks | Audit trail |

## Release Schedule

### Current

| Version | Target Date | Focus |
|---------|-------------|-------|
| 0.0.40 | July 2025 | Cost reduction (YOLO merge) |
| 0.0.41 | Aug 2025 | Bug fixes, SDK parity |
| 0.0.42 | Sep 2025 | MCP completeness, benchmark publish |

### Upcoming

| Version | Target Date | Focus |
|---------|-------------|-------|
| 0.1.0 | Oct 2025 | Multi-profile schema (breaking change) |
| 0.1.1-0.1.3 | Oct-Nov 2025 | Refinements |
| 0.2.0 | Dec 2025 | Social graph prototype, Q4 features |

## Success Metrics & KPIs

### Product Metrics

| KPI | Q3 Target | Q4 Target | 2026 Target |
|-----|-----------|-----------|-------------|
| Search ranking (LOCOMO) | Top 1 | Top 1 | Top 1 |
| Latency p95 (context API) | <100ms | <100ms | <50ms |
| LLM cost per cycle | 3 calls | 2.5 calls | 2 calls |
| Uptime | 99.5% | 99.5% | 99.9% |

### Community Metrics

| Metric | Q3 Target | Q4 Target | 2026 Target |
|--------|-----------|-----------|-------------|
| GitHub stars | 3K | 5K | 10K |
| PyPI downloads | 10K/month | 20K/month | 50K/month |
| Discord members | 500 | 1K | 2K |
| Active projects (cloud) | 500 | 1K | 5K |

### Business Metrics (Cloud)

| Metric | Q3 Target | Q4 Target | 2026 Target |
|--------|-----------|-----------|-------------|
| Free-tier users | 1K | 2K | 5K |
| Paid users | 50 | 150 | 500 |
| MRR | $2K | $8K | $25K |

## Dependencies & Constraints

### Technical Constraints

- PostgreSQL 17+ (pgvector extension required)
- LLM API availability (OpenAI fallback chain)
- Embedding model costs (monitor for price changes)
- Vector dimension compatibility (1536D standard)

### Resource Constraints

- Engineering team size (currently small)
- Community contributions (limited so far)
- Cloud infrastructure costs (scale with users)

### External Dependencies

- OpenAI API (primary LLM + embeddings)
- Volcengine/Doubao (fallback)
- Jina (alternative embeddings)
- Ollama/LMStudio (local inference)

## Risk Assessment

### High Risks

| Risk | Mitigation |
|------|-----------|
| LLM API price increases | Switch to local/alternative models, optimize prompts |
| Embedding model changes | Maintain compatibility with multiple models, re-embedding pipeline |
| Scale-related performance degradation | Database sharding, horizontal scaling planned |
| Community adoption slow | Marketing, integrations, public benchmarks |

### Medium Risks

| Risk | Mitigation |
|------|-----------|
| SDK parity gaps | Regular audits, feature flags for new APIs |
| Data migration complexity (breaking changes) | Versioned APIs, migration utilities |
| Operational burden (growing user base) | Automated monitoring, observability investments |

### Low Risks

| Risk | Mitigation |
|------|-----------|
| Minor bug reports | Standard bug triage, fast-track fixes |
| Documentation gaps | Continuous docs updates, community contributions |

## Communication & Stakeholder Updates

### Channels

- **GitHub**: Issues, PRs, Releases
- **Discord**: Real-time community, feature feedback
- **Changelog**: Documented changes per release
- **Roadmap**: This document (updated quarterly)

### Cadence

- **Weekly**: Internal standup (team)
- **Monthly**: Community update (Discord)
- **Quarterly**: Roadmap review & adjustment
- **Per-release**: Changelog + announcement

## Appendix: Historical Context

### Version Highlights

- **v0.0.32**: LOCOMO benchmark vs mem0/zep/langmem (SOTA achieved)
- **v0.0.33**: MCP server (Claude Desktop, Cursor support)
- **v0.0.34**: Background buffer processing, roleplay features
- **v0.0.35**: Simplified logging, API doc restructuring
- **v0.0.36**: Customizable prompt templates, search options
- **v0.0.37**: Event gist table (breaking migration), fine-grained search
- **v0.0.38**: Reduced-cost merge prompt
- **v0.0.39**: SummaryBlob support, better error logging
- **v0.0.40**: YOLO merge, 40-50% token cost reduction

### Community Milestones

- First public release: v0.0.1 (date unknown)
- LOCOMO benchmark published: v0.0.32 era
- MCP server launched: v0.0.33 era
- 1K GitHub stars: Early 2025
- Discord community >500: Mid-2025

## Future Vision (2027+)

### Long-Term Strategy

**Goal**: Memory becomes a fundamental data layer for AI applications, like databases are for web apps.

**Approach**:
1. **Consolidation**: Merge best of profile-based + event-based + graph-based approaches
2. **Ecosystem**: Expand SDKs, integrations with major LLM platforms
3. **Standards**: Propose memory interchange format (like OpenAI's structured outputs)
4. **Monetization**: Freemium model, enterprise support, managed hosting

### Aspirational Features (2027+)

- Multi-modality: Images, audio memories (not just text)
- Memory reasoning: AI explainability layer ("why was that memory retrieved?")
- Privacy preserving: Federated memory (user data stays local)
- Interoperability: Memory portability across LLM platforms
- Temporal dynamics: Memory degradation, salience decay
