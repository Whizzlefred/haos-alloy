# Changelog

## 1.0.1

- Fix: correct journal field names in relabel rules (`syslog_identifier`, `container_name`)
- Fix: remove non-existent `PRIORITY_KEYWORD` level relabel rule (Loki `detected_level` handles this automatically)

## 1.0.0

- Initial release
- Ship systemd journal logs to Loki via Grafana Alloy v1.13.1
- Auto-detect journal path (`/var/log/journal` or `/run/log/journal`)
- Configurable Loki URL, log level, and additional Alloy config blocks
- Static labels: `host`, `env`, `source_type`, `service`, `retention`
- Journal field extraction: `unit`, `hostname`, `syslog_identifier`,
  `transport`, `container_name`, `level`
- Alloy debug UI on port 12345
- Support for `amd64` and `aarch64` architectures
