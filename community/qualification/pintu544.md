# AeglysAI qualification technical evidence

- **Public fork URL, qualification branch URL and exact commit:**
  - Fork: https://github.com/pintu544/AeglysAI
  - Qualification branch: https://github.com/pintu544/AeglysAI/tree/qualification/pintu544
  - Branched from `main` at commit `ff3dbc7bcaecde6f279925f6b25307ff52af7ec7`
    ("Merge pull request #16 from shubh-techie/research/AEGLYSAI-DATA-002-dataset-validation").

- **Architecture: explain Domain, Application, Infrastructure and API boundaries:**

  I read the solution as a textbook Clean Architecture layout, and the
  architecture tests pin the dependency contract so it cannot quietly rot.
  `AeglysAI.Domain` depends on nothing and holds only the vocabulary of the
  problem — `Subject`, `Role`, `Permission`, `Resource`, `Action`, `AbacRule`,
  `AttributeSet` and `RiskContext` — as small immutable types with their own
  invariants (for example, `RiskContext` rejects undefined signals and values
  outside 0–1, and treats an absent signal as unknown, never as zero).
  `AeglysAI.Application` depends only on Domain and holds the behaviour: the
  three authorisation engines, the ABAC evaluator, the contextual risk engine,
  and the interfaces (`IAuthorizationEngine`, `IAbacEvaluator`,
  `IRbacPolicyProvider`, `IRiskContextProvider`, `ICurrentIdentity`) that the
  outer layers implement. `AeglysAI.Infrastructure` depends on Application and
  Domain and currently supplies only in-memory providers for the RBAC policy,
  the ABAC context and the risk context. `AeglysAI.Api` is the composition
  root and the only project that references Infrastructure: `Program.cs` wires
  dependency injection, JWT bearer authentication, the configuration binding
  for the `Rbac`, `Abac` and `ContextualRisk` sections, and the HTTP
  endpoints.

  A decision flows like this: a request arrives with a JWT; authentication
  validates it and `HttpCurrentIdentity` exposes the caller's issuer/subject
  pair through `ICurrentIdentity`; the endpoint builds an
  `AuthorizationRequest` from that validated identity plus the requested
  resource and action (never from request-supplied identity data); an
  observation wrapper records one structured audit event per completed
  evaluation; the engine chain runs — Model B first asks Model A, Model C
  first asks Model B — and the endpoint renders the decision, including the
  inner RBAC/ABAC results and, for Model C, the risk contributions, as JSON.

- **A/B/C baseline understanding:**

  Model A is a flat, positive-grant RBAC baseline. Roles and subject
  assignments come only from trusted server configuration; token claims grant
  nothing. Matching is ordinal and case-sensitive, with no wildcards,
  hierarchies or implicit administrator. If the subject has no assigned roles
  the answer is DENY (`NoAssignedRoles`); if no role grants the exact
  resource/action permission it is DENY (`NoMatchingPermission`); otherwise
  ALLOW (`PermissionGranted`), always with the policy version. An absent
  `Rbac` configuration section is an explicit empty policy, not a demo grant,
  so the default posture denies everything.

  Model B layers ABAC on top and can only narrow Model A: a Model A denial
  ends the evaluation. Otherwise the evaluator collects every ABAC rule whose
  resource and action match — no applicable rules is itself a DENY
  (`NoApplicableRules`) — and then checks all conditions of all applicable
  rules against subject, resource and environment attributes that come from
  server configuration, never from the request. A missing attribute denies
  with `MissingAttribute`, and that reason deliberately takes precedence over
  a value mismatch (`ConditionNotSatisfied`); only when every condition is
  satisfied does Model B allow. One consequence I noted while reading
  `AbacEvaluator`: because the conditions of all applicable rules are
  flattened and AND-ed together, two rules for the same resource/action
  cannot act as alternatives — a second rule only ever tightens the first.

  Model C adds a deterministic, fixed-weight risk score after a Model B
  allow; a baseline denial is returned untouched (`BaselineDenied`) and the
  risk provider is not even consulted. Five signals are weighted
  (authentication weakness 0.25, resource sensitivity 0.25, operation
  sensitivity 0.20, network exposure 0.15, device exposure 0.15), and the
  bands are: score below 0.25 → ALLOW (`LowContextualRisk`); below 0.50 →
  STEP_UP with a "verify stronger authentication and re-evaluate" obligation;
  below 0.75 → LIMIT with a rate-limit obligation of 10 requests per minute;
  otherwise DENY (`HighContextualRisk`). If any signal is missing, the level
  is Unknown and Model C denies with `MissingRiskContext` regardless of the
  numeric score. I confirmed the arithmetic against the live fixture: the
  example context (0, 0.5, 0.5, 0.5, 0) scored exactly 0.300 and returned
  STEP_UP. STEP_UP and LIMIT are recommendations with obligations, not
  implemented enforcement, and the whole A/B/C stack is an experimental
  baseline — the endpoints evaluate decisions without executing protected
  actions.

