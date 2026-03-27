# n8n — Executive Summary

## Purpose

n8n is a self-hosted visual workflow automation platform (forked from the open-source n8n community) that enables non-technical users and developers to orchestrate complex business processes across 200+ integrations without writing code. The FiveX deployment extends n8n with custom nodes and services for AI-assisted workflow automation, making it a central hub for business logic orchestration across the platform.

## Current State

- **Maturity:** Production (version 2.12.0, actively maintained)
- **Version:** 2.12.0 (semantic versioning applied)
- **Last Activity:** Recent (monorepo actively developed and merged regularly)
- **Test Coverage:** Comprehensive (Jest unit tests, Vitest frontend, Playwright E2E)
- **Lines of Code:** ~200,000+ (pnpm monorepo with 60+ packages)

## Tech Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| Node.js | 22.16+ | Runtime environment |
| TypeScript | Latest | Primary language for backend and node implementations |
| Vue 3 | Latest | Reactive workflow editor UI |
| Vite | Latest | Frontend bundler and dev server |
| Express | Latest | HTTP API server framework |
| TypeORM | Latest | Database abstraction (SQLite/PostgreSQL support) |
| Pnpm | 10.22.0+ | Monorepo package manager (npm blocked by preinstall check) |
| Turbo | 2.8.9 | Build orchestration and task running |
| Jest | 29.6.2 | Backend unit testing |
| Vitest | Latest | Frontend unit testing |
| Playwright | Latest | End-to-end testing |
| Biome | 1.9.0 | Code formatting and linting |

## Key Capabilities

- **Visual Workflow Editor:** Drag-and-drop interface for building workflows; node-based composition with connections representing data flow
- **200+ Integration Nodes:** Pre-built connectors for SaaS platforms, databases, APIs, communication tools (Slack, HubSpot, Salesforce, Google Sheets, GitHub, AWS, etc.)
- **AI/LangChain Nodes:** Custom nodes for AI-assisted workflows (Claude integration via `@n8n/nodes-langchain`)
- **Custom Node Development:** Extensible SDK allowing creation of domain-specific nodes
- **Webhook Triggers:** Inbound webhooks for event-driven workflow execution
- **Scheduling:** Cron-based and interval-based workflow triggers
- **Error Handling:** Fallback nodes, error routes, retry logic
- **Workflow Management:** Organization via projects, folders, tags; version control integration
- **Multi-User Support:** Workspace management, role-based access control (member/admin roles)
- **REST API:** Programmatic workflow management and execution
- **CLI Tools:** Command-line interface for node management, webhooks, workers
- **Docker Support:** Container deployment with environment configuration

## Architecture Overview

n8n is a **monorepo-based architecture with clear separation of concerns**:

### Frontend

- **`packages/editor-ui`** (Vue 3): Visual workflow editor, canvas, node palette, execution history
- **`@n8n/design-system`**: Reusable Vue component library ensuring UI consistency
- **`@n8n/i18n`**: Internationalization support for multi-language UI
- **Pinia stores**: State management for workflows, nodes, credentials, execution results

### Backend

- **`packages/cli`** (Express + TypeORM): Core API server, workflow execution, node management, user/project management
- **`packages/core`**: Workflow execution engine, context management, node orchestration
- **`packages/workflow`**: Core workflow and node interfaces
- **`packages/nodes-base`**: Built-in 200+ integration nodes
- **`@n8n/nodes-langchain`**: AI and LangChain integration nodes
- **Dependency Injection** (`@n8n/di`): IoC container for modular architecture
- **Event Bus**: Decoupled internal communication between services
- **Database**: TypeORM with SQLite (dev) or PostgreSQL (production) for workflow storage, credentials, execution logs

### Build & Development

- **Turbo Orchestration**: Manages build tasks across all packages
- **Biome**: Code formatting (replacing Prettier in recent versions)
- **ESLint**: Linting with shared configuration
- **lefthook**: Git hooks for pre-commit checks
- **TypeScript**: Full type safety with strict mode

### Testing Strategy

- **Jest + ts-jest**: Backend unit tests and node testing
- **Vitest**: Frontend unit tests
- **Playwright**: End-to-end testing with container support (fresh database per test)
- **Nock**: HTTP mocking for API testing
- **Test Isolation**: Parallel test execution with per-test fresh database via `test.use({ capability: { env: { TEST_ISOLATION: 'name' } } })`

## Integration Points

