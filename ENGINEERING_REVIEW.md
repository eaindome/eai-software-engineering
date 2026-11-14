# Engineering Review Playbook

> **Purpose:** A reusable, evolving review guide for human engineers and coding agents. Place this file at the repository root. Use it to review a feature *as implemented*, *as a business operation*, and *as part of the wider system*.
>
> **Core principle:** Don't merely ask whether the happy path works. Ask what must always remain true, what assumptions may be wrong, how the operation can fail, and how the system recovers.

**Version:** 1.1  
**Scope:** Backend-first, applicable to frontend and full-stack features  
**Review modes:** Feature review, pull-request review, production-incident follow-up, pre-release review

---

## 0. Review operating rules

1. **Understand before judging.** Read the feature's requirements, relevant code, schemas, tests, callers, and downstream consumers. Do not invent business rules.
2. **Separate facts from hypotheses.** Cite concrete code locations and observed behavior. Label unverified risks as hypotheses requiring investigation.
3. **Review in five passes.** Implementation defects → logical correctness → system-wide impact → assumptions and missing rules → failure and recovery.
4. **Prioritize by impact.** Security, financial correctness, data loss, privacy, and integrity outrank cosmetic concerns.
5. **Do not silently change business policy.** Record gaps and proposed options for the responsible decision-maker.
6. **Do not make unrelated edits.** During review, recommend changes first; implement only when authorized.
7. **Preserve confidentiality.** Do not put production personal data, secrets, or sensitive records in review reports or agent prompts.
8. **Aim for justified confidence, not perfection.** State what was checked, what was not checked, and residual risks.
9. **Distinguish pre-existing issues from introduced ones.** Tag all findings `[Introduced]` or `[Pre-existing]`. Do not block a change for pre-existing debt it did not create.

## 1. Establish the feature's contract

Before reviewing, answer:

- **Purpose:** What user/business problem does this feature solve?
- **Actors:** Who can initiate, approve, cancel, or observe it?
- **Entry points:** Which UI actions, APIs, jobs, events, or integrations invoke it?
- **Inputs and outputs:** What data enters and what results are promised?
- **States:** What are the valid states and allowed transitions?
- **Dependencies:** Which records, services, and workflows precede or follow it?
- **Critical invariants:** What must *always* be true, even under retries or concurrency?
- **Authority:** Which requirements are explicit, and which need product/domain approval?

If key facts are unknown, list the questions before proposing implementation changes.

---

## Phase 0 — Feature map (human-gated)

Run this phase **once per codebase**, before any pass-level review begins. Its purpose is to prevent shallow coverage by forcing explicit ordering and scoping before reviews start.

**Step 1 — Discovery (agent):** Scan the codebase and enumerate all features. For each, record: name, entry point(s), and a one-sentence purpose. Present the full list to the human reviewer.

**Step 2 — Ordering (human):** The human organizes features into a review sequence:

- **Main flow:** the core path through the system in the order operations actually happen (e.g. Onboarding → Computation → Benefit Audit → Payroll). Review these first, in dependency order — findings from an upstream feature give context for downstream ones.
- **Supporting features:** features that feed into or extend the main flow (e.g. Change Requests, Dependant Applications, Gratuity Recomputation). Review these after the main flow, grouped by which main-flow step they touch.
- **Risk flags:** the human marks any feature as elevated risk (financial impact, complex state machine, security-sensitive) to signal deeper pass coverage for that feature.

**Step 3 — Sequential review:** Reviews proceed one feature at a time, in the agreed order. Each feature gets its own invocation of the full five-pass review. Do not collapse multiple features into one prompt — breadth kills depth.

---

## Pass 1 — Implementation defects

**Question:** Is the existing code technically sound?

Check:

- [ ] Incorrect conditions, operators, null handling, boundary checks, or off-by-one errors
- [ ] Exceptions, unhandled failures, unsafe casts, and inconsistent error responses
- [ ] Input parsing, type conversions, date/time handling, precision and rounding
- [ ] Incorrect queries, joins, filters, mappings, pagination, and missing persistence calls
- [ ] Async/await mistakes, cancellation, timeouts, resource leaks, and blocking operations
- [ ] Dead code, duplicated logic, inconsistent naming, and unclear control flow
- [ ] Missing or misleading tests; code paths that tests never exercise
- [ ] Performance pitfalls (N+1 queries, unnecessary materialization, unbounded work)
- [ ] Dependency/configuration assumptions that fail outside local development

If a stack-specific companion file exists for this codebase, apply its checklist here.

**Output:** Reproducible bugs with file/line references, expected vs actual behavior, and suggested verification.

## Pass 2 — Logical and business correctness

**Question:** Does the feature enforce the *right* rules, not merely execute successfully?

Check:

