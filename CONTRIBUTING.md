# Contributing

Thanks for the interest. A few things up front:

- This is a small project with one maintainer. Bug reports get faster attention
  than proposals to enforce more.
- It is [AGPL-3.0-or-later](LICENSE). By submitting a contribution you agree to
  license it under the same terms — no CLA, no copyright assignment.
- There is no chat. The conversation lives in issues and pull requests.

## What it will and will not take

The package enforces what the Conventional Commits specification fixes and
nothing else: `feat` and `fix` as a floor, a type set a consumer may replace,
types matched in any case. A rule about how a commit *looks* — its case, its
length, its punctuation — is style, which the specification says nothing about
and which cocogitto, the other linter in use across these repositories, cannot
mirror. Proposals of that kind are declined, and the README says why.

## Reporting a bug

Open an [issue](https://github.com/TylerVigario/commitlint-config/issues/new/choose)
with the exact commit message, what you expected, what commitlint said, and the
versions involved. A security problem goes through [SECURITY.md](SECURITY.md)
instead.

## Getting it running

```bash
git clone https://github.com/TylerVigario/commitlint-config.git
cd commitlint-config
npm ci
node --test
```

`node --test` runs both suites. `Config behaves` asserts every clause of
`index.mjs`'s contract against a config on disk. `Consumer install resolves` packs
the package, installs the tarball into a scratch project and lints through it,
because `exports`, `main` and the peer dependency are only reached that way. CI
runs both on Linux and on Windows.

Testing a change from another repository: install the packed tarball, not a path
— `npm install ../commitlint-config` makes a symlink, and resolution then starts
in this checkout. The README's *Packaging* section has the detail.

## Shaping a change

- **A pull request title is a Conventional Commit.** It is checked, and under
  squash merging it becomes the commit on `main` — and the changelog line, and
  the version bump.
- **A change to what is enforced comes with the test that states it.** Each
  clause of the contract in `index.mjs` has an assertion in
  `test/config-behaves.test.mjs`; a new clause without one can be rewritten later
  without anything noticing.
- **Every action is pinned to a full commit SHA**, with the version in a
  comment. The repository refuses to run an unpinned one.
- **Releases are not made by pull request.** The Release workflow records the
  version and tags it; see the README's *Releases* section.
