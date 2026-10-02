# Rulesets

`main.json` is the branch protection payload for this repository, and `tags.json`
the tag protection. They are committed
because a ruleset is applied state that lives only on GitHub: it vanishes silently on
repository recreate, rename or fork, and nothing in a clone reveals it is gone.

The file is the source of truth. Apply it, do not hand-configure:

```bash
# create
gh api repos/<org>/<repo>/rulesets --method POST --input .github/rulesets/main.json

# update an existing one
gh api repos/<org>/<repo>/rulesets/<id> --method PUT --input .github/rulesets/main.json
```

Read back what is actually enforced — from the **rules** endpoint, not the legacy
branch-protection API, which reports `enforcement_level: off` even where a ruleset is
demonstrably active:

```bash
gh api repos/<org>/<repo>/rules/branches/main
```

The file carries only the fields the create and update endpoints accept. `id`,
`node_id`, `source` and `created_at` are server-assigned; committing them would invite
someone to edit a value the API ignores.

## Why each rule

**`pull_request`, 0 approvals** — a single owner cannot approve their own pull request,
so requiring one deadlocks the repository outright. The pull request is the record; the
approval count adds nothing where there is one reviewer.

**`allowed_merge_methods: squash`** — no merge commits, and no rebase either. A second
parent buys nothing merging into linear history; rebase is excluded because it replays
branch commits onto main verbatim, so a check validating the pull request title never
sees them.

**`deletion`, `non_fast_forward`** — the branch cannot be removed or rewritten.

**`required_linear_history`** — not the same lever as disabling merge commits in the
repository settings. Merge methods are settings: one API call re-enables them, touching
no ruleset and leaving nothing in the ruleset history. The rule holds the shape against
a setting drifting back.

**`required_status_checks`** — one context per assertion, no aggregate:

| context | from | asserts |
| --- | --- | --- |
| `Validate PR title` | `commit-convention.yml` | the pull request title parses as a Conventional Commit |
| `Config behaves (linux)` | `ci.yml` | every clause of `index.mjs`'s contract, against a config on disk |
| `Config behaves (windows)` | `ci.yml` | the same, on the platform where the harness assumptions hide |
| `Consumer install resolves (linux)` | `ci.yml` | the packed package, installed, reached by the name a consumer writes |
| `Consumer install resolves (windows)` | `ci.yml` | the same, on the platform where the harness assumptions hide |

These are **job names**, taken from each job's `name:` field. The two `ci.yml` jobs
run under a matrix, and a matrix *suffixes* the job name — so `name:` is set
explicitly there rather than left to default, and the suffix is a short stable label
rather than the runner image, which moves. Getting this wrong does not error: the
context simply never reports and the branch wedges. Renaming a job disables that
gate without a word of warning, and adding a job without adding its context here leaves
protection reading as complete while covering less. To change one: relax the rule, merge
the change, tighten it again, then verify against the rules endpoint.

`Lint main` is deliberately **not** required. It runs on `push` and cannot report on a
`pull_request` event, so requiring it would leave every pull request waiting on a context
that never arrives.

**`bypass_actors`** — one App or none, never an actor *type*. An actor-type exemption
is inherited by anything authenticating as that actor: on a single-owner repository
"repository admin" exempts the owner *and* every automation acting on the owner's
behalf, which is the entire population the rule exists to constrain.

The one App is the release's, **`commitlint-config-release` (App ID 5159975)**,
installed on this repository alone, because a release records its version and changelog
on `main` and nothing else may write it. It is listed only once installed: a bypass
naming an actor the forge cannot resolve fails the whole payload, not just that entry.
Its blast radius is its own permissions — contents read and write, and no workflows
scope, so the actor that can write the branch cannot rewrite what runs on it.

**`code_scanning`** — refuses a pull request that introduces a CodeQL error, or a
security finding of medium or higher. It reads the analysis rather than a job, which is
why the CodeQL jobs are not among the contexts above. Like a required context, it is
applied only once the thing behind it can answer — after CodeQL has analysed `main` —
or every pull request waits on an analysis that never comes.

## `tags.json`

A release here *is* a tag, consumed through `#semver:` ranges, so a tag that moved
would hand a consumer different code under a version they had already resolved — and
the one who already holds it is never told. `deletion`, `non_fast_forward` and
`update`: the last is the one that matters, because advancing a tag to a descendant is
a fast-forward and the first two say nothing about it. With all three a tag is
create-once — the first push of a name is accepted and every later one refused. It
covers the tags that predate it too.

No bypass actor. The release creates tags and never moves them, so it needs nothing
this ruleset withholds, and an actor listed here would be past `update` and `deletion`
as readily as anything else.

## Why this repository is worth protecting

It defines what a conforming commit is for every repository that consumes it. A bad
commit on `main` here changes what every downstream repository accepts, and the
`#semver:` ranges consumers pin against resolve from this repository's tags — so the
blast radius is every Node project in the estate, not just this one.
