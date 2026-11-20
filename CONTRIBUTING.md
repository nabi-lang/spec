# Contributing to the Nabi specification

This repository is the Nabi language and standard-library specification, in two
editions: English (`en/`) and Korean (`ko/`).

## Keeping the editions in sync

A change to one edition is mirrored in the other in the same change, with the
same section numbers and heading anchors. Cross-references link to `.md` files
with the generated heading anchors (GitHub heading rules).

## Style

Normative prose is precise and spare. Code examples are valid Nabi or NABION.

## Submitting

Open a pull request against `main`. The spec text is licensed CC BY 4.0; by
contributing you agree to license your work the same way.

## Sign your work

Nabi uses the [Developer Certificate of Origin 1.1](https://developercertificate.org/)
(DCO) instead of a CLA. By adding a `Signed-off-by` line to a commit you
certify that you wrote the change, or otherwise have the right to submit it
under this repository's license. `git commit -s` adds the line:

```text
Signed-off-by: Your Name <you@example.com>
```

Use your name and an email you can be reached at; the email must be the
commit's author email. The `DCO / Check sign-off` check fails a pull request with an
unsigned commit. To fix it, run `git commit --amend -s` (last commit) or
`git rebase --signoff main` (every commit), then push again. The full text
of the DCO is in the
[organization's CONTRIBUTING.md](https://github.com/nabi-lang/.github/blob/main/CONTRIBUTING.md#developer-certificate-of-origin).
