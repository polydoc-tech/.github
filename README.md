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

`scripts/check-public-scrub.mjs` checks the files a repo tracks against the rules in the
script and in `public-scrub.json`, and skips `vendor/` and `node_modules/`. A failure names
the file, line and rule.

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
