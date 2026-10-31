# Sanitised validation evidence — AeglysAI qualification (pintu544)

Environment: Linux (x86_64), .NET SDK 8.0.427 (repository `global.json` requests
8.0.204 with `rollForward: latestFeature`). No Docker daemon was available, so
the direct .NET route from the README was used. All machine-identifying paths
have been replaced with `<home>`. Collected on 31 October 2026.

| File | Contents |
| --- | --- |
| `restore.log` | `dotnet --version` (8.0.427) and `dotnet restore` — exit 0, 11 projects restored. |
| `build.log` | `dotnet build` — succeeded, 0 warnings, 0 errors. |
| `test-solution.log` | `dotnet test` on the full solution — exit 1; every test project aborted before executing any test (see note below). |
| `test-domain-diagnostic.log` | Single-project diagnostic rerun (`VSTEST_CONNECTION_TIMEOUT=240 dotnet test` on AeglysAI.Domain.Tests with `--diag`) — same abort. |
| `endpoints.log` | Live API run: `GET /health`, anonymous `POST /authorization/evaluate` (HTTP 401), and the repository DevTools demo (Models A/B/C for `reports/read`, then `reports/write`). |
| `missing-context.log` | Second live run with an extra local fixture grant for `reports/export` (RBAC + ABAC configured, no risk context): Model A ALLOW, Model B ALLOW, Model C DENY `MissingRiskContext` with score 1.00 and level Unknown. |

## Note on the test execution failure

`dotnet test` did not execute any tests in this environment, and no pass/fail
result is claimed. In both the full-solution run and the single-project
diagnostic run, `vstest.console` reported that it "failed to connect to
testhost process" (after 90 seconds, and after 240 seconds in the diagnostic
run) and the run was aborted. The diagnostic log shows the test host process
did start and connect, but the first protocol message it received failed JSON
parsing (`Newtonsoft.Json.JsonReaderException: Unexpected character
encountered while parsing value`), after which the channel closed with an
end-of-stream error — that is, the test platform's console-to-host handshake
failed inside this sandbox's networking, rather than any test being run and
failing. The failure is environmental; it is recorded here rather than worked
around or misreported.
