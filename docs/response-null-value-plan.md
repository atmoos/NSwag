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

- [x] Add `ResponseNullValue` to `TypeScriptClientGeneratorSettings`, type `TypeScriptNullValue`,
      **default `TypeScriptNullValue.Null`** (set default in the constructor for clarity).
- [x] Change [`ResultType`](../src/NSwag.CodeGeneration.TypeScript/Models/TypeScriptOperationModel.cs#L88)
      to read `_settings.ResponseNullValue` instead of `_settings.TypeScriptGeneratorSettings.NullValue`.
      → Kept the `optionalReturn` string mapping (the enum stringifies to "Null"/"Undefined", so it can't
      be emitted directly); only the source of the enum changed.
- [x] Keep the "`any` already includes null/undefined → no redundant union" fix (it is a genuine
      improvement and independent of this setting). → Kept; causes no extra churn (all unrelated
      snapshots reverted cleanly to `master`).
- [ ] ~~Expose the setting to templates as a rendered string helper.~~ **Deferred to Phase 3** — the
      helper isn't needed until the runtime templates consume it, and adding it now risks a naming clash
      with the `ResponseNullValue` enum property. Added in Phase 3 as `operation.ResultNullValue`
      (see the first Phase 3 step).
- [x] Wire the CLI: added a `ResponseNullValue` argument in
      [OpenApiToTypeScriptClientCommand.cs](../src/NSwag.Commands/Commands/CodeGeneration/OpenApiToTypeScriptClientCommand.cs)
      (mirrors the existing `NullValue` argument), get/set → `Settings.ResponseNullValue`.
- [x] Wire NSwagStudio: added a `ResponseNullValues` list on the view model and a bound `ComboBox`
      in `SwaggerToTypeScriptClientGeneratorView.xaml`, mirroring the existing `NullValue` dropdown.
- [x] Update `TypeScriptOperationReturnTests` to drive the new `ResponseNullValue` setting
      (instead of `TypeScriptGeneratorSettings.NullValue`). Also reverted the `[return: NotNull]`
      coupling workarounds in `TypeScriptOperationParameterTests` (+ the `NJsonSchema.Annotations` import).
- [x] Re-run tests and **regenerate/verify snapshots**. Result:
  - Unrelated Axios/Fetch/discriminator/parameter snapshots reverted to `master` (0 diff vs master).
  - The dedicated return-type snapshots keep both `nullSetting=Null` and `nullSetting=Undefined` variants.
  - **67 passed, 0 failed.** Working-tree-vs-master delta is now only: the 8 new return snapshots, the
    5 code files, and the new return tests — no unrelated churn.
- [x] **Commit** (green): "Introduce ResponseNullValue setting (type-only, default Null)". _(committed manually)_

## Phase 2 — Runtime test harness + RED commit (capture the bug)

Goal: prove the exact runtime crash the maintainer described, with tests that fail for the *right* reason.

- [x] Extend test infra to **execute** the generated client, not just compile it.
      → Added [`TypeScriptRunner`](../src/NSwag.CodeGeneration.TypeScript.Tests/TypeScriptRunner.cs): writes
      the generated code + a harness to a temp `.ts` in the test project, compiles with
      `tsc --module commonjs --target es2017 --lib es2017,dom --moduleResolution node`, runs the emitted
      `.js` with `node`, and returns stdout. Reuses the `npx`/`node` discovery pattern from `TypeScriptCompiler`.
- [x] Harness drives the **real public method through a mocked transport** (faithful to production). A
      single fake response satisfies both the Fetch shape (`status`/`headers.forEach`/`text()`) and the
      Axios shape (`status`/`headers`/`data`), so one harness serves both templates. It prints
      `__RUNTIME_RESULT__=<null|undefined|typeof>` for the .NET test to assert on.
- [x] Runtime scope: **Fetch + Axios** (deps already present). Other templates get consistency coverage
      via Phase 3 template edits + snapshots (runtime-executing them would require pulling in rxjs,
      @angular/core, jquery, aurelia — out of scope). See Decisions.
- [x] Cases covered (see [TypeScriptResponseRuntimeTests](../src/NSwag.CodeGeneration.TypeScript.Tests/TypeScriptResponseRuntimeTests.cs)):
  - [x] **Nullable DTO return** — JSON `null`, empty body, and 204 all funnel to null/undefined
        uniformly on both templates. `Undefined` → expect `undefined` (RED); `Null` → expect `null` (guard).
  - [x] **Nullable string return** — JSON `null` and 204 on both templates (the maintainer's example).
        Empty body for an Axios primitive stays `""` per the "value, not absent" decision, so it is
        intentionally **not** asserted for the string case.
- [x] Confirmed the `Undefined` cases fail **because runtime returns `null`** — every failure shows
      `Expected: "undefined" / Actual: "null"`, not a compile error or harness bug.
      → **Result: 20 runtime tests — 10 RED (all `Undefined`) + 10 green guards (all `Null`).**
- [x] **Commit** (red), clearly marked, e.g.:
      "RED: runtime tests expose null/undefined disagreement for ResponseNullValue=Undefined
      (known-failing, fixed in next commit)". Keep them unskipped so the red state is real; the marked
      commit message documents intent. _(committed manually)_

## Phase 3 — GREEN commit (make runtime consistent)

Goal: change the runtime conversion so it matches the declared type, for every supported template.

- [x] **Expose the rendered token to templates** (deferred from Phase 1). Added
      `TypeScriptOperationModel.ResultNullValue => _settings.ResponseNullValue ==
      TypeScriptNullValue.Undefined ? "undefined" : "null"` (mirrors `ResultType`, avoids clashing with
      the `ResponseNullValue` enum). `ResultType` now reuses it instead of the local `optionalReturn`.
- [x] **Coerce at the source** (final design — see the Decisions entry for the two reverted attempts).
      Helper `TypeScriptResponseModel.CoerceToResponseNullValue => ResponseNullValue == Undefined &&
      IsNullable && Type != "any"` gates a `?? {{ operation.ResultNullValue }}` appended to the
      `JSON.parse(...)` / Axios `resultData`, and the empty-body ternary emits the token, in
      [HandleStatusCode.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.HandleStatusCode.liquid).
      Plus `NullValue = Settings.ResponseNullValue` in
      [TypeScriptClientGenerator.cs](../src/NSwag.CodeGeneration.TypeScript/TypeScriptClientGenerator.cs)
      (was hardcoded `Null`). Together these make **JSON `null` and empty body** resolve to the token for
      every type / both templates, while the **Axios primitive empty string `""` stays `""`** (`"" ?? x`
      is `""`), honoring "value, not absent". No post-computation step; output is single-flow. Gated to
      `Undefined`, so `Null` mode is byte-identical to `master`.
- [x] **No-body / no-content branches**: replaced `null as any` (8×) and `return null` with
      `{{ operation.ResultNullValue }}` in HandleStatusCode.liquid, and the same (10× + `return null`) in
      [Return.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.Return.liquid).
      Text-identical in Null mode (token is `"null"`), so **zero default churn** here.
- [x] **204 / no-content resolves with the token and never throws** — the `response.IsSuccess`
      HasType-false branch and the `Return.liquid` early return both now emit the token; neither throws.
- [x] Covered all three "nothing" cases in the templates (empty body, JSON `null`, 204) per the
      Decisions section. Verified consistent across **all response-processing templates** (Fetch,
      Angular, AngularJS, JQueryCallbacks, JQueryPromises, Axios; Aurelia→Fetch) — all route through
      `Client.ProcessResponse`. K6 has no response processing (out of scope).
- [x] `Null` mode is **byte-for-byte identical** to `master` (coercion gated to `Undefined`; no-content
      token swaps render identical text since the token is `"null"`). Confirmed: churn snapshots restored
      to their pre-change content and the **full suite passes 87/87 without regeneration**.
- [x] Re-run runtime tests: previously-red `Undefined` cases now pass; `Null` cases still pass.
      → **20/20 runtime tests green.**
- [x] Regenerate/verify snapshots (runtime lines now show `undefined` for `Undefined` variants).
      → 23 snapshots updated & inspected; **full suite 87/87 green.**
- [x] **Commit** (green): "GREEN: convert response null→undefined at runtime when
      ResponseNullValue=Undefined (type/runtime now consistent)". _(committed manually; includes the (a)
      initializer fix — `let result…: any = {{ operation.ResultNullValue }}`. The (b) DataConversionCode
      indentation cleanup is deferred to a separate follow-up commit.)_

## Phase 4 — Consistency sweep & polish

- [x] Audit every template for a response-value `null` that bypasses the setting. Swept all
      `*.liquid`. Findings: the two JQuery `processX` **return-type signatures** hardcoded `| null`
      (`JQueryPromisesClient` unconditional; `JQueryCallbacksClient` gated on `HasResultType`) — fixed to
      `| {{ operation.ResultNullValue }}`. Zero churn (Null renders `| null`; no Undefined JQuery
      snapshots exist). The remaining `null`s are non-response contexts and correctly left alone:
      `throwException`'s ApiException `result` arg (error path, not the success value) and the `"null"`
      query-string literals / "cannot be null" messages in `Client.RequestUrl`/`Client.RequestBody`.
- [x] **Wrapped responses** (`WrapResponse`): confirmed by inspection that the payload follows the same
      rule — the success branch wraps the already-coerced `result` (`new ResponseClass(status, _headers,
      result{{sc}})`), and the no-body/204 branches wrap `{{ operation.ResultNullValue }} as any`. The
      wrapped result type is `ResponseClass<resultType>` where `resultType` carries the union. Note:
      there is **no `WrapResponses` test at all** (pre-existing gap) — behavior verified via code, not a
      snapshot/runtime test. Optional follow-up: add a wrapped nullable case.
- [x] Confirmed `any`/`object` returns stay unioned correctly — `ResultType` guards `not "any"`, so a
      nullable `any` renders `Promise<any>` (no `any | undefined`/`any | null`). Verified in the
      `..._is_only_any_...` snapshots.
- [x] Docs/XML comments for the new setting are in place (done in Phase 1): settings property XML doc,
      CLI `ResponseNullValue` argument description, and the Studio label + `ResponseNullValue` tooltip.
- [x] **Compile coverage across client templates** (plan omission: return tests only covered Fetch).
      Added `All_return_kinds_compile_for_client_template` — an `AssertCompile`-only `[Theory]` over a
      single `AllReturnKindsController` (nullable/non-nullable × string/object/any + void) for
      **Fetch, Axios, AngularJS, JQueryCallbacks, JQueryPromises**, each in **both** null-value modes
      (10 cases). Compile-only → no new snapshots, zero churn. Angular/Aurelia excluded (missing npm type
      deps: @angular/core+rxjs, aurelia-fetch-client) — same reason the rest of the suite doesn't compile
      them; would need those deps added to cover.
- [x] Full suite green: **97 passed, 0 failed**.
- [x] Final review of `git diff master`: the only snapshot changes are the **8 new return-test
      snapshots**; no existing snapshot churn. Code/tests limited to the intended files.

## Decisions

- **Supported templates: ALL TS templates** (Fetch, Angular, Aurelia, Axios, AngularJS,
  JQueryCallbacks, JQueryPromises). Generated TS must stay consistent across every template once the
  setting/feature is in.
- **Runtime coercion: at the source, gated to `Undefined` mode, zero default churn.**
  ~~First tried a symmetric post-computation line~~ (`result === null ? … : result` in both modes) —
  reverted: the `Null`-mode branch is a no-op producing non-sensical code and churning every
  nullable-response snapshot. ~~Then gated that post-line to `Undefined` only~~ — reverted too: it left an
  ugly two-step (NJsonSchema's conversion assigns `null`, then we reassign `undefined`).
  **Final design:** coerce the value *as it is produced*, so no post-step is needed:
  1. Flow the setting into NJsonSchema's conversion — [TypeScriptClientGenerator.cs](../src/NSwag.CodeGeneration.TypeScript/TypeScriptClientGenerator.cs)
     `NullValue = Settings.ResponseNullValue` (was hardcoded `Null`). No-op in `Null` mode.
  2. In [HandleStatusCode.liquid](../src/NSwag.CodeGeneration.TypeScript/Templates/Client.ProcessResponse.HandleStatusCode.liquid),
     the empty-body branch emits the token, and `?? {{ operation.ResultNullValue }}` is appended to the
     `JSON.parse(...)` / Axios `resultData`, gated by `response.CoerceToResponseNullValue`
     (`ResponseNullValue == Undefined && IsNullable && Type != "any"`). This converts a parsed `null`
     (which NJsonSchema's `x !== undefined ? …` guard would otherwise pass through) before conversion.
  `??` is safe here and adds no TS-version requirement — the templates already emit it (`baseUrl ?? …`).
  Result: single-flow output (e.g. `resultData = _responseText === "" ? undefined : JSON.parse(...) ??
  undefined; result = resultData;`), byte-identical `Null` mode, and only the 4 `Undefined` snapshots move.
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
- **Empty body = "value, not absent" for primitives.** The null token applies only where the value is
  genuinely null/absent (Fetch's `_responseText === ""` coercion, DTO funnels through `x ? convert : null`,
  JSON `null`, and 204). Axios delivers `data = ""` for an empty primitive body and returns it verbatim;
  that empty string is a real value and is left unchanged in **both** modes — forcing it to the token
  would change existing Null-mode Axios behavior, violating "preserve existing behavior." A nullable DTO
  return funnels all three cases to null uniformly across templates, so it is the vehicle for full
  3-case coverage; the string case is covered for JSON `null` and 204 only.
- **Runtime execution scope: Fetch + Axios** (their deps are already installed). The feature is still
  made consistent across *all* templates via Phase 3 template edits and snapshots; only the executable
  runtime proof is limited to these two, to avoid pulling rxjs / @angular/core / jquery / aurelia into
  the test project.

---

### Red/green commit summary (for reviewer legibility)

1. `Introduce ResponseNullValue setting (type-only, default Null)` — Phase 1, green.
2. `RED: runtime tests expose null/undefined disagreement …` — Phase 2, intentionally failing.
3. `GREEN: convert response null→undefined at runtime …` — Phase 3, makes (2) pass.
4. (optional) `Consistency sweep + docs` — Phase 4.
