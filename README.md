# @tylervigario/commitlint-config

Conventional Commits, without `@commitlint/config-conventional`'s house style — and
with a type enum that obeys the specification.

```json
{
  "extends": ["@tylervigario/commitlint-config"]
}
```

Three lines of JSON is the whole consumer config — the rules and the plugin live here.

### On the shorthand

`extends: ["@tylervigario"]` also works. commitlint appends `commitlint-config` to a bare scope,
the same way eslint does, so it resolves to this package. Both forms are asserted by the
`Consumer install resolves` job, against the packed tarball installed into a scratch
project — so this paragraph fails CI rather than going quietly stale.

The full name is used throughout anyway, for one reason worth knowing before choosing the
shorter one. commitlint tries the modern prefix first and a **legacy** one last, so when
the package is simply *not installed*, the shorthand reports the candidate it gave up on:

```
extends: ["@tylervigario"]                     Cannot find module "@tylervigario/conventional-changelog-lint-config"
extends: ["@tylervigario/commitlint-config"]   Cannot find module "@tylervigario/commitlint-config"
```

The first names a package that has never existed, and reads as though the shorthand were
wrong. It is not — that error means `npm install` has not run. The second names the
package you meant. Same failure, and only one of them points at it.

## Install

From npm:

```bash
npm install --save-dev @commitlint/cli conventional-changelog-conventionalcommits @tylervigario/commitlint-config
```

`conventional-changelog-conventionalcommits` is a **peer** dependency — the parser preset,
which is the only thing this needs from that side of the ecosystem. The consumer decides
its version.

`@commitlint/config-conventional` is **not** required and is no longer extended. From
0.3.0 this declares the parser preset directly rather than inheriting twelve rules and
disabling eight of them.

## What it enforces

Two things, and deliberately little else.

**`feat` and `fix` are a floor.** They are the only types the specification gives
behaviour to — MINOR and PATCH, with `!` for MAJOR — and the only strings anything
downstream reads. A consumer cannot remove them; the rule unions them in on every
evaluation.

**Everything else is a replaceable starter.** The reference implementation's
remaining nine ship as a default. A repository may swap the whole set for types its own domain warrants.

```js
// inherit the starter
export default { extends: ["@tylervigario/commitlint-config"] };

// replace it — feat and fix survive regardless
export default {
  extends: ["@tylervigario/commitlint-config"],
  rules: { "type-enum-any-case": [2, "always", ["deploy", "migrate"]] },
};

// the floor and nothing else
export default {
  extends: ["@tylervigario/commitlint-config"],
  rules: { "type-enum-any-case": [2, "always", []] },
};
```

## What passes unchecked

Only what git writes on a branch: its merge commits (`Merge branch 'main' into feat-x`)
and the commits `--fixup`, `--squash` and `--fixup=amend:` make. A commit-msg hook meets
these every day, and under squash merging none of them reaches the default branch.

commitlint's own list passes more, and this config checks the rest. GitHub's revert
button titles its pull request `Revert "feat: add login"`; squashed under that title it
lands as a commit git-cliff and cocogitto both skip, and the changelog goes on listing
the feature it removed. Here that title is refused, and `revert: feat: add login` is
accepted. The same goes for `Reapply "…"`, the forge's merge messages and a bare version.

A pull request titled `Merge branch …` or `fixup! …` still passes. Nobody titles one that
way, and it is the price of a hook that works. A repository that wants commitlint's
whole list back sets `defaultIgnores: true` in its own config.

## Why the set beyond the two is open

The specification defines behaviour for `feat` and `fix`, says of everything else
only that other types **MAY** be used, and enumerates nothing. Its FAQ goes further:
the flexibility "allows your team to come up with their own types and change those
types over time."

The familiar eleven are nobody's standard. They are **not Angular's** — Angular
publishes eight (`build`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `test`),
with no `chore`, no `style`, and `revert` as a special prefix rather than a type.
The eleven come from the reference implementation `conventional-changelog` ships,
read downstream as though it were the specification.

So any restriction past the two is a local decision, and one this config declines to
make on a consumer's behalf.

A narrowed enum costs twice: it rejects a commit the standard accepts, and it
forecloses the addition a new domain would justify. An earlier version of this file
did exactly that — it dropped `style` from the eleven, not by decision but because
the rule below needed a list and the list became policy.

