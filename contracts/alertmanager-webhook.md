# Alertmanager 경보 → OPS 장애 대응 봇

- 보내는 쪽: INFRA (`gitops/`의 PrometheusRule, Alertmanager 설정)
- 받는 쪽: OPS (webhook)
- 바꿀 때: infra, security 함께 리뷰

## 형식

Alertmanager webhook payload `version: "4"`를 그대로 받습니다. 예시는 [fixtures/alertmanager](../fixtures/alertmanager)에 있습니다.

## 모든 경보 규칙이 붙여야 하는 라벨

| 라벨 | 값 | OPS가 쓰는 곳 |
| --- | --- | --- |
| `alertname` | 규칙 이름 | 분류, 중복 묶기 |
| `severity` | `critical` / `warning` / `info` | `critical`만 롤백을 제안합니다 |
| `namespace` | 예: `maemusae` | 원인 분석 범위 |
| `app` | `backend` / `frontend` | 롤백 대상. INFRA 이미지 정의 파일의 키와 같은 값이어야 합니다 |

`app`이 없는 경보는 알림만 보내고 롤백은 제안하지 않습니다.

## 주석

| 주석 | 필수 | 내용 |
| --- | --- | --- |
| `summary` | 예 | 한 줄 요약. Discord 장애 알림 제목에 씁니다 |
| `description` | 아니요 | 수치와 임계값 |

## 서비스 지표

SERVICE 백엔드가 `/metrics`로 낼 Prometheus 지표입니다 ([SERVICE #4](https://github.com/saebyeok-baesong/SERVICE-watch-nyang/issues/4)에서 요청). 경보 규칙은 이 이름에 기댑니다:

| 지표 | 라벨 |
| --- | --- |
| `http_requests_total` | `handler`, `method`, `status` (`2xx`, `4xx`, `5xx`로 묶임) |
| `http_request_duration_seconds` | `handler`, `method` |

`/health`, `/health/ready`, `/metrics` 요청은 세지 않습니다. `/metrics`는 클러스터 안에서만 수집하고 Ingress에 열지 않습니다.
