# Upstream integration follow-up

Implementation base: upstream `mastra-ai/mastra` commit `ac5ece3a410222ea704d9d10cb6e18ce937680b3`, on local branch `feat/render-workflows` in the user's existing fork `ojusave/mastra`.

All implementation changes are inside `workflows/render/`. The root workspace already includes `workflows/*`. No core, server, sibling provider, root example, root lockfile, root changeset, shared CI or shared documentation was changed. Repository metadata for cloning/remotes/branch setup was separately authorized by the user's fork-and-clone request.

The package uses a reproducible published dependency baseline: Mastra core 1.67.0, Render SDK 1.1.0, Mastra PostgreSQL 1.25.0, Zod 3.25.76 and TypeScript 5.9.3. Local checks against those packages do not establish compatibility with unpublished workspace core. Core source was read to understand public extension points; the adapter does not import private source paths or copy the core engine.

Before upstreaming or publishing:

1. Agree with Mastra/Render maintainers on package ownership, final package name and support contract. The local package is private and its name is provisional.
2. Verify against the intended Mastra release/workspace core. Broaden peer ranges only with evidence. Run the repository's applicable provider conformance suite once shared-test wiring is authorized.
3. Add the root changeset, workspace lockfile/catalog changes, shared documentation navigation and CI matrix under a separately expanded edit boundary. The normal repository changeset instruction was deferred because the user explicitly restricted implementation changes to this package.
4. Extend the [hosted smoke evidence](hosted-validation.md) with deploy/redeploy during runs, root timeout/process loss, workspace rate pressure, infrastructure failure, submission connection loss and real model calls. Passing smoke checks do not establish production reliability.
5. Decide whether durable root replay is a requirement for adoption. This version intentionally fails after root loss. Adding root retries safely needs a separate replay/idempotency design and failure tests, not merely a retry setting.
6. Add an operator reconciliation path for submissions whose provider ID was never persisted. Current behavior prevents a blind duplicate but cannot automatically discover the accepted root.

The package is implemented, pushed to the user's fork and tested on hosted Render. See `hosted-validation.md` for evidence and defects fixed by those tests. No package publication, upstream PR or paid model request is included.
