# Plan: `ResponseNullValue` setting — type/runtime consistency for TypeScript clients

Branch: `enableTypescriptUndefinedResponse`

## Background / problem statement

The current branch reuses the existing DTO setting `TypeScriptGeneratorSettings.NullValue`
to decide whether a nullable return type is rendered as `T | null` or `T | undefined`
(see [TypeScriptOperationModel.cs:88](../src/NSwag.CodeGeneration.TypeScript/Models/TypeScriptOperationModel.cs#L88)).

Two problems flagged by the maintainer:

1. **Type/runtime disagreement.** The generated declaration can say `Promise<string | undefined>`,
   but at runtime a JSON `null` body still produces JavaScript `null`
   (`JSON.parse("null") === null`, and several template branches emit `null as any`).
   A caller that only checks `=== undefined` compiles but crashes at runtime.
2. **Wrong knob + huge test churn.** `NullValue`'s default already behaves as `Undefined`, so
   coupling the return type to it flipped ~15 unrelated snapshots (Axios/Fetch/etc.) from
   `| null` to `| undefined`. This conflates the DTO-initializer setting with response handling.

### Maintainer's requested design (alternative A)

- Add a **separate `ResponseNullValue` setting defaulting to `Null`** (preserve today's behavior).
- Opting into `Undefined` must change **both** the return type **and** the runtime response
  conversion **consistently across all supported templates**.
- Add **runtime tests** (not just compile/snapshot) proving the returned value.
- Explicitly handle **empty bodies and HTTP 204**: "no content" is distinct from JSON `null`
  and is not implied by a nullable schema.

## Key source locations

- Return type: [TypeScriptOperationModel.cs `ResultType`](../src/NSwag.CodeGeneration.TypeScript/Models/TypeScriptOperationModel.cs#L82-L104)
- Settings: [TypeScriptClientGeneratorSettings.cs](../src/NSwag.CodeGeneration.TypeScript/TypeScriptClientGeneratorSettings.cs)
- CLI command arg: [OpenApiToTypeScriptClientCommand.cs:110](../src/NSwag.Commands/Commands/CodeGeneration/OpenApiToTypeScriptClientCommand.cs#L110)
- Studio dropdown: [SwaggerToTypeScriptClientGeneratorViewModel.cs:75](../src/NSwagStudio/ViewModels/CodeGenerators/SwaggerToTypeScriptClientGeneratorViewModel.cs#L75)
- Template model (liquid surface): [TypeScriptClientTemplateModel.cs](../src/NSwag.CodeGeneration.TypeScript/Models/TypeScriptClientTemplateModel.cs)
- Runtime conversion templates:
  - [Client.ProcessResponse.HandleStatusCode.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.HandleStatusCode.liquid) — lines 41, 51, 54, 85‑105
  - [Client.ProcessResponse.Return.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.Return.liquid) — the `null as any` returns
- Tests + infra:
  - [TypeScriptOperationReturnTests.cs](../src/NSwag.CodeGeneration.TypeScript.Tests/TypeScriptOperationReturnTests.cs)
  - [TypeScriptCompiler.cs](../src/NSwag.CodeGeneration.TypeScript.Tests/TypeScriptCompiler.cs) (compile-only today; no runtime execution)

---

## Phase 0 — Baseline & safety net

- [x] Confirm the branch builds and the existing suite runs: `dotnet test src/NSwag.CodeGeneration.TypeScript.Tests`.
      → Build clean (0 warn/0 err); **67 passed, 0 failed** (~24s).
- [x] Note current snapshot delta vs `master` (`git diff master..HEAD --stat`) so we can prove
      Phase 1 reverts the unrelated churn.
      → Baseline delta: **27 `.verified.txt` snapshots changed** (~15 unrelated Axios/Fetch/etc. flipped
      to `| undefined`, plus 8 new `nullSetting=…` return-type snapshots), `TypeScriptOperationModel.cs`,
      and 2 test files (`TypeScriptOperationReturnTests.cs`, `TypeScriptOperationParameterTests.cs`).

## Phase 1 — Introduce the configuration value first (decouple, reduce churn)

Goal: default behavior becomes identical to `master`, so the ~15 unrelated snapshots revert.
No runtime behavior change yet — this is pure plumbing + wiring the **type** side onto the new knob.

- [ ] Add `ResponseNullValue` to `TypeScriptClientGeneratorSettings`, type `TypeScriptNullValue`,
      **default `TypeScriptNullValue.Null`** (set default in the constructor for clarity).
- [ ] Change [`ResultType`](../src/NSwag.CodeGeneration.TypeScript/Models/TypeScriptOperationModel.cs#L88)
      to read `_settings.ResponseNullValue` instead of `_settings.TypeScriptGeneratorSettings.NullValue`.
- [ ] Keep the "`any` already includes null/undefined → no redundant union" fix (it is a genuine
      improvement and independent of this setting). Verify whether it must stay to keep default
      snapshots clean; if it causes its own churn, capture that separately.
- [ ] Expose the setting to templates via `TypeScriptClientTemplateModel` and/or
      `TypeScriptOperationModel` as a rendered string helper, e.g.
      `ResponseNullValue => _settings.ResponseNullValue == Undefined ? "undefined" : "null"`.
      (Needed by Phase 3; add it now so the surface exists.)
- [ ] Wire the CLI: add a `ResponseNullValue` argument in
      [OpenApiToTypeScriptClientCommand.cs](../src/NSwag.Commands/Commands/CodeGeneration/OpenApiToTypeScriptClientCommand.cs)
      (mirror the existing `NullValue` argument at line 110), get/set → `Settings.ResponseNullValue`.
- [ ] Wire NSwagStudio: add a dropdown bound to a `ResponseNullValues` array, mirroring
      [`NullValues`](../src/NSwagStudio/ViewModels/CodeGenerators/SwaggerToTypeScriptClientGeneratorViewModel.cs#L75).
- [ ] Update `TypeScriptOperationReturnTests` to drive the new `ResponseNullValue` setting
      (instead of `TypeScriptGeneratorSettings.NullValue`).
- [ ] Re-run tests and **regenerate/verify snapshots**. Expected:
  - Unrelated Axios/Fetch/etc. snapshots revert to `| null` (match `master`).
  - The dedicated return-type snapshots keep both `nullSetting=Null` and `nullSetting=Undefined` variants.
- [ ] **Commit** (green): "Introduce ResponseNullValue setting (type-only, default Null)".

## Phase 2 — Runtime test harness + RED commit (capture the bug)

Goal: prove the exact runtime crash the maintainer described, with tests that fail for the *right* reason.

- [ ] Extend test infra to **execute** the generated client's `processResponse`, not just compile it.
      Options (pick the lightest that runs in CI where `npx`/node already exist per `TypeScriptCompiler`):
  - Emit a small harness `.ts`/`.js` that instantiates the generated client with a **fake
    fetch/http** returning a controlled `(status, body)`, calls the method, and prints/returns the
    resolved value; assert on it from the .NET test (compile with `tsc`, run with `node`), **or**
  - Add a focused Node test (jest/vitest or a plain node script) under the test project that the
    .NET test shells out to.
  - Decision point: prefer reusing the existing `npx`/`tsc` path in `TypeScriptCompiler` and add a
    `node` execution step returning stdout for assertions.
- [ ] Add runtime cases for `ResponseNullValue = Undefined` across **all TS templates**
      (Fetch, Angular, Aurelia, Axios, AngularJS, JQueryCallbacks, JQueryPromises) asserting the
      resolved value — TS output must be consistent everywhere once this feature lands:
  - [ ] **JSON `null` body**, HTTP 200, nullable schema → expected `undefined` (currently `null` → FAILS).
  - [ ] **Empty body** (`""`), HTTP 200 → expected `undefined`.
  - [ ] **HTTP 204 No Content** → expected `undefined` (no-content branch).
- [ ] Add the mirror cases for `ResponseNullValue = Null` asserting `null` (guards the default;
      these should pass immediately and lock in existing behavior).
- [ ] Confirm the `Undefined` cases fail **because runtime returns `null`** (inspect the assertion
      message — it must show `null` vs expected `undefined`, not a compile error or harness bug).
- [ ] **Commit** (red), clearly marked, e.g.:
      "RED: runtime tests expose null/undefined disagreement for ResponseNullValue=Undefined
      (known-failing, fixed in next commit)". Consider `[Fact(Skip=...)]`? **No** — keep them
      unskipped so the red state is real; the marked commit message documents intent.

## Phase 3 — GREEN commit (make runtime consistent)

Goal: change the runtime conversion so it matches the declared type, for every supported template.

- [ ] In [Client.ProcessResponse.HandleStatusCode.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.HandleStatusCode.liquid):
  - [ ] Empty-body ternary (lines 51, 54): `_responseText === "" ? null : ...` →
        use the configured null token (`{{ operation.ResponseNullValue }}` /
        `... as any`), i.e. `undefined` when opted in.
  - [ ] JSON `null` crash source: after `JSON.parse(...)`, **explicitly normalize** the result to the
        configured token in **both** modes — `result === null ? undefined : result` when `Undefined`,
        and `result === null ? null : result` (i.e. a visible assignment to `null`) when `Null`. This
        is the core fix — `JSON.parse("null")` returns `null` regardless of the empty-body ternary.
        The `Null` branch is intentionally explicit/symmetric rather than collapsing the already-present
        `null`, so both modes read the same way in the generated code and in the template.
  - [ ] Initializer `let result… : any = null;` (line 41) and the no-body success branches
        (`null as any`, lines 85‑105): emit the configured token.
- [ ] In [Client.ProcessResponse.Return.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.Return.liquid):
      replace the `null as any` returns with the configured token consistently for
      Fetch/Aurelia/Axios/Angular/AngularJS/default branches.
- [ ] Ensure **204 / no-content** path **resolves with the configured token and never throws**
      (distinct from body parsing); verify `HasResultType`/no-type branches at
      [HandleStatusCode.liquid:105](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.HandleStatusCode.liquid#L105).
- [ ] Cover all three "nothing" cases (empty body, JSON `null`, 204) in the templates so each maps to
      the configured token — see the Decisions section for the case definitions.
- [ ] Keep `Null` mode byte-for-byte identical to `master` (default users see zero change).
- [ ] Re-run runtime tests: previously-red `Undefined` cases now pass; `Null` cases still pass.
- [ ] Regenerate/verify snapshots (runtime lines now show `undefined` for `Undefined` variants).
- [ ] **Commit** (green): "GREEN: convert response null→undefined at runtime when
      ResponseNullValue=Undefined (type/runtime now consistent)".

## Phase 4 — Consistency sweep & polish

- [ ] Audit every template that emits a success/no-content return for a stray literal `null`
      (grep `null as any`, `= null`, `? null :` under `Templates/`); confirm none bypass the setting.
- [ ] Decide/document behavior for **wrapped responses** (`WrapResponse`) — the `result` payload
      inside the response class should follow the same rule.
- [ ] Confirm `any`/`object` returns stay unioned correctly (no `any | undefined` redundancy).
- [ ] Update docs/XML comments for the new setting (CLI help text + Studio tooltip).
- [ ] Full suite green: `dotnet test src/NSwag.CodeGeneration.TypeScript.Tests`.
- [ ] Final review of `git diff master..HEAD --stat`: only intended snapshots changed.

## Decisions

- **Supported templates: ALL TS templates** (Fetch, Angular, Aurelia, Axios, AngularJS,
  JQueryCallbacks, JQueryPromises). Generated TS must stay consistent across every template once the
  setting/feature is in.
- **Runtime normalization style: explicit and symmetric.** Use `result === null ? undefined : result`
  for `Undefined` and `result === null ? null : result` for `Null` — a visible assignment of the
  configured token in both modes, rather than collapsing the already-present `null`. Both branches
  then read identically.
  > **Review note:** `?? undefined` remains a fallback should the explicit form prove far too verbose
  > across the templates. `??` is safe (fires only on `null`/`undefined`, not `""`/`0`/`false`); the
  > explicit form is preferred purely for readability and symmetry.
- **All three "nothing" cases map to `ResponseNullValue`.** The generated code must resolve to the
  configured token (`null` or `undefined`) consistently in each of these distinct cases, matching how
  the current templates already treat them:
  1. **Empty body** — `_responseText === ""`: success status with no bytes. Not implied by a nullable
     schema; it is the transport signalling "no payload."
  2. **JSON `null` body** — `_responseText === "null"`: bytes were sent and `JSON.parse` decodes them
     to `null`. (The crash source — the `=== ""` check is false here, so the value passes straight
     through today.)
  3. **HTTP 204 No Content** — a status-code-level signal handled by a separate branch that never
     reads a body.
- **204 must not throw.** It resolves with the configured token — the empty value is representable by
  definition, so a 204 is a successful "no content," not an error.

---

### Red/green commit summary (for reviewer legibility)

1. `Introduce ResponseNullValue setting (type-only, default Null)` — Phase 1, green.
2. `RED: runtime tests expose null/undefined disagreement …` — Phase 2, intentionally failing.
3. `GREEN: convert response null→undefined at runtime …` — Phase 3, makes (2) pass.
4. (optional) `Consistency sweep + docs` — Phase 4.
