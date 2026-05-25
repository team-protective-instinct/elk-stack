# 캡스톤 ELK Stack 실행 가이드

이 저장소는 캡스톤디자인 프로젝트에서 로그를 수집하고, 검색하고, 공격 탐지 알림을 보내기 위한 로컬 ELK Stack 환경입니다.

구성 서비스는 다음과 같습니다.

- **Elasticsearch**: 로그 저장 및 검색
- **Logstash**: Beats/TCP 입력 로그를 Elasticsearch로 전달
- **Kibana**: Elasticsearch 로그 조회 및 시각화 UI
- **ElastAlert2**: 탐지 규칙에 맞는 로그가 발생하면 Webhook으로 알림 전송
- **Elasticsearch MCP Server**: MCP 클라이언트에서 Elasticsearch를 조회할 때 사용하는 선택 서비스

## 1. 사전 준비

팀원 PC에 아래 프로그램이 설치되어 있어야 합니다.

- Docker Desktop 또는 Docker Engine
- Docker Compose v2
- Git

설치 확인:

```sh
docker --version
docker compose version
git --version
```

Docker Desktop을 사용하는 경우 메모리는 최소 4GB 이상 할당하는 것을 권장합니다. Elasticsearch가 포함되어 있어 메모리가 부족하면 컨테이너가 정상적으로 뜨지 않을 수 있습니다.

## 2. 저장소 받기

```sh
git clone <프로젝트 저장소 URL>
cd elk-stack
```

이미 저장소를 받은 경우에는 프로젝트 루트, 즉 `docker-compose.yml` 파일이 있는 디렉터리에서 명령을 실행하세요.

## 3. 환경 변수 파일 생성

`.env.example`을 복사해서 `.env` 파일을 만듭니다.

```sh
cp .env.example .env
```

기본 실행만 할 경우 `.env.example`의 `changeme` 값을 그대로 사용해도 됩니다. 단, 여러 명이 공유하는 서버나 외부에 노출되는 환경에서는 반드시 비밀번호를 바꾸세요.

중요한 값:

| 변수 | 용도 |
| --- | --- |
| `ELASTIC_VERSION` | Elastic Stack 이미지 버전 |
| `ELASTIC_PASSWORD` | Elasticsearch `elastic` 계정 비밀번호 |
| `LOGSTASH_INTERNAL_PASSWORD` | Logstash가 Elasticsearch에 접속할 때 쓰는 비밀번호 |
| `KIBANA_SYSTEM_PASSWORD` | Kibana가 Elasticsearch에 접속할 때 쓰는 비밀번호 |
| `ES_PASSWORD` | ElastAlert2가 Elasticsearch에 접속할 때 쓰는 비밀번호 |

현재 `elastalert/config.yaml`은 `ES_PASSWORD`를 사용합니다. `.env.example`에는 `ELAST_ALERT_PASSWORD`만 있으므로, ElastAlert2까지 실행하려면 `.env`에 아래 줄을 추가하세요.

```env
ES_PASSWORD='changeme'
```

`ELASTIC_PASSWORD`를 변경했다면 `ES_PASSWORD`도 같은 값으로 맞추면 됩니다.

## 4. Docker 네트워크 생성

이 프로젝트의 Compose 설정은 외부 Docker 네트워크 `elk-stack`을 사용합니다. 최초 1회만 생성하면 됩니다.

```sh
docker network create elk-stack
```

이미 존재한다는 메시지가 나오면 정상입니다.

## 5. 최초 초기화

Elasticsearch 내부 사용자와 권한을 초기화합니다.

```sh
docker compose up setup
```

초기화가 끝나고 `setup` 컨테이너가 정상 종료되면 다음 단계로 넘어갑니다.

## 6. 전체 서비스 실행

백그라운드에서 실행하려면 다음 명령을 사용합니다.

```sh
docker compose up -d
```

처음 실행할 때는 이미지를 빌드하고 내려받기 때문에 시간이 걸릴 수 있습니다.

실행 상태 확인:

```sh
docker compose ps
```

로그 확인:

```sh
docker compose logs -f elasticsearch
docker compose logs -f logstash
docker compose logs -f kibana
docker compose logs -f elastalert
```

## 7. 접속 정보

| 서비스 | 주소 | 설명 |
| --- | --- | --- |
| Kibana | http://localhost:5601 | 로그 조회/시각화 UI |
| Elasticsearch | http://localhost:9200 | Elasticsearch HTTP API |
| Logstash Beats 입력 | localhost:5044 | Filebeat 등 Beats 입력 |
| Logstash TCP 입력 | localhost:50000 | TCP 로그 입력 |
| Logstash Monitoring API | http://localhost:9600 | Logstash 상태 확인 |
| Elasticsearch MCP | http://localhost:8085/mcp | 선택 사항, MCP 클라이언트용 |

