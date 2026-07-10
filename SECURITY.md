# Secrets handling standard

This is the org-wide standard for keeping secrets and personal data out of git. It applies to **every** repository. Copy the tooling in "Repo setup" into any project that doesn't have it yet.

## Rules

1. **Real secrets never enter git.** Not in `.env`, and **never** in `.env.example`, `*.example`, config samples, fixtures, test data, or documentation. If a value would let someone authenticate as you or the org, it does not belong in a tracked file.
2. **Sample/config files hold placeholders only** — `your-subdomain`, `you@example.com`, `your-api-token`. Never a real token, subdomain, host, or credential.
3. **No personal data in tracked files.** Use generic placeholders and shared **service/role accounts** — never a personal email address or name in env/example/config files.
4. **A leaked secret is compromised forever.** Rewriting git history does not un-leak it. Always rotate first (see runbook).

## Where secrets actually live

| Context | Location |
| --- | --- |
| Local development | git-ignored `.env` (copy from `.env.example`), or the `zcli login` credential store |
| CI / GitHub Actions | GitHub Actions **secrets** (`secrets.*`), referenced in workflow YAML |
| Never | any tracked file in the repo |

For this project the relevant vars are `ZENDESK_SUBDOMAIN`, `ZENDESK_EMAIL`, `ZENDESK_API_TOKEN` (see `.env.example`).

## Repo setup (copy into every project)

Three layers back each other up: a local pre-commit hook, a CI job, and org-level push protection (configured once in GitHub org settings).

### 1. `.gitignore`

Ensure it ignores `.env` and env variants:

```gitignore
.env
.env.*.local
```

### 2. Local pre-commit hook (`.husky/pre-commit`)

Requires [husky](https://typicode.github.io/husky/) and [gitleaks](https://github.com/gitleaks/gitleaks) (`brew install gitleaks`):

```sh
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

if command -v gitleaks >/dev/null 2>&1; then
  gitleaks protect --staged --redact --verbose
else
  echo "⚠️  gitleaks not found — skipping local secret scan. Install it: brew install gitleaks" >&2
fi
```

### 3. `.gitleaks.toml`

```toml
[extend]
useDefault = true

[allowlist]
description = "Ignore obvious placeholder values in sample/config files."
regexes = [
  '''your-api-token''',
  '''your-subdomain''',
  '''you@example\.com''',
]
```

Add project-specific `[[rules]]` for provider tokens (e.g. the `zendesk-api-token` rule in this repo's `.gitleaks.toml`).

### 4. CI job

Add a `secret-scan` job to the checks workflow. We run the gitleaks **binary** directly rather than `gitleaks-action@v2`, because that action requires a paid `GITLEAKS_LICENSE` for GitHub **organization** accounts:

```yaml
  secret-scan:
    name: Secret scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install gitleaks
        run: |
          GITLEAKS_VERSION=8.21.2
          curl -sSfL "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" | tar -xz gitleaks
          sudo mv gitleaks /usr/local/bin/gitleaks
      - name: Scan for secrets
        run: gitleaks detect --config .gitleaks.toml --redact --verbose --no-banner
```

### 5. Org-level (GitHub org admin, once)

In **Org Settings → Code security**: enable **Secret scanning** and **Push protection** for all repos and by default for new repos. Push protection blocks secret pushes at the server, so it can't be skipped by bypassing local hooks. (Free for public repos; private repos need GitHub Secret Protection / Advanced Security.)

A step-by-step click-path with licensing notes and a verification test is kept as a standalone guide to hand to whoever holds org-owner access — ask the maintainers for the link.

## Leak runbook

If a secret reaches git (or any shared surface):

1. **Rotate immediately.** Revoke the leaked credential and issue a new one at the provider. Do this first — before touching git.
2. **Replace the value** in the working tree with a placeholder and commit.
3. **Purge from history** with [`git filter-repo`](https://github.com/newren/git-filter-repo):
   ```sh
   printf '%s==>***REMOVED***\n' 'THE_LEAKED_VALUE' > replacements.txt
   git filter-repo --replace-text replacements.txt
   git push --force-with-lease --all
   ```
4. **Notify the team.** Everyone must re-clone or hard-reset after a history rewrite; open PRs may need rebasing.
5. If the repo is/was public, ask GitHub Support to purge cached views of the old commits.
