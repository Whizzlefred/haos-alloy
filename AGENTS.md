# AGENTS.md

## Project Overview

This is a **Home Assistant OS (HAOS) add-on** that runs **Grafana Alloy** to ship
systemd journal logs to a **Loki** instance. It is not a traditional software project
-- there is no application language runtime. The codebase consists of Docker, shell
scripts, YAML configuration, and Alloy config (HCL-like syntax).

Reference implementation: [ecohash-co/ha-addon-alloy](https://github.com/ecohash-co/ha-addon-alloy) (MIT).
Design context: [Whizzlefred/homelab#33](https://github.com/Whizzlefred/homelab/issues/33).

## Repository Structure

```
alloy/
├── config.yaml              # HAOS add-on manifest (options schema, arch, ports)
├── build.yaml               # Base image per architecture
├── Dockerfile               # Downloads pinned Alloy binary, copies rootfs
├── CHANGELOG.md
├── DOCS.md                  # User-facing docs (rendered in HA UI)
├── README.md
└── rootfs/etc/s6-overlay/s6-rc.d/
    ├── init-alloy/          # oneshot: reads options.json, generates config.alloy
    │   ├── run
    │   ├── type
    │   └── up
    ├── alloy/               # longrun: exec alloy run
    │   ├── run
    │   ├── type
    │   └── dependencies.d/
    │       └── init-alloy
    └── user/contents.d/
        ├── init-alloy
        └── alloy
repository.yaml              # HA add-on repository manifest (repo root)
```

## Build / Test / Lint Commands

### Build the add-on Docker image locally

```bash
docker build \
  --build-arg BUILD_FROM=ghcr.io/home-assistant/amd64-base-debian:bookworm \
  --build-arg BUILD_ARCH=amd64 \
  --build-arg BUILD_VERSION=dev \
  alloy/
```

### Lint shell scripts

```bash
shellcheck alloy/rootfs/etc/s6-overlay/s6-rc.d/init-alloy/run
shellcheck alloy/rootfs/etc/s6-overlay/s6-rc.d/alloy/run
```

### Validate YAML files

```bash
yamllint alloy/config.yaml alloy/build.yaml repository.yaml
```

### Validate Alloy config syntax (requires alloy binary)

```bash
alloy fmt /path/to/config.alloy
```

### Validate Dockerfile

```bash
hadolint alloy/Dockerfile
```

### Test the container locally

```bash
# Build and run with a mock options file
docker run --rm -v /var/log/journal:/var/log/journal:ro \
  -v $(pwd)/test-options.json:/data/options.json:ro \
  -p 12345:12345 \
  <image_name>
```

There is no automated test suite. Validation is done via:
1. ShellCheck on all shell scripts
2. `hadolint` on the Dockerfile
3. Manual deployment to HAOS and verifying logs appear in Loki

## Code Style Guidelines

### Shell Scripts (s6-overlay run scripts)

- Use `#!/usr/bin/with-contenv bash` as the shebang (s6-overlay convention for
  environment injection). Do NOT use `#!/bin/bash`.
- Always `set -e` at the top of init scripts (oneshot). Longrun service scripts
  should use `exec` to replace the shell process.
- Quote all variable expansions: `"${VAR}"`, not `$VAR`.
- Use `${VAR}` brace syntax consistently, even when not strictly required.
- Use uppercase for environment variables and config paths:
  `CONFIG_DIR`, `LOKI_URL`, `JOURNAL_PATH`.
- Use lowercase for local loop variables or throwaway values.
- Prefer `jq -r` to parse JSON from `/data/options.json`.
- Provide sensible defaults with jq: `jq -r '.log_level // "info"'`.
- Print a startup banner with key config values for log debugging.
- No trailing whitespace. End files with a newline.

### Dockerfile

- Base image: `ghcr.io/home-assistant/{arch}-base-debian:bookworm` (via `BUILD_FROM` ARG).
- Pin the Alloy version explicitly (e.g., `ALLOY_VERSION="1.13.1"`). Never use `latest`.
- Minimize layers: combine `apt-get update && install && cleanup` in one `RUN`.
- Use `--no-install-recommends` for apt packages.
- Clean up `/var/lib/apt/lists/*` and temp files in the same layer.
- Always use `COPY rootfs /` to install the s6 service tree.
- Include `io.hass.*` labels for the HA Supervisor.

### YAML Configuration (config.yaml, build.yaml, repository.yaml)

- Use 2-space indentation.
- Use `>-` for multiline description strings (folded, no trailing newline).
- Quote version strings: `version: "1.0.0"`.
- List architectures explicitly: `[aarch64, amd64]`.
- Always include `journald: true` in config.yaml (required for journal access).
- Default option values must be safe/obvious (e.g., `log_level: info`).

### Alloy Configuration (config.alloy, HCL-like)

- Auto-generated at runtime by `init-alloy/run` -- do NOT hand-edit static
  config.alloy files in the repo. The template lives in the init script.
- Use `//` comments to document sections.
- Group config into labeled sections: source, relabel, process, write.
- Use 2-space indentation for blocks.
- Use snake_case for label names: `source_type`, `container_name`.
- Keep label cardinality low -- only extract journal fields that are useful
  for querying: `unit`, `hostname`, `syslog_identifier`, `transport`,
  `container_name`, `level`.

### Naming Conventions

| Thing               | Convention       | Example                       |
|---------------------|------------------|-------------------------------|
| s6 service dirs     | kebab-case       | `init-alloy`, `alloy`         |
| Shell variables     | UPPER_SNAKE_CASE | `JOURNAL_PATH`, `LOKI_URL`    |
| YAML keys           | snake_case       | `log_level`, `loki_url`       |
| Alloy block labels  | snake_case       | `loki.source.journal "system"`|
| Loki label names    | snake_case       | `source_type`, `container_name`|
| Docker labels       | dot-separated    | `io.hass.name`                |
| Files               | lowercase        | `config.yaml`, `run`, `type`  |

### Error Handling

- Init scripts (`oneshot`): use `set -e` so any failure prevents the service
  from starting. This is intentional -- a bad config should not silently run.
- Service scripts (`longrun`): use `exec` to hand off to the alloy binary.
  s6-overlay handles restart on crash via the `watchdog` URL in config.yaml.
- Always validate that required options exist before using them. Check for
  empty/null values from jq.
- Journal path detection must have a fallback:
  try `/var/log/journal` first, then `/run/log/journal`.

### Labels (Loki)

The add-on must apply these static labels to match the homelab convention:

```
host        = "homeassistant"
env         = "prod"
source_type = "journald"
retention   = "30d"
service     = "homeassistant"
```

### Target Environment

- HAOS has a **read-only root filesystem** -- all persistent state goes in `/data/`.
- The add-on container gets journal access via `journald: true` in config.yaml.
- The journal bind-mount path inside the container may be `/var/log/journal` or
  `/run/log/journal` depending on the HAOS version. Always auto-detect.
- Target Loki endpoint: `http://192.168.x.x:3100/loki/api/v1/push`
  (configurable via add-on options, not hardcoded).
- Alloy debug UI is exposed on port 12345.
- Supported architectures: `amd64`, `aarch64`.

### Git Conventions

- **Never commit or push directly to `main`.** Always create a feature branch
  and open a pull request. See [CONTRIBUTING.md](CONTRIBUTING.md).
- **Every PR must be reviewed** before merging. No self-merges.
- Branch naming: `feat/`, `fix/`, `docs/`, `chore/`, `refactor/` prefix.
- Commit messages: imperative mood, under 72 chars ("Add init script for Alloy config generation").
- One logical change per commit.
- Squash fixup commits before opening the PR.
