# polydoc-tech shared configuration

The checks every polydoc-tech repo runs on a pull request, the shared agent rules and security
policy, and two Renovate presets.

## Files every repo shares

- `AGENTS.md`: the agent rules. Every repo carries a byte-identical copy. To change them, edit
  this copy and copy it over the others in the same pass.
- `CLAUDE.md`: imports `AGENTS.md`.
- `SECURITY.md`: GitHub shows it as the default security policy of every repo of the account
  that has none of its own.

## Guard

`.github/workflows/guard.yml` is a reusable workflow that every repo calls from its own CI,
so a pull request fails before it merges:

```yaml
  guard:
    uses: polydoc-tech/.github/.github/workflows/guard.yml@<commit> # main
```

Pin it to a commit. Each job checks this repo out at `job.workflow_sha`, the commit the caller
pinned, and runs the scripts and config found there, so a repo's checks stay fixed until a
Renovate digest PR moves the pin. Renovate reads the `# main` comment as the branch the digest
follows.

A private repository calls it with `with: public-scrub: false`. The scrub enforces what may
appear in a public repository; the other checks run either way. A repository turns the
`no-test` job on by naming its source directories, for example `with: source-paths: src/`.

Six jobs:

- `content`: the public scrub, the hidden-character check and the session-path check, all
  below.
- `workflows`: zizmor at `high` and actionlint over the repo's `.github`. zizmor gets the
  job's token, so its online audits run too, among them the check that a pinned commit
  belongs to the action's own repository. actionlint runs the shellcheck the runner image
  ships on every `run:` script.
- `secrets`: gitleaks over every commit reachable from the one under test, with `--redact`,
  because a public repo's CI log is public. A repo's own `.gitleaks.toml` and
  `.gitleaksignore` apply.
- `dependencies`: on a pull request only, the new-dependency check below. On any other
  event it is skipped, which a required check counts as passed.
- `lockfile-age`: on a pull request only, the lockfile age check below. Otherwise it is
  skipped.
- `no-test`: on a pull request by a person, in a repository that sets `source-paths`, the
  check on source changes without a test below. Otherwise it is skipped.

zizmor is pinned with every hash in `.github/guard-requirements.txt`; Renovate's
`pip_requirements` manager moves the version and rewrites the hashes. actionlint and gitleaks
are release downloads checked against upstream's own checksum files. Nothing bumps those two,
so a bump is a hand edit that moves the version and the hash together.
`.github/actionlint.yaml` ignores actionlint's errors on the `job.workflow_*` properties until
https://github.com/rhysd/actionlint/pull/707 ships in a release.

### A change here, and the repos that pin it

A new check, or a change to one, reaches a repo through the pull request that moves its guard
pin to the commit that made the change. That pull request runs the new check for the first
time, and goes green only once the repo passes it or lists its exceptions. Until then the repo
stays green on its old pin, and the `content` job in this repo's CI, which checks every repo
against the current commit, names it.

### Public scrub

`scripts/check-public-scrub.mjs` checks the files a repo tracks against the rules in the
script and in `public-scrub.json`, and skips `vendor/` and `node_modules/`. A failure names
the file, line and rule.

A new link to a public account that fails the check goes into `publicOwners`, and the repo
that needs it moves its guard pin to that commit. Locally:
`node scripts/check-public-scrub.mjs ../<repo> ...` and
`node --test scripts/check-public-scrub.test.mjs`.

`.github/workflows/public-scrub.yml` is the older reusable workflow that runs only the scrub,
read from `main`. It stays until every repo calls `guard.yml` instead.

### Hidden characters

`scripts/check-hidden-text.mjs` fails a repo that tracks a text file holding a bidirectional
control or mark (U+061C, U+200E, U+200F, U+202A to U+202E, U+2066 to U+2069), a zero-width
character (U+200B to U+200D, U+2060, and U+FEFF anywhere but the very start of a file), or a
Unicode tag character (U+E0000 to U+E007F). A bidi override shows a reviewer code in a
different order than the compiler reads it (CVE-2021-42574), and tag characters spell out
text that an agent reads and a reviewer cannot see.

A file that has to hold one, an official fixture for instance, goes into the repo's
`.github/hidden-text-allow.txt`, one path per line, `#` for comments. An entry whose file no
longer holds such a character fails too, so the list only shrinks. Skips and output follow
the scrub: no `vendor/` or `node_modules/`, and no line text in the log. Locally:
`node scripts/check-hidden-text.mjs ../<repo> ...` and
`node --test scripts/check-hidden-text.test.mjs`.

### Claude Code plans and session transcripts

