# qm adoption assessment — FCA Group

Decision record. Assessed against FCA Group's needs at under 10 users, weighted on cost to
run, fit with the existing tool stack, setup and maintenance effort, and what adoption would
actually improve.

## What this repository is

`Vegamus/Company-Harnes` is an unmodified clone of qm, Y Combinator's open-source
multiplayer agent harness for companies (`yc-software/qm`, MIT, `src/`, `plugins/`, `cli/`).

Two facts about this checkout:

- `deploy/layers/` contained only `README.md` before this document — nothing
  organisation-specific has been added. This is a vanilla copy, not a customised instance.
- `git log` is 40 commits dated 2026-07-29 to 2026-07-31, starting from a commit titled
  "Fresh repo history". Zero tags, empty `adrs/`, version `0.1.0` in `package.json`. The
  public project has cut no releases.

## Recommendation

**Do not adopt as the company agent platform at this time.**

The engineering quality is high — higher than the project's age suggests. The blocker is
fit, not quality: qm cannot talk to most of the tool stack the business runs on, and closing
that gap means rebuilding integrations by hand and then carrying a private fork of a pre-1.0
project with no release track record.

A narrow, time-boxed pilot on one non-sensitive brand is defensible. Committing group
operations to it today is not.

## 1. Fit with the existing tool stack — the decisive issue

qm has no third-party MCP support. The only MCP code in `src/` is
`src/harness/claude-harness.ts:373`, which uses `createSdkMcpServer` to expose qm's own
tools to the Claude SDK in-process with `strictMcpConfig: true`.
`src/harness/harness.ts:152` declares an `"mcp"` transport in a type union that nothing
implements. External MCP servers cannot be connected.

FCA Group's day-to-day stack is MCP-based. The OAuth connector registry (`PROVIDERS`,
`src/connectors/oauth.ts:215`) has exactly seven entries: `google`, `slack`, `notion`,
`linear`, `dropbox`, `github`, `x`.

| Currently in use | Available in qm |
| --- | --- |
| Gmail / Google Drive / Calendar | Yes — `google` |
| Notion | Yes — `notion` |
| GitHub | Yes — `github` |
| Slack | Yes — `slack` |
| HubSpot | No |
| Brevo | No |
| Apollo.io | No |
| Meta Ads | No |
| Jotform | No |
| Canva | No |
| Contentsquare, Vercel | No |

qm's prescribed substitute is a skill that shells out to `curl` in the sandbox. That is real
work per integration — auth, pagination, error handling, rate limits — reimplemented and
then owned in-house, replacing typed tools available today at no cost.

Skills themselves port well. qm skills are `SKILL.md` with YAML frontmatter
(`src/skills/frontmatter.ts`), the same shape as the existing library, so
`skilled-worker-visa-toolkit`, `seven-academy-cpd-course`, `classbridge-offers` and similar
carry over largely as-is. The exception is any skill that calls an MCP tool — which is
exactly the marketing set (Meta Ads, Brevo, Apollo, HubSpot).

Roughly a third of the stack carries over; two thirds needs rebuilding.

## 2. Cost to run at under 10 users

Nothing scales to zero. From the `fly.toml` templates: five always-on services totalling
7.5 GB of `shared-cpu-1x` RAM — core 2 GB (`deploy/core/fly.toml`), web-ui 2 GB, admin
2 GB, portal 1 GB, auth 512 MB. Portal, auth and web-ui set `min_machines_running = 1`
explicitly. Two egress proxies add 256 MB and 512 MB, for roughly 8.25 GB always on before
any token is billed.

Also required: Fly Managed Postgres (`fly mpg create --plan basic`,
`cli/src/backends/fly.ts:459`) and a Tigris object-storage bucket.

**Sandbox disk is the line item to price before anything else.** qm runs one persistent
microVM per *scope*, not per user. Scopes are `personal | channel | team | org | group`
(`src/types.ts:12`), so eight people plus ten Slack channels or projects is around eighteen
sandboxes. The Sprites profile declares `diskGb: 100` each
(`src/sandbox/sprites-sandbox.ts:318`) and the whole disk persists while idle — roughly
1.8 TB of persistent disk.

Compounding this: `reapDeepIdle` is implemented only in `src/sandbox/aws-sandbox.ts:486`.
The Sprites backend has no implementation, so on Fly nothing garbage-collects abandoned
scopes and the 3-day `DEEP_IDLE_MACHINE_MS` default (`src/config.ts:399`) never fires.
`SECURITY.md` separately records that file artifacts have no expiry and byte reclamation is
not implemented.

Obtain a written price for sandbox disk before committing. Sprites' billing model could not
be verified from the repository, and this is the number most likely to surprise.

Model spend is the other dominant cost, billed direct by the provider. The repository's own
catalog (`src/model/pi-models.ts`) prices the default `claude-opus-5` at $5 in / $25 out per
Mtok. Guardrails ship in `.env.example`: `BUDGET_USD_PER_WINDOW=25` per user per day and
`ORG_BUDGET_USD_PER_WINDOW=100` per day — an approximately $3k/month ceiling if fully
consumed. Set it lower on day one.

At under 10 users this is materially more expensive than the current setup, and the
infrastructure portion is fixed regardless of usage.

## 3. Setup and maintenance effort

Standing it up means 8–10 separately provisioned components on Fly: five service apps,
Postgres, object storage, a sandbox registry app, plus the egress proxy that the default
"Auto" security posture's protections depend on. That proxy is not deployed by the CLI and
is not part of the documented `qm up` flow; without it `src/config.ts:573` prints that
sandboxes run with no egress enforcement and continues.

