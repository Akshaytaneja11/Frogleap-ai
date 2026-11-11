# FrogLeap 0.1 — Repository Audit, Architecture, and Implementation Plan

**Status:** Planning baseline  
**Product direction:** Consumer-first AI productivity workspace  
**Initial platform:** Web  
**Audience:** Everyone, with an approachable default experience and progressive disclosure for advanced users  
**Model strategy:** Multi-provider  
**Commercial model:** Freemium  
**Base:** DeepSeek Harness (DSH), upstream `deepseek-ai/deepseek-harness`, MIT-licensed repository snapshot

---

## 1. Executive recommendation

Build FrogLeap as a distinct, consumer-facing web product on top of DeepSeek Harness, not as a rebrand of the upstream application and not as a fork that rewrites the agent runtime.

Reuse DSH for the capabilities it already provides: plugin composition, agent/session lifecycle, model adapters, tool execution, file handling, web tools, settings, credentials, and deliverables. Put FrogLeap-specific experience and commercial policy in a separately owned product layer and profile/bundle composition.

The critical architectural gap is not chat or model access. It is **multi-user SaaS ownership**: authentication, tenant isolation, server-side credential custody, usage entitlement enforcement, billing, abuse controls, and operational support. These must be designed before a public multi-user launch. DSH's local, per-harness-home credential and session model must not be treated as a complete SaaS tenancy system.

**0.1 goal:** a reliable web MVP where a user can chat, select a model/provider, upload and analyse files, run grounded web research, and use a guided decision workflow. Keep document/presentation generation and autonomous multi-step agents as follow-on releases unless existing capability proves easy to expose safely.

---

## 2. Repository audit

### 2.1 Repository identity and state

- Repository: `Akshaytaneja11/Frogleap-ai`
- Confirmed fork of `deepseek-ai/deepseek-harness`.
- Default branch: `master`.
- Audited HEAD: `e8f7b93a344964de3dca1ace9e72b6e4d6de87cb`.
- Upstream snapshot version in root package: `0.2.2-rc.2`.
- Root package manager: pnpm `11.28.5`; Node engine: `>=24.9.0`.
- Root workspace spans `packages/*/*`, `apps/*`, native packages, vendor packages, Python SDK runtime, and website.
- Fork currently has one branch (`master`) and no open pull requests or issues.
- GitHub reports admin and push permission for the connected owner account.
- The current HEAD has no combined status checks reported by GitHub. This is not evidence that the code fails; it means this audit has not established a passing CI run for the fork.

### 2.2 Relevant existing capabilities

| Area | Existing building blocks | Implication for FrogLeap |
|---|---|---|
| Web client | `apps/web`, `packages/client/web`, client UI packages | Reuse the existing web shell and extend/replace branded surfaces deliberately |
| Composition | `packages/bundle/base`, `packages/bundle/web-app`, Cordis profiles and patches | Add FrogLeap-specific bundle/profile composition; avoid patching the agent loop |
| Agent and sessions | Core agent, agent-loop, session event log and persistence packages | Reuse turn lifecycle, history, streaming and durable session model |
| Multi-provider models | `dsh-llm-pi-ai`, DeepSeek adapters, provider settings and model-selection UI | Provider-neutral model routing is already a strong base |
| Credentials | Credential seam and local file-backed provider | Useful for local development; not sufficient as the SaaS tenant vault |
| Tools and research | Web search/fetch, filesystem, attachment, skills, jobs, workflows | Compose selected tools into a restricted consumer-facing experience |
| Deliverables | Deliverables and office/skill-related packages | Investigate for later exportable reports and documents |
| Metering | Token-meter package is present in the workspace | Assess as an input to server-side usage accounting; do not assume it implements plan enforcement or billing |
| Identity | Anonymous per-harness-home identifier | Not equivalent to customer accounts, login, organisations, or tenant ownership |
| API transport | Typert Gateway, remotes, remote-access checks | Useful RPC foundation; still needs an authenticated tenant/principal boundary |

### 2.3 Important findings and risks