- **Setup and .NET 8 prerequisites:**

  Prerequisites are Git, a POSIX shell and a .NET 8 SDK (the repository's
  `global.json` requests SDK 8.0.204 with `rollForward: latestFeature`; I used
  SDK 8.0.427). I took the direct .NET route from the README —
  `dotnet restore`, `dotnet build`, `dotnet test`, then
  `dotnet run --project src/AeglysAI.Api -- --urls http://localhost:5080` —
  because no Docker daemon was available in my environment, so the `./dev`
  Compose route could not be used. To exercise the authenticated endpoints I
  used the repository's own DevTools project
  (`dotnet run --project deployment/docker/AeglysAI.DevTools -- init <dir>`),
  which generates the synthetic, Development-only local trust configuration
  and a one-hour token for the fictional subject `synthetic-developer`. I
  placed the generated file as `appsettings.Development.json` for the run and
  removed it afterwards; no source files were changed at any point.

- **One observation:**

  When Model C has no risk context for an otherwise allowed request, the
  denial is reported with a numeric score of 1.00 — the maximum — even though
  nothing was measured. The mechanism is in `ContextualRiskEngine.Assess`: a
  missing signal contributes its full weight to the score with the reason
  `MissingContextConservativeMaximum`, and the assessment's level becomes
  Unknown, which `ModelCAuthorizationEngine` turns into DENY
  `MissingRiskContext`. I verified this live: with a local fixture that lets
  Models A and B allow `reports/export` but configures no risk context for
  it, Model A returned ALLOW and Model B returned ALLOW, while Model C
  returned DENY / `MissingRiskContext` with `"score": 1.00`,
  `"level": "Unknown"` and all five contributions flagged
  `MissingContextConservativeMaximum`. The behaviour is deliberate and pinned
  by unit tests (for example `Any_missing_signal_is_unknown_and_denied` and
  `Entirely_missing_context_is_unknown_with_maximum_score`), and the
  fail-closed outcome is, in my view, correct. My observation is about the
  presentation: in the API response — and, following SPEC-008, in the audit
  event — that substituted worst-case number is indistinguishable in shape
  from a measured maximum-risk score. A consumer that reads `score` without
  `level`, or a dashboard that averages scores, would read "we had no
  evidence" as "the riskiest request we have ever seen". Limitations of this
  observation: I observed it on the synthetic fixture configuration only, and
  through the API only for the fully missing case; the partially missing case
  follows the same code path (any missing signal makes the level Unknown) and
  is covered by the repository's theory tests, but I did not exercise it over
  HTTP.