## Why the enum is replaced rather than configured

The built-in `type-enum` is `enums.indexOf(value) > -1` — an exact match against a
lowercase array, with no case option exposed. Specification item 15:

> The units of information that make up Conventional Commits MUST NOT be treated as
> case-sensitive by implementors, with the exception of BREAKING CHANGE which MUST be
> uppercase.

The type is such a unit, so the built-in is non-conformant and cannot be configured
out of it. Replacing it is the only route that keeps an enum while obeying the spec.

The replacement takes its list from the rule's **value** rather than a module
constant, which is what makes it extensible at all.

## Why there is no `extends: @commitlint/config-conventional`

Its only contribution to *parsing* is one line — `parserPreset`. The other twelve
entries are house style. An earlier version of this file inherited all of it and then
disabled eight, which is a lot of argument to end up where declaring the preset
directly starts.

Style rules are dropped rather than tuned because the specification says nothing
about appearance, and **cocogitto** — the other linter in use across these
repositories — has no equivalent for any of them. Enforcing appearance only where a
toolchain happens to be able to means the same commit passes in one repository and
fails in another, which is not a standard.

## Packaging, and one trap when working on this locally

**`extends` resolves from the config file's real directory, not the consumer's.** That
one fact explains everything below.

Installed normally — from npm, or from a tarball — this package lands as a real directory
at `node_modules/@tylervigario/commitlint-config`. Resolving the parser preset from there walks up
into the consumer's `node_modules` and finds it. That is why
`conventional-changelog-conventionalcommits` is a **peer** dependency: the consumer already
has it, and the consumer picks the version.

**`npm install ../commitlint-config` breaks that, silently and confusingly.** A path
install creates a *symlink* to the source checkout, so resolution starts in
`~/src/commitlint-config`, which has no `node_modules`. Every message then fails, with no
rule named:

```
Error: Cannot find module "conventional-changelog-conventionalcommits"
       from "C:\Users\tyler\src\commitlint-config"
```

Nothing is wrong with the package. To test it the way it ships:

```bash
cd commitlint-config && npm pack          # → tylervigario-commitlint-config-0.4.0.tgz
cd ../consumer && npm install ../commitlint-config/tylervigario-commitlint-config-0.4.0.tgz
```

The same applies to `npm link`, for the same reason.

**A JSON consumer config is fine.** Plugins are objects containing functions, so a
consumer *authoring* a plugin needs a `.js` config — but inheriting one does not, because
the functions live in this package. `.commitlintrc.json` with three lines works.

**Published to npm, with provenance.** It exists for anyone whose commits
config-conventional rejects for opinions the specification does not hold, and npm is
where they would look. `package-lock.json` records the exact version and its integrity,
so a consumer upgrades deliberately. Every version is published by this repository's
Release workflow, as npm's trusted publisher, and carries a provenance attestation npm
checks for the consumer.

## Verifying a release

```bash
# In a consumer, npm's own check of every installed package's signatures and provenance:
npm audit signatures

# The tarball, against the attestation on the GitHub release beside it:
gh attestation verify tylervigario-commitlint-config-<version>.tgz \
  --repo TylerVigario/commitlint-config \
  --signer-workflow TylerVigario/commitlint-config/.github/workflows/release.yml \
  --source-ref refs/heads/main

# The release itself: its tag, its commit and its assets, as GitHub published them:
gh release verify v<version> --repo TylerVigario/commitlint-config
```

The tarball's attestation is signed by `release.yml` running on `main`, and by nothing
else; one from any other workflow or branch is not this package's.

## Releases

A release is a version on npm, published by the Release workflow from a tarball it
packs once. The same tarball and its attestation are the assets of the GitHub release
`v<version>`, and the tag names the commit that records that version in `package.json`
and its entry in [`CHANGELOG.md`](CHANGELOG.md). The version comes from the commits since the last
tag: `feat` bumps the minor version, `fix` the patch, and before 1.0 a breaking
change bumps the minor as well. Tags are create-once — none is moved or deleted —
so the commit a version names today is the one it names for ever.

## Licence

Copyright © 2026 Tyler Vigario.

Free software under the [GNU Affero General Public License](LICENSE), version 3
or (at your option) any later version.