`scripts/check-session-paths.mjs` fails a repo that tracks a text file naming a path into
Claude Code's plan files or session transcripts: `.claude` followed by `/plans/`, or by
`/projects/` anywhere but a project's `memory/` directory. Claude Code deletes both once they
are older than `cleanupPeriodDays`, 30 days by default
(https://code.claude.com/docs/en/claude-directory, "Cleaned up automatically"), and both exist
only on the machine that wrote them. A committed path to one is a link no other reader can
follow, so what a reader needs from such a file goes into the repo itself. The tilde, the home
directory and a relative prefix all count.

A file that has to keep such a path goes into the repo's `.github/session-paths-allow.txt`,
one path per line, `#` for comments. An entry whose file no longer names such a path fails
too, so the list only shrinks. Skips and output follow the scrub. Locally:
`node scripts/check-session-paths.mjs ../<repo> ...` and
`node --test scripts/check-session-paths.test.mjs`.

### New dependencies

`scripts/check-new-dependencies.py` compares the dependency manifests of a pull request's
test merge commit with those of its base. A package name that no manifest of the same
ecosystem listed before is new, and each new one must:

- exist on npm, PyPI or Packagist with at least one published release;
- be at least 30 days past its first release, or carry a
  `Young-Dependency: <name> <reason>` line in the pull request body;
- carry a `New-Dependency: <name> <reason>` line in the pull request body.

A model can invent a package name, and a squatter can register the invented name with
malware in it. The first rule catches a name nobody registered, the second a name
registered last week, and the third puts the reason where a reviewer reads it.

It reads `package.json`, `pyproject.toml`, `requirements*.txt`, `requirements*.in` and
`composer.json`, and skips a requirements file that pip-compile wrote, since that pins every
transitive package. A dependency moved between sections or manifests is not new. A local path
is not checked; a git or URL source needs its reason line, but no registry can vouch for it.

The body comes from the pull request event, so a re-run reads it as it was when the run
started. After adding a line, push a commit or close and reopen the pull request, unless the
calling workflow also runs on `pull_request` `edited`. Locally:
`PR_BODY="..." python3 scripts/check-new-dependencies.py <base> <head>` in a repo, and
`python3 scripts/check-new-dependencies.test.py` here (`GUARD_LIVE_REGISTRIES=1` adds one
case against the real registries).

### Lockfile age

`scripts/check-lockfile-age.py` compares every yarn 1 `yarn.lock` at a pull request's test
merge commit with the same path at its base. Each `name@version` the base did not hold,
dependencies of dependencies included, must have been published on npm at least 72 hours
before the run. Renovate ages the version it proposes, but the `yarn install` it then runs
locks the newest version inside each range, whatever its age, for the package and for
everything below it, because yarn 1 has no publish-date cutoff. `package-lock.json` is not
read.

Only added versions are read: one packument can weigh over 30 MB, and a version on the base
was checked by the pull request that added it. A local path is not read. A tarball from a host
other than the npm registry is named in the output and not checked. A registry that fails
three times makes the job exit 2, never pass.

Nothing re-runs a failed job once its versions have aged. A repo that wants the pull request to
go green on its own, so that auto-merge goes ahead, calls
`.github/workflows/lockfile-age-rerun.yml` once a day; its header shows the caller. It runs
`scripts/rerun-lockfile-age.py`, which re-runs the newest `lockfile-age` job on the head of
each open pull request when that job failed. It needs only the run's own token, with
`actions: write`. Locally: `python3 scripts/check-lockfile-age.py <base> <head>` in a repo, and
`python3 scripts/check-lockfile-age.test.py` and `python3 scripts/rerun-lockfile-age.test.py`
here (`GUARD_LIVE_REGISTRIES=1` adds one case against the real npm).

### Source changes without a test

`scripts/check-untested-source.py` reads the files a pull request adds or changes, compared
with its base. When one of them is source code and none of them is a test, the pull request
body needs a `No-Test: <reason>` line.

- Source: a file under one of the caller's `source-paths` with a code extension (`.py`,
  `.ts`, `.tsx`, `.mts`, `.cts`, `.js`, `.jsx`, `.mjs`, `.cjs`, `.php`) that is not a test.
- Test: any file in the repository with a `test`, `tests`, `e2e` or `__tests__` directory on
  its path, or named `*.test.*`, `*.spec.*`, `test_*.py` or `*_test.py`. A new fixture under a
  test directory counts; Markdown there does not.
- A deleted file needs no test. A rename reads as a deletion and an addition, so a moved
  source file needs a test or the line. `vendor/` and `node_modules/` are skipped.
- A pull request opened by a bot is skipped.
- A `No-Test:` line inside an HTML comment or a fenced code block does not count, so a pull
  request template can show the line without meeting it.

The body is read as with the new-dependency check. Locally:
`PR_BODY="..." python3 scripts/check-untested-source.py <base> <head> src/` in a repo, and
`python3 scripts/check-untested-source.test.py` here.

### Where the content checks run

- In each repo's own CI, through the `content` job of `guard.yml`.
- In `.github/workflows/ci.yml`, job `content`, over this repo and a fresh clone of every
  non-fork repo of the account, on each push and pull request here and daily at 04:23 UTC.

## Renovate presets

The tool repos extend `github>polydoc-tech/renovate-config`. The two presets here are for
repos that want this older base:

- `default.json` (`local>polydoc-tech/.github`): weekly schedule, dependency dashboard,
  semantic commits, grouped patch, minor and major updates, security alerts labelled and
  assigned. No auto-merge.
- `automerge.json` (`local>polydoc-tech/.github:automerge`): extends the base and adds
  auto-merge for patch and digest updates (and security updates) once CI passes. Only use
  this in repos that run a check on `pull_request`, otherwise updates merge with no gate.