The `qm init` → `check` → `doctor` → `secrets push` → `up` flow is well built and
agent-assisted, but the manual gates are irreducible: cloud account and billing, nine or
more generated secrets, a Slack app with 22 bot scopes created by hand, an OAuth app
registered per connector, and DNS/DKIM for sign-in email — which the CLI's own reference
material calls the step most likely to stall an otherwise-autonomous deploy. Budget half a
day to two days for a clean Fly deploy with no connectors.

Day-2 operations carry the larger cost, and land internally given there is no dedicated
platform engineer:

- No deploy CI, by design. `docs/deploy-directory.md`: "No workflow in the qm source
  repository deploys a production stack."
- No schema migrations. 38 `CREATE TABLE` statements apply as lazy idempotent DDL under an
  advisory lock (`src/persistence/pg-pool.ts:42`). No version table, no down-migrations.
- No external monitoring or alerting. Observability is an in-process admin page; there is no
  OTel, Prometheus, Datadog or Sentry integration.
- Manual restore. `qm rollback` restores code and configuration only, then prints the RDS
  snapshot command to run separately.
- Code contributions are not accepted. `CONTRIBUTING.md` asks for human-written prose in
  `adrs/`, which the maintainers implement if they agree. Every needed integration therefore
  becomes a permanent private fork with ongoing merge discipline. `adrs/` is currently
  empty, so that channel has never been exercised.

## 4. Data protection

MetropolVisa handles passports and immigration case files; Medkay handles health and
recruitment data. Both fall under UK GDPR special-category or identity-document handling.
`SECURITY.md` line 32 states qm is "not a hardened public or multi-tenant service boundary"
and line 3 calls it "early, experimental software". Specifics verified in code:

- No erasure or DSAR path. The only bulk delete, `resetUserToBrandNew`
  (`src/api/routes/admin/users.ts:294`), removes personal-scope sessions only — file
  artifacts, blobs, memory, keychain, Slack mirror bodies and audit survive.
  `/v1/admin/retention` is DAU/WAU/MAU growth analytics, not data retention.
- Verbatim prompt capture is on by default. `session_llm_requests`
  (`src/sessions/postgres-session-store.ts:214`) stores every model request in a plain text
  column, indefinitely.
- Egress is unrestricted by default. The shipped org policy is
  `{ allowedHosts: [], deniedHosts: [] }` (`src/resolution/config-store.ts:295`), and an
  empty allowlist means allow-everything (`src/resolution/egress-policy.ts:77`).
- The command policy has one hard denial — `mkfs` and fork bombs
  (`src/policy/command-policy.ts:5-24`). Nothing blocks exfiltration.
- No posture provides both approvals and screening. `strict` is
  `{ inboundScreening: "off", toolApprovals: "all" }`; `auto` is
  `{ inboundScreening: "external", toolApprovals: "none" }`
  (`src/security/security-posture.ts:14-18`).
- All deploy templates set `primary_region = "sjc"` with no region abstraction in code, and
  prompts go to Anthropic, OpenAI or OpenRouter — all US. Each is a sub-processor requiring
  a DPA and a transfer mechanism.
- The audit log is not tamper-evident, and writes are fire-and-forget: on failure it logs to
  console and the audited action proceeds (`src/admin/postgres-audit-log.ts:54-67`).

These are documented candidly by the maintainers in a 13-item known-limitations list rather
than concealed. Documented is not fixed, and several have no configuration workaround.

## 5. Genuine upside

Were the fit problem solved, these are real gains over the current setup:

1. Per-person and per-room isolation carried through consistently — each employee and each
   Slack channel gets its own memory, files, keychain, crons and durable sandbox. The five
   brands could be five scopes with separate context. This is the problem qm is built for.
2. Background work. Crons (`src/cron/`, 1-minute minimum, IANA timezones, durable via
   pg-boss) and process watches (`src/monitors/`) run unattended — scheduled lead digests,
   campaign monitoring, inbox triage. Not possible today.
3. Durable sandboxes. Installed tools stay installed; each scope has a real computer rather
   than a fresh session.
4. Shared, governed skills — versioned, HMAC-signed, ACL'd, admin-gated for org-wide
   promotion, importable as git skill packs. The existing skill library becomes managed
   assets rather than copies.
5. Internal app publishing (`src/deploy/`) with immutable versions, rollback and scope-bound
   ACLs.
6. Slack-native multiplayer, including thread turn-detection where the agent decides whether
   to contribute.
7. Vendor neutrality — four harnesses (Pi, OpenCode, Codex, Claude Code) over one core,
   three model providers.
8. Spend caps per user and per org, enforced server-side.

None outweigh the integration rebuild today. Several become compelling once MCP support
lands.

## 6. Revisit criteria

Reassess when all three are true:

1. Third-party MCP support ships.
2. A release track record exists to extrapolate from — tags, cadence, issue-resolution
   history.
3. A data-retention and erasure story exists.

## 7. Optional pilot shape

If the upside is worth testing before those conditions are met:

- One brand only — Class Bridge or Seven Academy marketing operations. Explicitly exclude
  MetropolVisa and Medkay: no passports, no case files, no health data.
- Fly rather than AWS; far less to stand up.
- Harden defaults on day one: `SANDBOX_BACKEND=sprites` with `SPRITES_EGRESS_PROXY_URL`
  set, a non-empty per-scope `allowedHosts`, command policy switched to `mode: "allowlist"`,
  and `ORG_BUDGET_USD_PER_WINDOW` well below 100.
- Fix success criteria in advance: hours saved by background work, against engineering hours
  spent rebuilding integrations.
- Stop at four weeks if the rebuild is outpacing the benefit.

Organisation-specific work stays under `deploy/layers/fca-group/` per `AGENTS.md`, keeping
core byte-identical to upstream so merges stay small.
