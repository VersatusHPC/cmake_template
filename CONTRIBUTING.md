# Contributing

## Commit message policy

This project uses [Conventional Commits](https://www.conventionalcommits.org/) enforced by [gitlint](https://jorisroovers.com/gitlint/).

- Format: `type(scope?): subject` with allowed types `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, `test`.
- Subject: imperative mood, no trailing punctuation, max 72 characters.
- Body: wrap lines at 80 characters; add context for the change when needed.
- Sign-offs: include `Signed-off-by: Full Name <email>` in the body; sign commits when possible via `git commit -S -s`.

Example:

```
feat(network): Add TLS client certificate rotation

Implement automatic rotation of TLS client certificates before expiry.

Signed-off-by: Jane Doe <jane@example.com>
```

## Setting up the commit-msg hook

The repository includes a `.githooks/commit-msg` hook that validates messages automatically. Enable it by pointing Git to the hooks directory:

```sh
git config core.hooksPath .githooks
```

This requires gitlint to be installed:

```sh
pipx install gitlint
```
