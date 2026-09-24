# Hosted validation

Hosted validation passed on 23 September 2026 (24 September UTC) in the `samples` Render workspace, using a dedicated Workflow service, web example and PostgreSQL database. Two defects found by hosted tests were fixed before the final passing run. The example used deterministic mode and made no model calls.

The fork branch is `feat/render-workflows`. Deployment configuration is confined to this package.

## Deployment and results

- Tested application commit: `d1e31106f37626076ce9e70805948742491f7cb0`.
- Workflow: `wfl-daq7jmh42hec738ka3c0`, version `wfv-daq7s3p42hec738lcmn0`, status `ready`.
- Web: `srv-daq7jrh7lnhs73bvkco0`, deploy `dep-daq7s3tg1s2s73fflkb0`, status `live`.
- PostgreSQL: `dpg-daq7hlp42hec738k2l6g-a`, PostgreSQL 17, Oregon, free test instance.
- [Hosted example](https://mastra-render-integration-test-web.onrender.com) and [web service configuration](https://dashboard.render.com/web/srv-daq7jrh7lnhs73bvkco0).
- Caller and worker use the same immutable build ID. The worker has no `RENDER_API_KEY`.

| Check | Observed result |
| --- | --- |
| HTTP contract | Health, authentication, input validation, asynchronous acceptance, duplicate ID/input, conflicting input and cross-user lookup/cancel checks passed |
| Native graph | One root and four successful children, with actual parent IDs; the three review task runs overlapped for 4.83 seconds |
| Failure and retry | Clarity review failed after two attempts; successful siblings each ran once; revision was not dispatched; root failed after one attempt |
| Native paused status | Lookup returned HTTP 200 and Mastra `running` while the native root waited for children |
| Cancellation | Immediate cancellation passed; a separate active-root cancellation canceled all three review children |
| Web restart | Render replaced instance `n78x8` with `mzvxd`; a job submitted before restart completed and was retrieved by the new backend with the same ID and result |
| Completion notification | A subscription established before completion received the native completion event |
| Local regressions | 39 tests, strict typechecking and packaged example build passed |

Key run identities:

| Scenario | Mastra run ID | Render root ID |
| --- | --- | --- |
| Success | `be38ef29-76d5-43ff-a764-6c5cd6d112d7` | `trn-08l4gdaq7t75g1s2s73ffp8i0` |
| Failure and retry | `7806765c-a4a9-4e4d-9e65-6bcadd10f0b7` | `trn-08l4gdaq7tb142hec738lhme0` |
| Immediate cancellation | `da6b3033-fbb5-4e9b-aa86-197576d287bf` | `trn-08l4gdaq7teh7lnhs73c0ptjg` |
| Active cancellation | `c3980916-2713-47a7-86ec-ba14c41436c3` | `trn-08l4gdaq7tlp7lnhs73c0qnig` |
| Backend restart | `f7dd6599-3926-4c1a-ba03-48b6acc2b447` | `trn-08l4gdaq7uap7lnhs73c0stdg` |

[Machine-readable observations](hosted-results.json) retain native child IDs and attempt counts. An additional diagnostic subscribed only after completion and received no historical event within 30 seconds. Do not assume event replay; the provider checks status before subscribing and falls back to polling after a bounded wait.

## Test environment lifetime and remaining limits

The temporary caller API credential expires on **25 September 2026 at 16:59 UTC**. Replace `RENDER_API_KEY` on the web service and redeploy before relying on ongoing submissions. The worker needs no such credential. The demo login uses a tester token from the web service's `DEMO_API_TOKENS`; tokens are deliberately absent from the repository and this report.

The free PostgreSQL test instance expires on **24 October 2026**. The free web service can sleep when idle. Auto-deploy is disabled on both services so later documentation commits do not change the tested application version. This is a temporary test deployment, not a production service.

These checks do not prove durable root replay, hosted root crash/timeout recovery, deploy transitions during active runs, workspace quota behavior, ambiguous submission recovery, production authentication/retention, paid model behavior or full Studio compatibility. Root retries remain zero, as documented in the package README.

## Reproducible build

Set the service root directory to `workflows/render` and use:

```sh
bash scripts/build-editorial-example.sh
```

The script builds and packs the provider, installs the example's dependencies and then installs the actual provider archive into the example. It does not install the monorepo workspace or rely on a published integration package.

Workflow run command:

```sh
node examples/editorial-review/node_modules/tsx/dist/cli.mjs examples/editorial-review/worker.ts
```

Web start command:

```sh
node examples/editorial-review/node_modules/tsx/dist/cli.mjs examples/editorial-review/server.ts
```

Both processes require matching `APP_BUILD_ID`, `RENDER_WORKFLOW_SLUG` and `DATABASE_URL`, and `REVIEW_MODE=deterministic` for tests without model calls. Set `NODE_VERSION=24.18.0` and `TSX_DISABLE_CACHE=1`. Do not set `RENDER_USE_LOCAL_DEV` for hosted operation.

The web service additionally needs `HOST=0.0.0.0`, a Render-assigned `PORT`, `DEMO_API_TOKENS` with strong random values, and `RENDER_API_KEY` for task submission and lookup. The worker uses the native task context to chain tasks and does not need a management API key. `/healthz` checks the HTTP process; it is not proof that database access or workflow submission works.

Credentials belong in the Render environment and ignored local test configuration. Do not commit connection strings, tokens or populated environment files. The demo's configured bearer tokens remain a demonstration authentication scheme.

## Repeat the application checks

Set `DEMO_BASE_URL` to your deployed example, `DEMO_TEST_TOKEN` to one configured user's bearer token, and `DEMO_OTHER_TOKEN` to a different user's token. Optionally set `DEMO_RESULTS_FILE` to an ignored output file. Then run from the package directory:

```sh
TMPDIR="$PWD/.scratch/tmp" TSX_DISABLE_CACHE=1 \
  node node_modules/tsx/dist/cli.mjs scripts/hosted-smoke.ts
```

This creates three real jobs. It checks health, authentication, validation, asynchronous acceptance, duplicate submission, input conflicts, owner isolation, deterministic output, child failure and cancellation. It refuses agent mode. Job IDs are printed before submission so an interrupted test can reconnect without blindly resubmitting.

## Defect found during hosted validation

The initial deployment at `c708b97a51ec17ce2bbbfdaf57ebb6084e81e2e0` registered successfully, but its first root failed before dispatching children. Mastra's `createRun` calls `getWorkflowRunById` while hydrating the worker run. The adapter unnecessarily reconciled that lookup through the management API, requiring a worker API key.

The fix associates the active handler with its Mastra workflow/run identity and reads the persisted binding for that exact worker-local lookup. Caller lookups still reconcile authoritative Render status. Native child dispatch continues through the task context, and the worker requires no management API credential. The regression fails before the fix and passes afterward.

The next hosted graph completed successfully but polling exposed an unhandled native `paused` status while the root awaited its children. The adapter now treats that status as an active Mastra `running` execution and preserves a pending cancellation. This does not add Mastra suspend/resume. A second regression covers lookup and cancellation through this state.