- **One proposed improvement:**

  Surface evidence completeness explicitly in the Model C result, without
  changing any decision. Concretely: add a top-level marker to the model-c
  API response and to the audit event — for example a `contextComplete`
  boolean plus the list of missing signal names — populated from data the
  engine already computes (each contribution already carries a null value and
  the `MissingContextConservativeMaximum` reason). The engineering value is
  that operators and downstream consumers can distinguish "denied because
  evidence was absent" from "denied because measured risk was high" without
  reverse-engineering the contributions array, which matters for exactly the
  audience an authorisation audit trail exists for. It is deliberately a
  surfacing change only: outcomes, scores, reasons and obligations stay
  identical, so the pinned decision semantics in the existing tests are
  untouched. Validation: new Application-level tests asserting the marker for
  complete, partially missing and fully missing contexts; integration tests
  over the model-c endpoint for the documented fixture (complete context →
  STEP_UP with the marker set to complete) and for an allowed-but-unprofiled
  action such as my `reports/export` fixture (→ DENY `MissingRiskContext`
  with the marker set to incomplete and all five signals listed); and a full
  `dotnet test` run showing no decision, score or reason changed anywhere.
  This is a proposal only — I have not implemented it.

- **Validation results: exact restore/build/test commands, environment, actual outcomes and limitations:**

  Environment: Linux (x86_64), .NET SDK 8.0.427, no Docker daemon; repository
  at commit `ff3dbc7bcaecde6f279925f6b25307ff52af7ec7`. All commands were run
  from the repository root.

  - `dotnet restore` — exit code 0; all 11 projects in the solution restored.
  - `dotnet build` — succeeded: "Build succeeded. 0 Warning(s), 0 Error(s)".
  - `dotnet test` — **did not execute in this environment; I claim no test
    results.** The command exited with code 1 after every test project
    (ArchitectureTests, Domain.Tests, Application.Tests, Experiments.Tests,
    IntegrationTests) aborted before running a single test:
    `vstest.console` reported it "failed to connect to testhost process after
    90 seconds" and the run was aborted. I investigated once, with a
    single-project diagnostic run
    (`VSTEST_CONNECTION_TIMEOUT=240 dotnet test tests/AeglysAI.Domain.Tests/AeglysAI.Domain.Tests.csproj --diag:<log>`);
    it aborted the same way after 240 seconds. The diagnostic log shows the
    test host process did start and connect, but the first protocol message
    it received failed JSON parsing
    (`Newtonsoft.Json.JsonReaderException: Unexpected character encountered
    while parsing value`), after which the channel closed with an
    end-of-stream error — the test platform's console-to-host handshake
    failed inside this sandbox's networking rather than any test running and
    failing. Zero tests were executed, so there are honestly no passed/failed
    counts to report, and the working-baseline evidence below rests on the
    successful build and the live API run instead. This is the main
    limitation of my qualification run, and it is environmental: the same
    commands are the repository's documented validation route.
  - Live API run (Development environment, synthetic local trust fixture
    generated by the repository's DevTools):
    - `GET /health` → `{"status":"healthy"}`.
    - `POST /authorization/evaluate` without a token → HTTP 401.
    - With the fixture token, via the repository's demo harness (exit code
      0): Model A on `reports/read` → ALLOW (`PermissionGranted`, policy
      `example-model-a-v1`); Model B → ALLOW (`ConditionsSatisfied`); Model C
      → STEP_UP (`AdditionalAssuranceRequired`, score 0.300, level Medium,
      obligation `VerifyStrongerAuthenticationAndReevaluate`); Model A on
      `reports/write` → DENY (`NoMatchingPermission`).
    - Second run, with an extra local grant so Models A and B allow
      `reports/export` and no risk context configured for it: Model A ALLOW,
      Model B ALLOW, Model C DENY (`MissingRiskContext`, score 1.00, level
      Unknown) — the run behind my observation above.
  - Working-baseline summary: restore and build fully succeed and the API
    demonstrably runs and decides as documented; the automated test suite
    could not be executed in this environment for the reason given, and I
    have not presented its results as passing.

- **Sanitized build/run/test evidence location:**

  Sanitised logs (machine-identifying paths replaced) are retained by me and
  submitted with the private qualification form: `restore.log`, `build.log`,
  `test-solution.log` (the aborted full-solution run), `test-domain-diagnostic.log`
  (the single-project diagnostic rerun), `endpoints.log` (health, anonymous
  rejection and the A/B/C demo), `missing-context.log` (the
  `reports/export` run), and a short `README.md` explaining the test-platform
  failure with the diagnostic excerpt. My registration ID and GitHub username
  are provided in the private qualification form only, as instructed.
