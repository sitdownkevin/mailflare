# This installation

- Installation repository: https://github.com/sitdownkevin/mailflare
- Upstream: https://github.com/hieunc229/mailflare
- Worker name: `mailflare`
- Intended email domain: `849238.xyz`

## Upstream updates

`Check upstream updates` runs daily at 01:23 UTC (09:23 China time), and can also be run from GitHub Actions with **Run workflow**. It opens a PR from upstream `main` into this repository's default branch when commits are missing. An existing PR is reused and follows later upstream commits. It does not reset branches, execute upstream code, deploy, or migrate the production database. No personal access token or Cloudflare secret is required.

GitHub repository Settings → Actions → General must allow GitHub Actions to create pull requests. The workflow itself grants only Contents read and Pull requests write. Scheduled workflows in inactive public repositories can be disabled by GitHub after 60 days; re-enable the workflow from Actions if this occurs. Scheduled execution times are approximate.

Review each update, especially `wrangler.jsonc`, `package-lock.json`, `.github/workflows/`, and `drizzle/migrations/`. Check or approve validation runs in the PR. Obtain a current database backup before applying migrations. Resolve any conflicts without discarding local configuration. Merge with **Create a merge commit**, not squash/rebase, to preserve ancestry and prevent repeated update proposals.

The inherited `Dashboard Update` workflow (`deploy-update.yml`) directly merges and migrates production before pushing. It is not the recommended update path for this installation. Do not configure the optional application `GITHUB_UPDATE_TOKEN` unless intentionally using that separate path. Upstream documentation currently calls it `update.yml`, but the actual application dispatches `deploy-update.yml`.

## Deployment requirements

Mailflare uses Next.js/OpenNext on Workers, D1, R2, inbound/outbound Queues, a realtime Durable Object, Email Routing, Email Sending, and a daily backup cron. Deploy `worker.ts`, which wraps the generated Next.js worker and handles email and queue events. Pages alone is insufficient.

Keep Worker name, self-reference service, and email routing destination equal to `mailflare`. Store runtime `CF_TOKEN` as a Cloudflare secret scoped to the intended account/domain; never commit it. The deployment credential and runtime credential have different purposes. `CF_TOKEN` is used to provision email routing/DNS at runtime. Optional Turnstile requires both a build-time public site key and a runtime secret.

R2 must be enabled. Receiving and outbound sending have different plan requirements: consult Cloudflare Email Service pricing before enabling a paid subscription. A custom email domain must be active in the same Cloudflare account.

For Git-connected deployment, connect this repository's `main` branch to Workers Builds, install with `npm ci`, and use `npm run deploy` as the deploy command (leave the separate build command empty to avoid building twice). Supply the actual D1 database ID through the installation's Cloudflare/Wrangler configuration before migrations. Build first, migrate second, deploy the complete Worker last. Do not run both Workers Builds and another automatic deploy pipeline for the same branch.

Before first deployment run:

```sh
npm ci
node --test tests/*.test.mjs
npm run lint
npx opennextjs-cloudflare build
```

The upstream application currently suppresses TypeScript errors during Next builds. A successful build alone is not proof of type correctness. After deployment, complete `/setup` and create the admin account immediately, then connect the domain and test incoming/outgoing mail. Later upgrades require D1 migrations; the first-run setup is not an upgrade mechanism. A Worker code rollback does not roll back D1 schema changes.
