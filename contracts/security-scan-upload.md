# 보안 스캔 결과 S3 업로드

GitHub Actions의 보안 스캔(Trivy, gitleaks) 결과를 OPS로 넘기는 방법입니다. 러너는 클러스터 밖이라 ClusterIP인 OPS api에 닿지 않습니다. 그래서 CI는 도구 JSON을 S3에 올리고, 클러스터 안 OPS CronJob이 가져와 security DB에 넣습니다. 클러스터 안에서 도는 kube-bench는 이 경로를 쓰지 않고 지금처럼 OPS api로 바로 보냅니다.

| 쓰는 쪽 | 하는 일 | 담당 |
| --- | --- | --- |
| SERVICE, OPS, INFRA의 GitHub Actions | 스캔 JSON을 자기 레포 prefix 아래에 올린다 | 각 레포 담당 팀 |
| INFRA Terraform | 버킷, 쓰기 역할(GitHub OIDC), 읽기 역할(Pod Identity)을 만든다 | 인프라팀 |
| OPS `python -m app.security.ingest_s3` | 새 객체를 읽어 파싱하고 security DB에 넣는다 | 인프라팀(보안 보고 경로는 보안팀) |

형식을 바꿀 때는 세 팀이 함께 리뷰합니다.

## 버킷

| 항목 | 값 |
| --- | --- |
| 이름 | `maemusae-security-scans-<account_id>` (INFRA Terraform 키 `security-scans`) |
| OPS 환경변수 | `SECURITY_SCAN_BUCKET` |
| lifecycle | 90일 뒤 객체 삭제. 이미 가져온 결과는 security DB에 남는다 |

## 객체 키

```
<repo>/<tool>/<YYYY-MM-DD>/<github_run_id>.json
```

| 자리 | 값 |
| --- | --- |
| `repo` | `SERVICE-watch-nyang`, `OPS-watch-nyang`, `INFRA-watch-nyang` 중 하나 |
| `tool` | `trivy`, `gitleaks` 중 하나 |
| `YYYY-MM-DD` | 스캔한 날(UTC) |
| `github_run_id` | `$GITHUB_RUN_ID` |

본문은 도구의 JSON 출력을 바꾸지 않고 그대로 올립니다. 감싸거나 줄이지 않습니다.

| `tool` | 도구 명령 | 본문 모양 |
| --- | --- | --- |
| `trivy` | `trivy image --format json` 또는 `trivy fs --format json` | 객체(`Results`) |
| `gitleaks` | `gitleaks detect --report-format json` | 배열 |

한 실행에서 같은 도구를 두 번 돌리면(예: 이미지 둘) 키가 겹쳐 뒤의 것이 앞의 것을 덮습니다. 그럴 때는 하나의 JSON으로 합치거나 실행을 나눕니다.

## 권한

| 주체 | 권한 |
| --- | --- |
| 각 레포의 GitHub OIDC 역할 | `s3:PutObject`, 자기 `<repo>/` prefix 아래만 |
| OPS CronJob(namespace `incident`, ServiceAccount `ops-security-ingest`, EKS Pod Identity) | 그 버킷에 `s3:ListBucket`, `s3:GetObject` |

정적 AWS 키는 두지 않습니다.

## OPS가 가져가는 방법

- 버킷 전체를 나열하고, 아직 가져오지 않은 키만 읽습니다. 가져온 키는 security DB에 unique로 남겨 몇 번을 돌려도 한 번만 들어갑니다.
- 결과의 대상(`target`)은 `repo`, 실행 링크는 `https://github.com/saebyeok-baesong/<repo>/actions/runs/<github_run_id>`입니다.
- 키나 본문 형식이 틀린 객체는 사유를 남기고 다시 읽지 않습니다. 고쳐 올리려면 새 `github_run_id`로 올립니다.
- gitleaks 결과는 모두 CRITICAL입니다. 비밀번호나 키가 커밋에 들어간 것 자체가 바로 교체할 일입니다. 찾은 비밀값(`Secret`, `Match`)은 저장하지 않습니다.

gitleaks 출력에는 찾은 비밀값이 들어 있습니다. 보내는 쪽은 이 파일을 S3 말고 다른 곳(Actions 아티팩트, 로그)에 남기지 않습니다.

## GitHub Actions 예시

역할 ARN은 INFRA Terraform 출력값을 레포 변수(`vars.SECURITY_SCAN_ROLE_ARN`)로 넣습니다.

```yaml
permissions:
  id-token: write   # GitHub OIDC
  contents: read

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.SECURITY_SCAN_ROLE_ARN }}
          aws-region: ap-northeast-2
      - name: gitleaks
        run: |
          # 찾은 것이 있어도 결과를 올려야 하므로 종료 코드로 Job을 멈추지 않는다
          gitleaks detect --report-format json --report-path gitleaks.json --exit-code 0
      - name: upload to S3
        env:
          BUCKET: maemusae-security-scans-${{ vars.AWS_ACCOUNT_ID }}
        run: |
          key="${GITHUB_REPOSITORY#*/}/gitleaks/$(date -u +%F)/${GITHUB_RUN_ID}.json"
          aws s3 cp gitleaks.json "s3://${BUCKET}/${key}" --content-type application/json
          rm -f gitleaks.json
```

Trivy도 같습니다. `trivy image --format json --output trivy.json "$IMAGE"` 뒤에 키의 `gitleaks`를 `trivy`로 바꿔 올립니다.