1. **Upstream is a developer preview.** The README explicitly warns of compatibility-breaking changes. Pin the upstream base and establish a deliberate upstream-sync process.
2. **The runtime is highly composable.** DSH's documented architecture says new capabilities should be mounted through Cordis services, events, bundles, and profile patches. Avoid direct agent-loop changes.
3. **Current identity is installation-scoped and anonymous.** No first-party consumer account/authentication product was identified in the reviewed identity package.
4. **Credential storage is local-host oriented.** The local provider stores secrets under a harness home. Its own documentation warns that agent tool processes run as the same OS user, so it cannot isolate secrets from the agent. Never expose this local credential model as a multi-tenant SaaS vault.
5. **Provider flexibility already exists.** The pi-ai adapter supports multiple providers and custom OpenAI-compatible routes with credential references. Build FrogLeap's provider UX and server-side policy around this rather than inventing a new model protocol.
6. **Usage metering is not commercial entitlement.** Token accounting, plan limits, quotas, concurrency caps, and payment status must be enforced by trusted server-side services.
7. **The Web UI is coupled to a Host/runtime.** A public SaaS deployment needs an explicit deployment and tenant-lifecycle design; simply putting the existing local Web UI behind a public URL is not sufficient.
8. **Security is a launch gate.** DSH can execute model-generated commands, load plugins, access files and network resources. Public multi-user deployment must use strong isolation, least privilege, narrow tool allowlists, quotas, audit trails, and independent security review.

### 2.4 Audit scope and limitations

This is a static repository review of the root instructions, architecture/development/user guides, root and selected package manifests, bundle composition, credential/model-provider documentation, and CI workflow configuration. No source files were changed during the audit. No dependency install, build, unit test, browser test, deployment, or penetration test was run. The fork's current CI status is unverified. Detailed implementation work must inspect the exact source and tests for each selected extension point before editing it.

---

## 3. Product definition for 0.1

### Primary promise

**FrogLeap helps people move from questions to useful outcomes in one approachable AI workspace.**

### MVP jobs-to-be-done

1. Ask questions and have a useful, streaming conversation.
2. Choose an available AI model/provider without exposing provider complexity to beginners.
3. Upload a document and ask questions or request a summary/analysis.
4. Research a topic on the web and receive sourced findings.
5. Work through a structured decision: goal, options, criteria, trade-offs, risks, and recommendation.

### 0.1 navigation

- Home — universal prompt entry and suggested tasks.
- Assistant — conversations and history.
- Research — web research with source references.
- DecisionLab — guided decision workflow.
- Projects — saved workspaces and related conversations (only if tenancy/persistence design supports it).
- Settings — profile, model preferences, connected providers, privacy, and usage.

### Explicitly out of scope for 0.1

- Autonomous agents that can take consequential external actions without user approval.
- Public plugin marketplace or arbitrary third-party plugin installation.
- Team/enterprise administration and organisation workspaces.
- Mobile native apps.
- Complex document/presentation studio.
- Full workflow builder.
- Enterprise SSO, SCIM, and advanced compliance certifications.

These remain future opportunities, not MVP dependencies.

---

## 4. Target architecture

### 4.1 Logical layers

```text
FrogLeap Web Experience
  Home | Assistant | Research | DecisionLab | Projects | Settings
                    |
FrogLeap Product API / Control Plane
  Auth | Tenant Context | Entitlements | Usage | Audit | Billing
                    |
FrogLeap Runtime Adapter / Orchestrator
  User-to-runtime mapping | Session ownership | Tool policy
                    |
DeepSeek Harness Runtime (isolated per tenant or isolated execution unit)
  Cordis profile + FrogLeap bundle/patches
  Agent | Sessions | LLM adapters | Tools | Attachments | Deliverables
                    |
Providers and Infrastructure
  Model APIs | Search | Object storage | Database | Queue | Observability
```

### 4.2 Design rules

- Keep FrogLeap-owned code in clearly named packages/apps and avoid editing upstream-owned packages unless an audited extension gap requires it.
- Prefer DSH plugin seams, bundle composition, profile patches, and documented APIs.
- Put authentication and tenant resolution at the trusted server boundary. Never trust a tenant ID supplied only by the browser.
- Every session, attachment, project, usage record, credential, and deliverable must be associated with an authenticated owner/tenant and checked on every access.
- Use a managed server-side credential vault for FrogLeap-provided model access. Encrypt secrets at rest with a managed key service; redact them from logs, telemetry, traces, errors, and client payloads.
- If users bring their own API keys, store them encrypted per account and route requests through a server-side provider adapter. Do not persist keys in browser localStorage or expose them to model tools.
- Do not let a tenant's agent process read another tenant's files, credentials, environment, session store, or runtime home.
- Use isolated runtime workers/containers or an equivalent reviewed isolation boundary. A shared process with only UI-level tenant filtering is not acceptable.
- Apply plan entitlements before dispatch and enforce limits again at the tool/runtime boundary.
- Use an explicit provider/model allowlist and per-provider timeout, retry, spend, and rate limits.
- Require user approval for external side effects and high-impact operations. Default 0.1 to read-only research and document analysis tools.