- [ ] Valid and invalid input combinations, including relationships across fields
- [ ] Required fields, optional fields, defaults, and domain-specific constraints
- [ ] Date ordering, age/tenure rules, timezones, inclusive/exclusive deadlines
- [ ] Financial calculations, rounding, currency, units, and approved formulas
- [ ] State transitions: permitted, forbidden, repeated, and reversed actions
- [ ] Can every reachable state exit via at least one valid transition? Map dead-end states explicitly.
- [ ] Does the implementation match any existing design document or specification? If they diverge, establish which is authoritative before proceeding.
- [ ] Authorization for the *specific resource*, not just authenticated access
- [ ] Uniqueness and referential integrity at the database level where appropriate
- [ ] Idempotency: repeated requests do not duplicate business effects
- [ ] Concurrent requests cannot violate invariants (atomic updates, constraints, locking)
- [ ] Validation exists at the trusted backend boundary, not solely in the UI
- [ ] Error messages are safe, meaningful, and actionable
- [ ] Business rules are traceable to approved requirements or documented decisions

**Output:** Violated invariants, missing rules, and a proposed decision table or state-transition matrix.

## Pass 3 — Whole-system impact

**Question:** How does this feature interact with everything around it?

Trace the feature **backward** (what leads here) and **forward** (what this triggers).

- [ ] Upstream eligibility, prerequisites, approvals, and data ownership
- [ ] Downstream records, calculations, notifications, reports, documents, and payments
- [ ] Shared entities changed by multiple features or administrators
- [ ] Cross-module consistency and compatibility with existing workflows
- [ ] Transaction boundaries and integration/event delivery guarantees
- [ ] Auditability: actor, timestamp, before/after state, reason, correlation ID
- [ ] Data migrations, legacy records, and backward compatibility
- [ ] Access controls across API, background jobs, exports, and download endpoints
- [ ] Operational effects: monitoring, support tooling, reporting, and user guidance
- [ ] Whether a local fix introduces a regression elsewhere

**Output:** Dependency/impact map and regression-test targets.

## Pass 4 — Challenge assumptions and discover gaps

**Question:** What has everyone assumed without confirming?

Actively ask:

- What if the user clicks the wrong action, double-clicks, refreshes, or uses two tabs?
- What if the UI is stale and the underlying record has changed?
- What if two users or jobs update the same record at once?
- What if a valid-looking field combination is impossible in the business domain?
- What if an administrator overrides, withdraws, reopens, or corrects an action?
- What if a request bypasses the frontend entirely?
- What if the user has incomplete information, misunderstands terminology, or follows a different real-world workflow?
- What if a record is missing, duplicated, historical, migrated, or partially completed?
- What if a rule varies by role, programme, effective date, or policy exception?
- What if the requirement describes only the happy path?
- Are there codebase-specific documented patterns that look unusual but are intentional? Check decision logs, architecture notes, or convention documentation before flagging as a defect.

For every gap, record **the question, plausible scenarios, risk, options, and the decision owner**. Never treat an unapproved option as established policy.

**Output:** Clarification register and explicit assumptions.

## Pass 5 — Failure, recovery, and resilience

**Question:** What happens when something goes wrong, halfway through or afterward?

- [ ] Partial failures do not leave inconsistent or irrecoverable business state
- [ ] Retries and timeouts are safe; side effects are idempotent where required
- [ ] Duplicate messages, reordered events, and at-least-once delivery are handled
- [ ] Concurrent decisions resolve deterministically without invalid states
- [ ] Failed external calls have suitable retry, reconciliation, or compensation paths
- [ ] User mistakes have an approved correction/undo/reinstatement workflow if needed
- [ ] Recovery does not require routine manual production database edits
- [ ] Logs preserve diagnostic detail while public errors reveal no sensitive internals
- [ ] Alerts and correlation IDs help support staff investigate incidents
- [ ] Recovery operations are authorized, audited, and tested
- [ ] Backups, rollback plans, and data repair approaches exist for high-impact changes

**Output:** Failure-mode table, recovery plan, and operational follow-ups.

---

## Cross-cutting risk lenses

Apply these to **every pass**, scaled to the feature's risk:

| Lens | Key question |
| --- | --- |
| Security | Can an unauthorized actor read, change, approve, or export this resource? |
| Privacy | Could data or raw errors expose another person's information? |
| Integrity | Can duplicates, races, or partial writes break business invariants? |
| Financial correctness | Could this cause an incorrect amount, entitlement, or payment? |
| Usability | Could a reasonable user misunderstand the action or make a recoverable mistake? |
| Reliability | What happens under timeout, retry, outage, or stale state? |
| Observability | Can support reconstruct what happened without guessing? |
| Performance | Will the design hold at realistic volume and concurrency? |
| Maintainability | Are rules centralized, understandable, and testable? |

## Severity and confidence

| Severity | Meaning |
| --- | --- |
| **Critical** | Credible exposure of sensitive data, unauthorized financial action, severe corruption, or major production outage |
| **High** | Important business rule broken, incorrect financial outcome, unauthorized operation, or serious data-integrity risk |
| **Medium** | Meaningful workflow failure with a workaround, inconsistent behavior, or operational burden |
| **Low** | Minor defect, maintainability issue, or limited usability concern |

