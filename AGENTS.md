# AI Agent Instructions for elk-stack

> **Workspace context**: This is part of a multi-project workspace. See root [AGENTS.md](../AGENTS.md) for overall architecture, data flow, and startup order.

## Overview

ELK stack for log collection, storage, and alerting. Elasticsearch + Logstash + Kibana + ElastAlert2.

## Commands

- **Start stack**: `docker compose up -d` (starts ES, Logstash, Kibana, ElastAlert2)
- **First-time setup**: `docker compose up setup` to initialize users
- **Stop**: `docker compose down` (add `-v` to remove volumes)

## Configuration

- Copy `.env.example` to `.env`
- Key variables: `ELASTIC_PASSWORD`, `LOGSTASH_INTERNAL_PASSWORD`, `KIBANA_SYSTEM_PASSWORD`, `ES_PASSWORD` (for ElastAlert2)

## Ports

- Elasticsearch: 9200
- Logstash: 5044 (Beats), 50000 (TCP)
- Kibana: 5601

## ElastAlert2 rules

Located in `elastalert/rules/`:
- `webhook_test.yml` — fires on any log within 5 min, POSTs to `http://host.docker.internal:8000/webhook`
- `sql_injection_rule.yml` — matches SQL keywords, POSTs to external webhook.site (test only)

## Logstash pipeline

Simple pipeline in `logstash/pipeline/logstash.conf`:
- Input: Beats (5044) + TCP (50000)
- Output: Elasticsearch (no parsing filters configured yet)

## Gotchas

- Uses external Docker network `elk-stack` — must be created before starting: `docker network create elk-stack`
- On Linux, replace `host.docker.internal` with actual host IP or `172.17.0.1` in webhook URLs