### 4.3 Recommended deployment topology

For early private alpha, use one control-plane web service plus isolated runtime workers with tenant-scoped storage. For public launch, scale workers independently and ensure each runtime's filesystem, credentials, network policy, and session persistence are isolated. A tenant-to-runtime mapping service should own lifecycle and routing.

Avoid assuming that a single shared local DSH home can safely serve unrelated customers.

---

## 5. 0.1 implementation roadmap

### Stage 0 — Architecture and baseline (1–2 weeks)

- Pin the upstream commit and record the sync strategy.
- Create a FrogLeap-owned package/app namespace and dedicated web profile/bundle.
- Document the supported Node/pnpm setup and local developer workflow.
- Map the existing web shell, model settings, credentials, attachments, web search, and deliverables to concrete extension points.
- Establish lint, typecheck, focused tests, build, and preview deployment gates.
- Produce a threat model and data-flow diagram before multi-user data is introduced.

**Exit criteria:** a clean checkout builds and launches an unmistakably FrogLeap-branded local web experience without breaking the upstream default profile.

### Stage 1 — Consumer web foundation (2–3 weeks)

- Build a responsive Home and navigation experience.
- Implement account registration/login, session management, logout, and account recovery through a selected auth provider.
- Define tenant/user IDs and a server-derived request context.
- Add database migrations for users, tenants, projects, sessions mapping, usage events, and entitlements.
- Implement server-side session ownership checks and secure file/object-storage access.
- Add privacy, terms, data deletion, and account export paths.

**Exit criteria:** two test accounts cannot read or mutate each other's conversations, files, settings, or usage.

### Stage 2 — Assistant and multi-model routing (2–3 weeks)

- Reuse DSH's provider-neutral LLM service and pi-ai provider adapter.
- Provide a beginner-friendly model picker with recommended/default options and an advanced provider settings view.
- Support platform-managed model access first; add encrypted BYOK credentials as an optional advanced feature.
- Add streaming, conversation history, rename/archive/delete, stop generation, retries, and clear provider errors.
- Add server-side provider allowlists, spend ceilings, timeouts, and rate limits.

**Exit criteria:** supported providers pass contract tests; secrets never reach browser storage, model-visible context, or logs; conversation ownership is enforced.

### Stage 3 — Files and research (2 weeks)

- Add upload validation, file-size/type limits, malware scanning where appropriate, private object storage, and retention policy.
- Reuse DSH attachment and file-analysis capabilities behind tenant-aware access controls.
- Add research workflow with search/fetch tools, citations, source list, and clear distinction between source-backed facts and model synthesis.
- Limit web and file tools to the least privilege needed for the task.

**Exit criteria:** file analysis and research work end-to-end; citations resolve; cross-tenant access tests fail closed.

### Stage 4 — DecisionLab (1–2 weeks)

- Build a guided flow: define decision → capture options → select criteria/weights → compare evidence → surface risks/assumptions → recommendation.
- Show transparent scoring and allow users to edit criteria and weights.
- Keep the recommendation explainable; do not present a model-generated score as objective certainty.
- Save/export a decision brief.

**Exit criteria:** deterministic scoring tests pass; model reasoning is separated from arithmetic; user can inspect and change assumptions.

### Stage 5 — Freemium, quality, and private beta (2–3 weeks)

- Define Free and paid entitlements (message/compute budget, file limits, research limits, model access, concurrency).
- Implement usage ledger, quota reservation, final usage reconciliation, idempotency, and abuse controls.
- Add billing provider integration only after entitlements and usage accounting are stable.
- Add observability, error reporting, support tooling, backup/restore, data deletion, and incident procedures.
- Run security review and load tests; deploy a limited private beta.

