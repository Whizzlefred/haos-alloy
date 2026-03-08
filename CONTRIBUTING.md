# Contributing

Thank you for your interest in contributing to **haos-alloy**. This document
covers the workflow, quality expectations, and conventions that every
contributor -- human or AI agent -- must follow.

## Git Workflow

### Golden rule: never commit or push directly to `main`

The `main` branch is protected. All changes go through feature branches and
pull requests, no exceptions.

```
# 1. Create a branch from main
git checkout main && git pull
git checkout -b feat/my-change

# 2. Make your changes, commit
git add -A && git commit -m "Add my change"

# 3. Push the branch
git push -u origin feat/my-change

# 4. Open a pull request on GitHub
gh pr create --title "Add my change" --body "Description of what and why"
```

### Branch naming

Use a short prefix that describes the type of change:

| Prefix     | Use case                          |
|------------|-----------------------------------|
| `feat/`    | New functionality                 |
| `fix/`     | Bug fix                           |
| `docs/`    | Documentation-only changes        |
| `chore/`   | Build, CI, dependency updates     |
| `refactor/`| Code restructuring, no behavior change |

Examples: `feat/add-log-level-filter`, `fix/journal-path-fallback`,
`docs/update-readme`.

### Pull requests

- Every PR **must be reviewed** before merging. Do not self-merge.
- Keep PRs small and focused -- one logical change per PR.
- Write a clear title (imperative mood) and a short body explaining **why**
  the change is needed.
- If the PR relates to a GitHub issue, reference it: `Closes #12`.
- Ensure all validation passes before requesting review (see below).

### Commits

- Use imperative mood: "Add journal path fallback", not "Added" or "Adds".
- Keep the subject line under 72 characters.
- One logical change per commit. Avoid "fix typo" commits stacked on top of
  each other -- squash them before opening the PR.
- Do not commit secrets, credentials, IP addresses that should stay private,
  or editor/IDE configuration files.

## Validation Checklist

Run these before pushing. There is no CI pipeline yet, so local validation
is the only gate.

### Shell scripts

```bash
shellcheck alloy/rootfs/etc/s6-overlay/s6-rc.d/init-alloy/run
shellcheck alloy/rootfs/etc/s6-overlay/s6-rc.d/alloy/run
```

### Dockerfile

```bash
hadolint alloy/Dockerfile
```

### YAML

```bash
yamllint alloy/config.yaml alloy/build.yaml repository.yaml
```

### Docker build (smoke test)

```bash
docker build \
  --build-arg BUILD_FROM=ghcr.io/home-assistant/amd64-base-debian:bookworm \
  --build-arg BUILD_ARCH=amd64 \
  --build-arg BUILD_VERSION=dev \
  alloy/
```

### Local container test

```bash
docker run --rm \
  -v /var/log/journal:/var/log/journal:ro \
  -v "$(pwd)/test-options.json":/data/options.json:ro \
  -p 12345:12345 \
  <image_name>
```

Verify Alloy starts, the debug UI responds on `http://localhost:12345`, and
logs appear at the configured Loki endpoint.

## Code Style

Detailed style rules live in [AGENTS.md](AGENTS.md). The short version:

- **Shell**: `#!/usr/bin/with-contenv bash`, `set -e` in init scripts,
  `exec` in longrun scripts, quote all `"${VARS}"`.
- **Dockerfile**: pin versions, single-layer installs, clean up in the same
  layer, `COPY rootfs /`.
- **YAML**: 2-space indent, quote version strings, `>-` for descriptions.
- **Alloy config**: generated at runtime by `init-alloy/run` -- edit the
  template in the shell script, not a static `.alloy` file.

## What to Contribute

Good first contributions:

- Improving documentation (DOCS.md, README, inline comments)
- Adding ShellCheck / hadolint fixes
- Adding CI via GitHub Actions (shellcheck, hadolint, docker build)
- Supporting additional architectures
- Adding configurable Loki labels via add-on options

## Reporting Issues

Open a GitHub issue with:

1. What you expected to happen
2. What actually happened (include add-on logs if possible)
3. Your HAOS version and architecture (amd64 / aarch64)

## License

By contributing you agree that your contributions will be licensed under the
same license as this project.
