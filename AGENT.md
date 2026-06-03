Binding rules for AI + engineers. Conflict with this file → STOP and ask.
Legend: ✅ required · ❌ forbidden · ⚠️ anti-pattern · 🛑 hard-stop. When unsure → stricter, simpler option.

## 1. Principles
- Dependencies point inward: UI → Service → Domain (domain depends on nothing).
- One module = one reason to change. KISS. DRY only after 3rd repetition. YAGNI.
- ❌ Framework types crossing layers. ❌ Hidden global/mutable state.

## 2. Java (Spring Boot)
Layers: Controller → Service → Repository (each calls only the one below).
- Naming: classes `PascalCase`, members `camelCase`, const `UPPER_SNAKE`. Suffix by role: `*Controller/*Service/*Repository/*Request/*Response/*Mapper/*Exception`. Package by feature.
- Controller: map HTTP↔DTO, `@Valid`, call one service, return DTO. ❌ logic/loops/DB/entities.
- Service: ALL business logic, rules, orchestration; DTO/domain only. ❌ HTTP types.
- Repository: persistence only. ❌ logic, DTO mapping, calling other repos/services.
- ✅ Entities never cross controller boundary. `@Transactional` on service (`readOnly` for reads).
- ❌ `@Transactional` on controller/repo/self-invoked. ❌ swallow exceptions, exceptions as control flow, `500` for business errors.
- ✅ Typed domain exceptions → one `@RestControllerAdvice`. Constructor injection + `final`.
- ❌ Field injection, `Util/Helper/Manager` dumping grounds, returning `null` (use `Optional`).
- ⚠️ Fat controller, anemic service, N+1.

## 3. React
- Functional components + hooks only; small, single-purpose; TypeScript strict.
- Naming: components/types `PascalCase`, hooks `useXxx`, vars `camelCase`. One component/file, named exports. Feature-based co-located folders (`components/ hooks/ services/ types.ts`).
- ✅ Split presentation (props in) from logic (custom hooks). Hooks at top level; full deps arrays.
- ✅ Network only via `services/api` + React Query/SWR. Local state first; global store only for auth/theme.
- ❌ Business logic in UI (pricing/tax/eligibility = backend). ❌ `fetch`/`axios` in components. ❌ side effects in render, `useEffect` for derived state (use `useMemo`), mutate props, store derived/duplicated server data, `any`, class components.
- ⚠️ useEffect waterfalls, deep prop drilling.

## 4. Salesforce (Apex + LWC)
Apex layers: Trigger → Handler → Service → Selector(SOQL)/Domain. Assume 200+ records always.
- Naming: `*TriggerHandler/*Service/*Selector/*Domain/*Test`; custom fields `__c`. One trigger/object. Declare `with sharing` on every class.
- ✅ Bulk query/DML; pre-load into `Map`, look up in loop. Logic-free, recursion-safe triggers.
- ❌ SOQL/DML in loops. ❌ hardcoded IDs (use Custom Metadata). ❌ logic in trigger body. ❌ `without sharing` unless justified.
- LWC: data down via `@api`, events up via `CustomEvent`, `@wire` for data. ❌ mutate `@api`, business rules in JS (push to Apex), cross-component DOM access.
- ⚠️ SOQL/DML in loop, recursive trigger without guard, `@api` mutation.

## 5. API & Integration
- One envelope: `{ success, data, error:{code,message,details}, timestamp }`.
- Status: 400 validation · 401/403 auth · 404 missing · 409 conflict · 422 business · 500 unexpected. Centralize mapping.
- ❌ Stack traces/SQL to clients, ad-hoc error shapes.
- Java↔React: versioned REST `/api/v1`, typed contracts from OpenAPI.
- SF↔Backend: Named Credentials, idempotent, retry-safe, heavy work async. ❌ sync callouts in triggers.

## 6. Testing
- Java: JUnit5+Mockito (services), `@WebMvcTest`/`@DataJpaTest`; test edge/failure paths.
- React: RTL behavior tests, mock API layer. ❌ implementation-detail tests.
- Salesforce: >75% coverage floor with real assertions; bulk tests (200+ records); data in-test.
- ❌ Coverage-only tests without assertions.

## 7. Security
- Validate/sanitize server-side (allow-list). Parameterized queries / bind vars only.
- Authn + authz on every endpoint; FLS/sharing in Apex (`WITH SECURITY_ENFORCED`). Secrets in vault/Named Credentials.
- ❌ String-built SQL/SOQL, secrets/PII in code or logs, `dangerouslySetInnerHTML` on untrusted data, UI hiding as access control.

## 8. Performance
- Java: no N+1 (fetch join/`@EntityGraph`), paginate, select needed columns.
- React: prevent re-renders (stable keys, `useMemo`/`useCallback` where measured), virtualize lists, code-split.
- Salesforce: selective SOQL on indexed fields, async for heavy work, no overfetching.

## 9. AI Guardrails

### 9.1 Pre-flight (before coding)
Answer all four, else STOP and ask:
- Restate task + affected layers. - Verify every API/field/type/schema exists. - Find existing pattern to follow. - List unknowns/assumptions.

### 9.2 Self-check (before returning code) — fix or flag, don't ship failures
- All symbols verified (no invented). - Boundaries respected. - Matches patterns/naming, no unapproved deps.
- Handles errors/nulls/edge/bulk. - Security: validated, authz, no secrets, no string SQL/SOQL. - In scope. - Assumptions/`TODO: verify` stated.

### 9.3 Hard-stop — refuse/ask, never guess
🛑 Symbol can't be verified · must invent contract/schema/config · forced boundary violation · new dep without approval · ambiguous/contradictory request · changes a public contract/schema/security control unauthorized.
Output: the blocker + the question needed. Never fill gaps with guesses.

### MUST / MUST NOT
- ✅ Reuse patterns/naming/utilities; read before writing; ask when uncertain; explicit > magic; stay in scope; state assumptions.
- ❌ Invent APIs/fields/schema/config; hallucinate schema; add frameworks without approval; cross boundaries; use reflection/metaprogramming over explicit code; change contracts/schema silently; rewrite out-of-scope code.
- Rule: can't point to where a symbol is defined → don't use it.

## 10. Review Checklist (blocking)
- [ ] Thin controllers; logic in services; repos/selectors persistence only; DTOs at boundary.
- [ ] No direct fetch / no business logic in React/LWC; SF one trigger/object, SOQL in selectors.
- [ ] `@Transactional` on service; no SOQL/DML in loops; Apex bulkified; no N+1; reads paginated.
- [ ] Server validation + authz everywhere; FLS/sharing enforced; no secrets/PII; standard envelope + status.
- [ ] Pre-flight done; self-check (§9.2) passed; no invented symbols; no unapproved deps; scope respected; assumptions stated.
- [ ] Meaningful tests with assertions (behavior, bulk-safe for SF).
- [ ] Explicit readable code; no dead code, no leftover `console.log`/`System.debug`.

Final rule: stricter, simpler, more explicit — and ask. Boring maintainable code > clever code that breaks at scale.
