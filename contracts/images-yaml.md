# 이미지 정의 파일 `gitops/apps/images.yaml`

INFRA 레포에서 각 앱이 지금 어떤 이미지로 떠 있어야 하는지 정하는 파일입니다. Argo CD는 이 파일을 읽어 배포하고, 장애 대응 봇은 이 파일의 git 이력으로 배포 시각과 직전 이미지를 알아냅니다.

| 쓰는 쪽 | 하는 일 |
| --- | --- |
| SERVICE, OPS의 CI | 새 이미지를 ECR에 올린 뒤 자기 앱 줄을 새 digest로 바꾸는 PR을 엽니다 |
| Argo CD | `main`의 이 파일대로 sync합니다 |
| OPS 장애 대응 봇 | 이력을 읽고, 롤백 때 한 앱 줄을 직전 digest로 되돌리는 PR을 열고, 승인되면 merge합니다 |

담당은 인프라팀입니다. 형식을 바꿀 때는 OPS 담당(인프라팀)과 CI를 고치는 담당(풀스택팀)이 함께 리뷰합니다.

## 형식

평평한 YAML 매핑입니다. 키는 앱 이름, 값은 digest로 고정한 이미지 주소입니다.

```yaml
# 앱 이름: 저장소@sha256:digest. 각 앱 레포 CI가 PR로 자기 줄을 고친다
web: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/web@sha256:3f1c...(64자)
api: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/api@sha256:9ab2...(64자)
ops: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/ops@sha256:77de...(64자)
```

규칙은 다음과 같습니다.

- 키는 소문자, 숫자, `-`만 씁니다(`^[a-z0-9][a-z0-9-]*$`). 키는 Deployment의 컨테이너 이름, 알림의 `app` 라벨과 같습니다.
- 값은 `<저장소>@sha256:<64자 16진수>`입니다. `:latest` 같은 태그는 쓰지 않습니다. digest여야 롤백이 정확히 같은 이미지로 돌아갑니다.
- 한 앱은 한 줄입니다. 값을 여러 줄로 나누거나 앵커(`&`, `*`)를 쓰지 않습니다. 봇은 그 줄 하나만 바꾸고 나머지 줄과 주석은 그대로 둡니다.
- 주석은 자유롭게 씁니다.
- 한 커밋에서 여러 앱 줄을 바꿔도 되지만, 앱 하나만 바꾸면 이력으로 배포 시각을 더 정확히 알 수 있습니다.

## 배포 이력을 읽는 방법

봇은 `main`에서 이 파일을 바꾼 커밋을 최신부터 읽습니다. 지금 값이 처음 들어온 커밋의 커밋 시각을 그 앱의 배포 시각으로, 그 바로 전 커밋의 값을 직전 이미지로 봅니다. PR merge 시각과 Argo CD sync 시각의 차이(보통 수 분)는 무시합니다.

## 롤백 커밋

봇이 merge하는 롤백 커밋의 제목에는 `[ops-rollback]`이 들어갑니다(예: `ops: api 롤백 [ops-rollback] (#12)`). 봇은 이 표시가 있는 배포 뒤의 장애에는 롤백을 다시 제안하지 않습니다. 직전 이미지가 방금 되돌린 문제 이미지이기 때문입니다. 사람이 손으로 되돌릴 때는 이 표시를 쓰지 않습니다.

## 예시 데이터

Alertmanager가 OPS로 보내는 payload 예시는 [`fixtures/alertmanager-webhook.json`](../fixtures/alertmanager-webhook.json)입니다. OPS 테스트는 그 사본을 씁니다.
