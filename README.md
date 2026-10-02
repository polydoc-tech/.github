# polydoc-tech shared configuration

Org-wide Renovate presets. Repos reference these instead of duplicating the full policy.

## Presets

- `default.json` (`local>polydoc-tech/.github`) — base policy: weekly schedule, dependency
  dashboard, semantic commits, grouped patch / minor / major updates, security alerts
  labelled and assigned. No auto-merge.
- `automerge.json` (`local>polydoc-tech/.github:automerge`) — extends the base and adds
  auto-merge for patch and digest updates (and security updates) once CI passes. Only use
  this in repos that run a check on `pull_request`, otherwise updates merge with no gate.

## Usage

Repo without a PR-triggered CI check:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>polydoc-tech/.github"]
}
```

Repo with a PR-triggered CI check:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>polydoc-tech/.github:automerge"]
}
```

## Public scrub

Every repo this account owns is public, so planning notes, runbooks, local paths and links
to private repos must stay out of them. `scripts/check-public-scrub.mjs` fails a repo that
tracks:

- a file whose name contains `roadmap` in any case, or starts with `PASS-`, `SUBMISSION` or
  `PR-COMMENT`;
- a line that links to a roadmap Markdown file, names a `<word>-hq` store, or holds a local
  home or workspace path;
- a GitHub link or an `owner/repo#N` reference whose owner is not in `publicOwners` in
  `public-scrub.json`. Lockfiles are exempt from this rule only, because they list every
  dependency's publisher;
- a file named in `agentFiles` in `public-scrub.json` that differs from the copy in this
  repo. The list is empty here.

`vendor/` and `node_modules/` are skipped. The patterns are generic and the output names
only the file, line and rule, never the matched text: a public repo's CI log is public too,
so the check must not publish what it guards.

Where it runs:

- `.github/workflows/ci.yml`, job `public-scrub`, over this repo and a fresh clone of every
  non-fork repo of the account, on each push and pull request here and daily at 04:23 UTC.
- `.github/workflows/public-scrub.yml`, a reusable workflow a repo calls from its own CI so a
  pull request fails before it merges:

  ```yaml
    public-scrub:
      uses: polydoc-tech/.github/.github/workflows/public-scrub.yml@main
  ```

A new link to a public account that fails the check goes into `publicOwners` in the same
pull request. Locally: `node scripts/check-public-scrub.mjs ../<repo> ...` and
`node --test scripts/check-public-scrub.test.mjs`.
