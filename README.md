# haos-alloy

Home Assistant OS add-on that runs [Grafana Alloy](https://grafana.com/oss/alloy/)
to ship systemd journal logs to a [Loki](https://grafana.com/oss/loki/) instance.

## Why

HAOS has a read-only root filesystem and no package manager, so standard log
shippers (Promtail, Fluent Bit, Vector) cannot be installed directly. The
only maintained Promtail add-on
([mdegat01/addon-promtail](https://github.com/mdegat01/addon-promtail)) is
abandoned and broken on HAOS 14+ due to the compact journal format introduced
in systemd 252.

This add-on solves the problem by running a current Alloy binary as a
managed HAOS add-on with full journal access.

## Features

- Ships all systemd journal logs (HA Core, Supervisor, add-ons, OS) to Loki
- Auto-detects journal path (`/var/log/journal` or `/run/log/journal`)
- Configurable via the Home Assistant UI (Loki URL, log level)
- Alloy debug UI on port 12345
- Survives HAOS reboots and updates
- Supports `amd64` and `aarch64`

## Installation

### As a local add-on

1. Copy the `alloy/` directory to the `/addons/` share on your HAOS instance
2. In Home Assistant, go to **Settings > Add-ons > Add-on Store**
3. Click the three-dot menu and select **Check for updates**
4. The "Grafana Alloy" add-on appears under **Local add-ons** -- click **Install**

### As a repository add-on

1. In Home Assistant, go to **Settings > Add-ons > Add-on Store**
2. Click the three-dot menu and select **Repositories**
3. Add this repository URL: `https://github.com/Whizzlefred/haos-alloy`
4. Install "Grafana Alloy" from the store

## Configuration

| Option              | Default                                         | Description                        |
|---------------------|--------------------------------------------------|------------------------------------|
| `loki_url`          | `http://localhost:3100/loki/api/v1/push`         | Loki push API endpoint             |
| `log_level`         | `info`                                           | Alloy log level (debug/info/warn/error) |
| `additional_config` | *(empty)*                                        | Raw Alloy config blocks to append  |

## Building Locally

```bash
docker build \
  --build-arg BUILD_FROM=ghcr.io/home-assistant/amd64-base-debian:bookworm \
  --build-arg BUILD_ARCH=amd64 \
  --build-arg BUILD_VERSION=dev \
  alloy/
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full validation checklist
and development workflow.

## Architecture

The add-on uses [s6-overlay](https://github.com/just-containers/s6-overlay)
as the process manager (standard for HAOS add-ons):

1. **init-alloy** (oneshot) -- reads `/data/options.json`, detects the
   journal path, and generates `/etc/alloy/config.alloy` at startup
2. **alloy** (longrun) -- runs `alloy run` with the generated config;
   s6-overlay handles restarts on crash

The Alloy pipeline: `loki.source.journal` reads the journal, passes entries
through `loki.relabel` (extracts `unit`, `syslog_identifier`,
`container_name`, `level`), then through `loki.process` (drops empty lines,
adds static labels), and finally writes to Loki via `loki.write`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the development workflow and
guidelines. Key rules:

- **Never commit or push directly to `main`** -- always use a feature branch
- **All changes require a pull request** with at least one review
- Run `shellcheck`, `hadolint`, and `yamllint` before pushing

## Related

- Design context: [Whizzlefred/homelab#33](https://github.com/Whizzlefred/homelab/issues/33)
- Reference implementation: [ecohash-co/ha-addon-alloy](https://github.com/ecohash-co/ha-addon-alloy) (MIT)
- [Grafana Alloy documentation](https://grafana.com/docs/alloy/latest/)
- [HAOS add-on development docs](https://developers.home-assistant.io/docs/add-ons)

## License

MIT
