FROM grafana/promtail:latest

COPY promtail-config.yaml /etc/promtail/promtail-config.yaml

# Promtail natively supports env variable expansion
ENTRYPOINT ["/usr/bin/promtail", "-config.expand-env=true", "-config.file=/etc/promtail/promtail-config.yaml"]