**Exit criteria:** quota bypass tests fail closed; usage records reconcile; billing webhooks are idempotent; backup/restore and tenant isolation are tested; no unresolved critical/high security findings.

**Planning range:** approximately 10–14 weeks for a small, focused team, subject to discovery and security work. This is an estimate, not a delivery commitment.

---

## 6. Freemium policy proposal

Do not hard-code plan rules into the UI or DSH plugin configuration. Store plan entitlements in the product control plane and enforce them server-side.

| Capability | Free | Paid (initial hypothesis) |
|---|---|---|
| Chat | Limited monthly usage | Higher monthly allowance |
| Models | Curated standard models | Broader model selection |
| Research | Small monthly allowance | Larger allowance |
| File analysis | Small file/usage limits | Larger limits |
| DecisionLab | Core guided workflow | Higher usage and export options |
| History | Standard retention | Extended retention |
| BYOK | Optional, subject to support/security decision | Optional |

Exact quotas, prices, provider cost assumptions, and fair-use limits should be chosen after instrumented alpha usage. Avoid promising unlimited AI usage.

Usage enforcement should account for provider/model cost, tokens, tool execution time, file processing, and concurrency—not only message count.

---

## 7. Quality, security, and release gates

### Functional
- New user onboarding and account recovery.
- Streaming assistant responses and cancellation.
- Provider selection and provider failure handling.
- Conversation persistence and deletion.
- File upload, analysis, and access control.
- Research with verifiable citations.
- DecisionLab scoring and export.

### Security and privacy
- Tenant isolation tests for every data access path.
- Encrypted credentials and rotation/revocation.
- No secrets in browser storage, logs, telemetry, or prompts.
- Runtime filesystem and process isolation.
- SSRF protections for URL fetches and web tools.
- File type/size validation and safe parsing.
- Rate limits, spend caps, abuse detection, and audit logs.
- Account deletion and retention enforcement.

### Engineering
- Focused unit and integration tests for each feature.
- Typecheck and lint.
- Build artifacts and production build checks.
- Browser end-to-end tests for critical journeys.
- Dependency and license review.
- Backup/restore rehearsal.
- Deployment rollback procedure.

No production launch should be declared solely because the UI works locally.

---

## 8. First engineering tasks

Recommended order for the first implementation sprint:

1. Create a dedicated FrogLeap development branch and preserve upstream `master`.
2. Inspect the full source and tests for the Web shell, bundle composition, model settings, credentials, attachments, and web search.
3. Add a FrogLeap-owned bundle/profile and branding entry points.
4. Build the first FrogLeap Home shell on the existing client framework.
5. Prototype the multi-provider selection path with at least two provider types using test credentials.
6. Draft the tenant/auth/storage control-plane interfaces and prove isolation with two test tenants before exposing any shared runtime.
7. Add CI checks for FrogLeap-owned packages and a reproducible local run guide.

### Decision gates still required before implementation

- Authentication vendor and account lifecycle.
- Hosting/cloud and primary database/object storage.
- Whether 0.1 launches as private single-user alpha first or multi-user private beta.
- Which managed model providers are enabled at launch and the initial cost ceilings.
- Whether BYOK is included in 0.1 or deferred.
- Billing provider and target launch geography.

---

## 9. Upstream and brand stewardship

- Keep a recorded upstream remote/commit and periodically compare upstream changes.
- Do not silently merge upstream changes into a release branch; run the agreed compatibility and regression suite.
- Preserve upstream license and third-party notices.
- Describe the relationship truthfully as “built on DeepSeek Harness” where relevant.
- Do not use DeepSeek's marks or presentation in a way that suggests official endorsement.
- Give FrogLeap its own name, logo, visual system, product copy, domain, privacy terms, and support identity.

---

## 10. Audit conclusion

The repository is a strong agent-runtime foundation for FrogLeap's multi-model assistant and tool workflows. It is not, by itself, a complete consumer SaaS platform. The highest-risk and highest-value early work is the FrogLeap web experience plus the authenticated multi-tenant control plane and isolated runtime boundary.

**Recommendation:** proceed with Stage 0, but do not begin public multi-user rollout until identity, tenant isolation, credential custody, quota enforcement, and runtime security have been demonstrated and reviewed.