- **FiveX Platform:** n8n serves as orchestration hub connecting mod_java, mod_node, fivex_ui, tracker, and other services
- **Custom AI Nodes:** LangChain integration enables Claude, GPT, and other AI models in workflows
- **PostgreSQL:** Shared `fivex` database for workflow persistence and credential storage
- **Message Bus:** Real-time workflow triggers and notifications
- **External Services:** 200+ pre-built integrations (Salesforce, HubSpot, Slack, Google Workspace, AWS, etc.)
- **Webhook APIs:** Inbound integrations for event-driven processes

## Dependencies

**Build & Runtime:**
- Node.js 22.16+ (strict requirement)
- Pnpm 10.22.0+ (npm install blocked by preinstall check)
- Turbo build system
- TypeScript compiler

**Database:**
- SQLite (development, embedded)
- PostgreSQL 12+ (production)

**External Services:**
- Various SaaS APIs for pre-built integration nodes
- Claude/OpenAI/other AI APIs for LangChain nodes
- Webhooks/APIs consumed by workflows

## Known Risks & Technical Debt

1. **Monorepo Complexity:** 60+ packages with interdependencies make dependency management and version updates risky. Build times are substantial; Turbo helps but parallel execution can hide failures until full build.

2. **Test Maintenance Burden:** 200+ E2E tests with Playwright requires significant maintenance. The Janitor tool (test architecture enforcer) helps prevent drift, but TCR (Test && Commit || Revert) workflow requires discipline.

3. **Type Safety Enforcement:** While TypeScript is strict, `as` type casting is still permitted in test code, allowing type safety gaps. Anti-pattern documentation exists but enforcement is cultural.

4. **Build Output Verbosity:** Full monorepo build produces extremely verbose output. Mitigated by redirecting to log file (`pnpm build > build.log 2>&1`), but log inspection is manual.

5. **Package Manager Lock-In:** Preinstall check enforces pnpm and blocks npm. Migration to alternative package managers or future pnpm issues could cause friction.

6. **Credential Security:** Credentials stored in database as JSON. While supported by TypeORM, encryption-at-rest is not documented as built-in. Team should verify encryption strategy.

7. **Workflow Execution Limits:** No documented limits on workflow size, execution time, or concurrency. Production deployments should define SLOs and add rate limiting.

8. **Custom Node Distribution:** Deploying custom nodes requires code changes and rebuild. Community node marketplace exists in official n8n but distribution mechanism for FiveX custom nodes needs clarification.

9. **Database Migration Risk:** TypeORM migrations for workflow and credential schema changes require careful sequencing. Test failures during migrations can leave database in inconsistent state.

10. **Performance Profiling:** No documented baseline for workflow execution latency, memory usage under load, or horizontal scaling characteristics. Load testing recommended before production scale-up.

## Roadmap Considerations

1. **Phase 1: FiveX Custom Nodes** — Develop nodes for core FiveX services (Dynamic REST API calls, message bus publishing, Sovereign governance integration, tracker GPS queries). Package as reusable node library.

2. **Phase 2: Workflow Templates** — Create templates for common FiveX workflows (data synchronization, lead enrichment, contract generation, deployment orchestration). Share via template library.

3. **Phase 3: AI Assistant Integration** — Extend LangChain nodes to support Claude, GPT, Grok, Llama with role-specific system prompts. Enable AI-driven workflow generation and optimization.

4. **Phase 4: Governance Integration** — Wire n8n workflows to Sovereign Framework deliberation engine; allow workflows to trigger constitutional reviews; log all critical workflow decisions to audit trail.

5. **Phase 5: Scaling & Performance** — Establish performance baselines with Lighthouse/profiling tools. Implement worker-based architecture for long-running workflows. Add horizontal scaling support via Redis job queue.

6. **Phase 6: Security Hardening** — Audit credential handling. Implement encryption-at-rest for sensitive data. Add audit logging for credential access. Implement rate limiting on webhook endpoints.

7. **Phase 7: Observability** — Integrate with monitoring/observability stack (Prometheus, Grafana, or similar). Add distributed tracing for multi-service workflows. Log all execution events to centralized log system.

8. **Phase 8: Community & Ecosystem** — Publish FiveX custom nodes to n8n community marketplace. Contribute improvements back to upstream n8n. Build partner integration program.

## Success Metrics

- **Workflow Execution Reliability:** >99.9% uptime for scheduled workflows
- **Latency:** Median workflow execution < 5 seconds; P95 < 30 seconds
- **Node Coverage:** 100% of FiveX core services have custom nodes available
- **Template Adoption:** 80%+ of new workflows start from templates
- **Error Rates:** < 0.1% of scheduled executions fail due to platform issues
- **Test Coverage:** > 80% line coverage for critical packages (core, cli, nodes-base)
- **Performance Baseline:** Documented memory/CPU usage under standard workloads
- **User Adoption:** Measurable increase in workflow automation vs. manual processes across organization
