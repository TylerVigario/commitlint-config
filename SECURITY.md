# Security

## Reporting

**Do not open a public issue.**

Use [private vulnerability reporting](https://github.com/TylerVigario/commitlint-config/security/advisories/new)
on this repository. It goes to the maintainer and to nobody else.

Expect an acknowledgement within a few days. This is a one-maintainer project
run alongside other work, so a fix may take longer than an acknowledgement — you
will be told which is happening.

## What is in scope

`index.mjs` is loaded and run by commitlint in every repository that extends it,
on developers' machines and in their CI. So, in particular:

- anything in it that executes, reads or reaches further than judging a commit
  message
- a way for a commit message to make it misbehave rather than simply pass or fail
- the packaging — `exports`, `main`, the peer range — resolving to something
  other than this package
- the workflows here, which run holding this repository's token

## What is not

- commitlint itself, and the `conventional-changelog-conventionalcommits` parser
  preset. Report those upstream
- a rule that accepts or rejects a commit wrongly. That is a bug: open an issue
- dependency advisories with no path to exploitation here. Every dependency is a
  devDependency, installed to run this repository's tests and never by a
  consumer. Report them, but they are triaged as maintenance