Kibana 로그인 기본값:

- 아이디: `elastic`
- 비밀번호: `.env`의 `ELASTIC_PASSWORD` 값, 기본값은 `changeme`

Kibana는 Elasticsearch보다 늦게 준비될 수 있습니다. `docker compose up -d` 이후 1~2분 정도 기다린 뒤 접속하세요.

## 8. 정상 동작 확인

Elasticsearch 확인:

```sh
curl -u elastic:changeme http://localhost:9200
```

`.env`에서 `ELASTIC_PASSWORD`를 바꿨다면 `changeme` 대신 바꾼 비밀번호를 넣으세요.

Logstash TCP 입력 테스트:

```sh
printf 'capstone elk test log\n' | nc localhost 50000
```

Kibana에서 로그를 확인하려면 Discover 메뉴에서 데이터 뷰를 생성해야 할 수 있습니다. Logstash 출력으로 들어간 로그는 일반적으로 Elasticsearch 인덱스에 저장됩니다.

## 9. ElastAlert2 탐지 규칙

탐지 규칙은 `elastalert/rules/` 아래에 있습니다.

- `sql_injection_rule.yml`: SQL Injection 탐지
- `bruteforce_rule.yml`: Brute Force 탐지
- `command_injection_rule.yml`: Command Injection 탐지
- `file_upload_webshell_rule.yml`: Web Shell 업로드 탐지
- `service_exploit_rule.yml`: 서비스 취약점 공격 탐지
- `xss_rule.yml`: XSS 탐지

ElastAlert2는 1분마다 규칙을 실행하고, 최근 15분 버퍼를 기준으로 Elasticsearch를 조회합니다.

현재 규칙들은 Webhook URL로 아래 주소를 사용합니다.

```text
http://host.docker.internal:8000/webhook
```

따라서 알림을 받으려면 호스트 PC에서 Webhook 서버가 `8000` 포트로 실행 중이어야 합니다. Linux 환경에서는 `host.docker.internal`이 동작하지 않을 수 있으므로 실제 호스트 IP 또는 `172.17.0.1`로 바꿔야 할 수 있습니다.

## 10. Elasticsearch MCP Server 사용

MCP 서버는 Compose에 포함되어 있으며 전체 실행 시 함께 올라옵니다.

헬스 체크:

```sh
curl http://localhost:8085/ping
```

MCP 엔드포인트:

```text
http://localhost:8085/mcp
```

외부에 공개하지 말고 로컬 개발용으로만 사용하세요.

## 11. 서비스 중지 및 초기화

컨테이너 중지:

```sh
docker compose down
```

컨테이너와 Elasticsearch 저장 데이터까지 모두 삭제:

```sh
docker compose down -v
```

데이터를 삭제하면 기존 인덱스와 로그가 사라집니다. 팀원과 공유 중인 데이터가 있다면 실행 전에 확인하세요.

## 12. 자주 발생하는 문제

### `network elk-stack not found` 오류

외부 네트워크가 없어서 발생합니다.

```sh
docker network create elk-stack
docker compose up -d
```

### Kibana 로그인이 안 됨

`.env`의 `ELASTIC_PASSWORD`와 초기화 시점의 Elasticsearch 비밀번호가 다를 수 있습니다.

개발 환경에서 데이터를 지워도 된다면 아래 순서로 다시 초기화하세요.

```sh
docker compose down -v
docker compose up setup
docker compose up -d
```

### ElastAlert2가 Elasticsearch 인증에 실패함

`.env`에 `ES_PASSWORD`가 있는지 확인하세요.

```env
ES_PASSWORD='changeme'
```

`ELASTIC_PASSWORD`를 변경했다면 `ES_PASSWORD`도 같은 값으로 맞추세요.

### 포트 충돌이 발생함

아래 포트를 다른 프로그램이 사용 중인지 확인하세요.

- `9200`, `9300`: Elasticsearch
- `5601`: Kibana
- `5044`, `50000`, `9600`: Logstash
- `8085`: Elasticsearch MCP Server

충돌하는 프로그램을 종료하거나 `docker-compose.yml`의 포트 매핑을 변경해야 합니다.

## 13. 팀원 실행 순서 요약

처음 실행하는 팀원은 아래 순서대로 실행하면 됩니다.

```sh
cp .env.example .env
```

`.env`에 아래 값 추가:

```env
ES_PASSWORD='changeme'
```

```sh
docker network create elk-stack
docker compose up setup
docker compose up -d
docker compose ps
```

그 다음 브라우저에서 Kibana에 접속합니다.

```text
http://localhost:5601
```

로그인:

- 아이디: `elastic`
- 비밀번호: `changeme`