For each finding also mark **Confirmed** (reproduced or proven from code) or **Potential** (plausible but unverified). Severity reflects impact; confidence reflects evidence.

## Required review output

Use this structure for human reviews and agent reports:

```markdown
# Review: <feature / PR>
Date: <YYYY-MM-DD>
Scope: <files, modules, workflows>
Review mode: <feature | PR | incident | release>

## Feature understanding
- Goal:
- Actors:
- Entry points:
- States/transitions:
- Invariants:
- Upstream/downstream dependencies:

## Findings
### F-001 — <concise title>
- Pass: <1–5>
- Severity: <Critical | High | Medium | Low>
- Confidence: <Confirmed | Potential>
- Status: <[Introduced] | [Pre-existing]>
- Evidence: <file:line, query, reproduction, or test>
- Scenario: <how it occurs>
- Impact: <business/system/user consequence>
- Recommendation: <specific corrective action>
- Verification: <test or observation proving resolution>

## Missing business decisions
| Question | Why it matters | Options | Decision owner |
| --- | --- | --- | --- |

## Regression tests to add
- [ ] <test and expected behavior>

## Review coverage and limitations
- Examined:
- Not examined:
- Tests run and results:
- Assumptions:
- Residual risks:

## Suggested next actions
1. <highest-impact action>
```

## Example scenarios from real-world workflow patterns

These are **illustrations**, not universal business rules:

### Sponsorship: accepting an offer creates a contract

- **Invariant:** If policy permits only one contract per application, enforce it with a database unique constraint on `ApplicationId` (adjust for versioned/reissued contracts).
- **Retry:** Double-clicking or retrying must not create a second contract or repeat downstream side effects.
- **Identifier:** Contract numbers must be unique if used as unique lookups; generation must be concurrency-safe.
- **Access:** Downloading a contract must verify the requesting user's entitlement, even if the contract number is unique.
- **Recovery:** An accidental rejection may require an approved reinstatement workflow, with an audit trail rather than routine direct database edits.

### Pension management: date relationships

- Validate birth, employment, and retirement dates **together**, not only as valid date strings.
- Check ordering, permitted employment ages, retirement rules, policy exceptions, and effects on existing calculations.
- Do not invent statutory thresholds; obtain the approved rules from the domain owner.

### Error handling

- Return stable, safe user-facing errors; keep detailed diagnostics in protected logs.
- Distinguish expected validation failures from unexpected exceptions.
- Do not confuse hiding a raw error with fixing the underlying reliability problem.

---

## Agent instructions (read before every automated review)

You are acting as a **reviewer, not an autonomous policy-maker**.

1. Read this file and the repository's architecture, conventions, and relevant requirements.
2. Summarize the feature and its invariants before reporting defects.
3. Review one feature at a time, in the human-specified order from Phase 0. Do not attempt multiple features in a single invocation — breadth kills depth.
4. Trace callers, persistence, authorization, and downstream effects when repository access permits.
5. Prefer evidence-backed, high-impact findings. Include exact paths and line numbers when available.
6. Never claim a test passed unless it was actually run. Never invent code locations or requirements.
7. Distinguish confirmed bugs from hypothetical risks and unresolved business decisions.
8. If a stack-specific companion file exists for this codebase, apply it alongside Pass 1.
9. Do not edit code, migrations, or production data unless explicitly instructed.
10. Do not expose secrets or sensitive user data in output.
11. Report coverage limitations and the most valuable next tests.

**Suggested agent prompt:**

> Review `<feature/path/PR>` using `ENGINEERING_REVIEW.md`. First establish the business workflow and invariants. Then perform all five passes, prioritizing security, integrity, concurrency, and failure recovery. Produce the required review output with evidence and actionable regression tests. Do not modify files. Flag any missing business rules for human decision.

## Review completion checklist

- [ ] Feature purpose, actors, and dependencies understood
- [ ] All five review passes completed or limitations declared
- [ ] Critical invariants identified and checked
- [ ] Authorization and data isolation reviewed
- [ ] Retry, concurrency, and partial failure considered
- [ ] Negative, boundary, and regression tests identified
- [ ] Business-policy gaps assigned to decision owners
- [ ] Findings prioritized with evidence and confidence
- [ ] Residual risks and review limitations recorded

---

## Evolution log

Keep this file alive. When a production incident teaches a reusable lesson, add the general principle, review question, and regression-test pattern **without copying confidential incident data**.

| Date | Version | Change | Lesson / trigger |
| --- | --- | --- | --- |
| 2026-11-13 | 1.0 | Initial five-pass review playbook | Move beyond happy-path code review |
| 2026-11-14 | 1.1 | Phase 0 feature map; dead-end state check; design doc divergence check; intentional-pattern guard in Pass 4; pre-existing vs introduced tagging (Rule 9); one-feature-per-invocation agent rule; stack companion reference | CPMS Management Portal review cycle |

### Future enhancements

- Domain-specific appendices (pensions, sponsorships, payments, identity)
- CI automation and reusable test templates
- Architecture-specific failure-mode catalogs
- Team review metrics and incident-derived regression library
