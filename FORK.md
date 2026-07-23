# Ouihelp's fork of chancy

This repository is a fork of [TkTech/chancy](https://github.com/TkTech/chancy).
It exists to carry patches we need before or instead of upstream, and it is
distributed to our other projects through AWS CodeArtifact rather than PyPI.

Upstream is the source of truth for everything except our patches. We take
their releases, we do not maintain a divergent codebase.

## The invariant

**Fork-specific configuration never goes into a file upstream owns.**

Everything about how we version, build and distribute the fork lives in
`scripts/release`, `.mise.toml` and this file. All three are at paths upstream
has nothing at, so they cannot conflict on a sync.

This matters more than it sounds. `pyproject.toml` and
`.github/workflows/release.yml` are the two files upstream edits most: 18 of
the last 20 upstream commits touching `pyproject.toml` are a bump of the
`version` line, and `release.yml` is rewritten regularly. Any fork value on
those lines is a guaranteed conflict on every single sync.

Concretely, this means:

- We do **not** commit a fork version into `pyproject.toml`.
- We do **not** declare the CodeArtifact index in `pyproject.toml`;
  `scripts/release` passes it with `uv publish --publish-url`.
- We do **not** edit the CI workflows to change what they publish.

Both files are currently byte-identical to upstream, and the entire fork
delta is three files upstream does not have. Keep it that way — when you need
somewhere to put a fork setting, add it to one of those three rather than to a
file upstream also edits.

## Versioning

A fork release is the upstream version the tree is sitting on, plus a fourth
release segment:

```
upstream 0.25.0  ->  0.25.0.1, 0.25.0.2, ... 0.25.0.5
upstream 0.25.1  ->  0.25.1.1, ...
```

This is valid PEP 440 and orders the way you want: `0.25.0 < 0.25.0.5 < 0.25.1`.

That version is written into `pyproject.toml` only for the duration of the
build and reverted immediately after, so it never lands in a commit. **The git
tag is the record of which revision produced a published version.** A checkout
of this repo therefore reports upstream's version, not the published one — if
you need to know what a given release was built from, look at the tag.

## Releasing

```
mise run release-check      # every check and the full build, uploads nothing
mise run release            # the real thing
```

These are thin wrappers around `scripts/release --dry-run` and
`scripts/release`, which remain runnable directly if you do not use mise.

The script, in order:

1. Refuses to run on a dirty tree — it reverts `pyproject.toml` by restoring a
   backup, and that must not be confused with real uncommitted work.
2. Reads CodeArtifact as your current AWS profile, prints the account and role
   ARN, and stops if the account is not `202878675042`. This is how you find
   out you are on the wrong profile *before* anything is built or uploaded.
3. Prompts for the version. It suggests the next one but never accepts a bare
   enter — the version is always typed in full, checked against what is already
   published and against existing tags.
4. Builds the API plugin UI with npm, then the wheel.
5. Verifies the wheel declares the right version and actually contains the UI.
6. Uploads it, then tags and pushes `v<version>`.

Prerequisites: `uv`, `npm`, `git`, `python3` — all provided by `.mise.toml` —
and an AWS session on account `202878675042` whose role can publish to the
`engineering-artifacts` repository (`codeartifact:PublishPackageVersion`). The
script catches missing credentials and a wrong account up front, but a
permissions refusal only surfaces at upload time, after the build.

### The npm step is not optional

`chancy/plugins/api/dist` holds the compiled dashboard and is matched by the
`dist/` rule in `.gitignore`, so it is never committed. Hatchling bundles only
what is on disk at build time. Build the wheel without running the UI build
first and you get a valid wheel that silently ships an API plugin with no
dashboard. `scripts/release` builds it and then asserts it landed.

## Local setup: `.mise.toml`

`.mise.toml` pins the toolchain and carries the release tasks. Upstream has no
mise config, so like `scripts/release` it is ours to change freely.

It pins `python`, `uv` and `node`. **`node` is there for a reason**: a release
builds the API plugin dashboard with npm, so a machine that can `uv build` is
not necessarily a machine that can cut a correct release. Pinning it removes
the question.

The tasks are deliberately one-liners calling `scripts/release`. The release
logic stays in the script so it has a single place to be read and reviewed, and
so it keeps working for anyone not using mise.

### AWS_PROFILE is deliberately not set

`.mise.toml` sets no AWS variables. Releasing publishes to the **production
account, `202878675042`** — not the dev/staging account you may be used to on
other repositories — and the profile that reaches it differs from person to
person, so committing one would be wrong for everyone but its author.

Point your profile at production before releasing, either per invocation:

```bash
AWS_PROFILE=<your production profile> mise run release
```

or once, in a `.mise.local.toml` next to `.mise.toml`:

```toml
[env]
AWS_PROFILE = "<your production profile>"
```

**`.mise.local.toml` must never be committed.** Note that it is not listed in
this repository's `.gitignore` and will not be: `.gitignore` is upstream's
file, and adding a fork-only entry to it is exactly what the invariant forbids.
Keep it out of git through your global ignore file (`~/.config/git/ignore`) or
this clone's `.git/info/exclude`.

Getting the profile wrong is cheap: `scripts/release` prints the account and
role it resolved to, and refuses to build anything if the account is not
`202878675042`.

## Syncing from upstream

`main` mirrors upstream. Fork patches are branched off it and released from a
branch carrying them (currently `release-tag`).

```
git fetch https://github.com/TkTech/chancy.git main:refs/remotes/upstream/main
git rebase upstream/main          # on the branch carrying our patches
```

As long as the invariant above holds, this is uneventful. Rehearsed on
2026-07-23 against the 15 upstream commits pending at the time — a `version`
bump to 0.25.1, a rewrite of `release.yml`, and the migration of
`[tool.uv] dev-dependencies` to `[dependency-groups]` — our patches rebased
with zero conflicts. That was a rehearsal on a throwaway branch: the sync
itself has not been performed, and `main` is still behind upstream.

If you do hit a conflict, note that **during a rebase `--ours` is upstream, not
you.** The semantics are inverted relative to a merge. A reflexive
`git checkout --ours` on a CI file will quietly reinstate upstream's behaviour.

## Do not cut a GitHub Release on this fork

We release by pushing a tag. `scripts/release` does that for you.

Upstream's `release.yml` still contains their `Release to PyPI` job, gated on
`if: github.event_name == 'release'`. We deliberately left it untouched so the
file stays identical to upstream. Cutting a GitHub Release here would fire it.
It would fail rather than publish anything — PyPI's trusted publisher is
configured for `TkTech/chancy`, not for us — but it produces a confusing red
build for no reason.

This is the one part of the setup that rests on convention rather than on a
mechanism. If it ever bites us, the fix is to reintroduce a one-line guard in
`release.yml` and accept the recurring conflict on that file.

## Consuming the fork

Downstream projects pull it from the private index, not from git:

```toml
[[tool.uv.index]]
name = "engineering-artifacts"
url = "https://aws@ouihelp-202878675042.d.codeartifact.eu-west-3.amazonaws.com/pypi/engineering-artifacts/simple/"
explicit = true

[tool.uv.sources]
chancy = { index = "engineering-artifacts" }
```

If the project sets `exclude-newer`, chancy needs an exemption or a freshly
published version will be invisible to the resolver:

```toml
[tool.uv.exclude-newer-package]
chancy = false
```

Bumping a consumer is then `uv lock --upgrade-package chancy`.

### One-time AWS setup

`chancy` exists on public PyPI, so CodeArtifact's origin controls matter here
in a way they do not for our own packages: an upstream ingest can block our
publishes, or serve the public package in place of ours.

`0.25.0.5` was published on 2026-07-23, so publishing is clearly permitted.
That says nothing about the *other* half — confirm `upstream=BLOCK` is actually
set, or a future resolution could hand a consumer TkTech's `chancy` instead of
ours. Applying it is idempotent:

```bash
aws codeartifact put-package-origin-configuration \
  --domain ouihelp --domain-owner 202878675042 \
  --repository engineering-artifacts \
  --format pypi --package chancy \
  --restrictions publish=ALLOW,upstream=BLOCK \
  --region eu-west-3
```
