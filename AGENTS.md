# AI Agent Instructions for elk-stack

> **Workspace context**: This is part of a multi-project workspace. See root [AGENTS.md](../AGENTS.md) for overall architecture, data flow, and startup order.

## Role

Elasticsearch, Logstash, Kibana, ElastAlert2, and Elasticsearch MCP stack for collecting DVWA/Filebeat logs, storing searchable events, and POSTing alerts to the backend.

## Commands

- Start stack: `docker compose up -d`.
- First-time setup: copy `.env.example` to `.env`, then run `docker compose up setup` to initialize users.
- Stop stack: `docker compose down`.
- Avoid `docker compose down -v` unless the user explicitly wants volumes/data removed.

## Configuration

- Key `.env` variables include `ELASTIC_PASSWORD`, `LOGSTASH_INTERNAL_PASSWORD`, `KIBANA_SYSTEM_PASSWORD`, and `ES_PASSWORD` for ElastAlert2.
- `.env.example` may mention similarly named ElastAlert credentials; `elastalert/config.yaml` actually reads `${ES_PASSWORD}`.
- ElastAlert uses `TZ=Asia/Seoul`.

## Services and ports

- Elasticsearch: `9200`, `9300`
- Logstash: `5044` Beats, `50000/tcp`, `50000/udp`, `9600`
- Kibana: `5601`
- ElastAlert2: container-only service that reads rules from `elastalert/rules/`
- Elasticsearch MCP: `127.0.0.1:8085:8080`, local endpoint `http://localhost:8085/mcp`

Runtime services use the external Docker network `elk-stack`; create it first if missing:

```sh
docker network create elk-stack
```

## Logstash pipeline

Pipeline file: `logstash/pipeline/logstash.conf`.

- Inputs: Beats `5044` and TCP `50000`. Docker Compose exposes UDP `50000`, but the current Logstash pipeline does not consume UDP.
- Output: Elasticsearch using `logstash_internal` and `${LOGSTASH_INTERNAL_PASSWORD}`.
- No parsing filters are configured, so ElastAlert rules depend heavily on raw `message` content and Filebeat fields.

## ElastAlert2 rules

Current rules under `elastalert/rules/`:

- `bruteforce_rule.yml`
- `command_injection_rule.yml`
- `file_upload_webshell_rule.yml`
- `service_exploit_rule.yml`
- `sql_injection_rule.yml`
- `xss_rule.yml`

Common rule contract:

- Index pattern is `.ds-logs-generic-default-*`.
- Rules target DVWA logs with `(fields.service:"dvwa-apache" OR service.name:"dvwa-apache")`.
- Alerts POST to `http://host.docker.internal:8000/webhook`.
- Dynamic payload usually maps `timestamp: "@timestamp"` and `log_message: "message"`.
- Static payload must include backend-required fields such as `alert_name` and `severity`; rules also include `title` and `rule_name`.

ElastAlert cadence in `elastalert/config.yaml`:

- `run_every: 1 minute`
- `buffer_time: 15 minutes`
- `writeback_index: elastalert_status`

## Gotchas

- `host.docker.internal` works on Docker Desktop macOS/Windows. On Linux, replace it with the host's reachable IP, commonly `172.17.0.1`.
- Backend endpoint is `POST /webhook`.
- Keep `alert_name` and `severity` in static payloads; backend validation requires them.
- If Filebeat/Logstash field names change, update all ElastAlert rules consistently.

## Verification notes

- Use `docker compose config` after compose/config edits.
- Use container logs or Kibana/Elasticsearch queries to verify ingestion and ElastAlert behavior.
- No formal test suite is configured.
