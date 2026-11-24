# 0001 레포 구조와 팀별 소유

- 상태: 채택
- 날짜: 2026-11-24

## 배경

매무새(SNACKCUP/maemusae 포크)를 EKS에 올리고 AI 기반 장애 대응 자동화를 붙인다. 풀스택, 인프라, 보안 세 팀이 함께 작업한다. 배포는 Argo CD(GitOps)로 하고, 장애 대응 봇은 사람 승인 뒤에 롤백한다.

## 결정

레포를 네 개로 나눈다.

| 레포 | 담당 | 바뀌는 빈도 |
| --- | --- | --- |
| SERVICE-maemusae | fullstack | 매일. upstream(SNACKCUP)을 계속 따라간다 |
| OPS-maemusae | infra (보안 보고 경로는 security) | 자주 |
| INFRA-maemusae | infra (`gitops/security/`, `iam/`은 security) | 드물게, 리뷰를 엄격하게 |
| SPEC-maemusae | 세 팀 공동 | 결정이 있을 때 |

보안팀은 레포를 따로 갖지 않는다. 보안팀이 만드는 것(정책, kube-bench Job, NetworkPolicy, IAM)은 클러스터에 배포되는 설정이라, 레포를 나누면 Argo CD가 보는 GitOps 원천이 둘이 된다. 대신 CODEOWNERS로 경로를 소유해 그 경로의 변경은 보안팀 승인 없이 merge되지 않게 한다.

## 이유

- Argo CD가 읽는 설정은 앱 소스와 분리한다. 앱 레포에 두면 이미지 태그를 올리는 커밋이 앱 CI를 다시 돌리고, 배포 기록이 앱 히스토리에 섞인다.
- SERVICE는 포크라서 인프라 파일을 넣지 않는다. upstream을 sync할 때 충돌을 줄이려고 포크 수정은 새 파일 위주로 한다.
- 장애 대응 봇의 롤백 권한을 INFRA 레포 하나로 좁힌다. 봇 코드와 인프라가 한 레포면 봇이 자기 코드와 VPC 설정까지 쓸 수 있게 된다.

## 배포와 롤백 흐름

nerd-nam의 `image-pr.yml` 방식을 따른다.

1. SERVICE, OPS의 CI가 OIDC로 ECR에 이미지를 올린다. 정적 AWS 키는 두지 않는다.
2. CI가 INFRA의 이미지 정의 파일에서 자기 줄을 digest로 고치는 PR을 연다. 이때 쓰는 GitHub App(이미지 PR App)은 INFRA 쓰기 권한만 갖고 리뷰 규칙을 우회하지 않는다.
3. 사람이 PR을 merge하면 Argo CD가 sync한다.
4. 장애 대응 봇의 롤백도 같은 길로 간다. 봇이 이전 digest로 되돌리는 PR을 열고, Discord 승인이 곧 그 PR의 merge다. kubectl로 직접 되돌리지 않는다. Argo CD self-heal이 직접 바꾼 상태를 다시 되돌리기 때문이다. 롤백 PR은 이미지 PR App과 다른 GitHub App(롤백 App)이 열고 merge한다. Discord 승인이 사람 리뷰이므로 롤백 App은 INFRA `main` ruleset의 우회 목록에 들어가 리뷰 없이 merge한다.

## 결과

- 변경 하나가 PR 두 개로 나뉠 때가 있다. 예: SERVICE에 `/metrics`를 추가하면 INFRA에 ServiceMonitor도 추가한다.
- 레포 사이의 계약은 `contracts/`에 두고, 바꿀 때는 양쪽 담당 팀이 함께 리뷰한다.

## GitHub App

INFRA에 쓰는 GitHub App은 둘이다.

| App | 키를 갖는 곳 | 하는 일 | INFRA `main` 리뷰 규칙 |
| --- | --- | --- | --- |
| 이미지 PR App | SERVICE, OPS의 GitHub Actions(조직 secret) | 새 digest로 `images.yaml`의 자기 줄을 바꾸는 PR을 연다 | 따른다. 사람이 리뷰하고 merge한다 |
| 롤백 App | 클러스터 안 OPS(Secrets Manager `maemusae/github-rollback-app`) | 롤백 PR을 열고 Discord 승인 뒤 merge한다 | 우회한다. Discord 승인이 사람 리뷰다 |

둘로 나누는 이유는 다음과 같다.

- 키를 갖는 곳이 다르다. 한쪽 키가 새도 다른 쪽 권한은 드러나지 않는다.
- 리뷰 규칙을 우회하는 권한은 롤백 App에만 있어야 한다. App이 하나면 CI에 있는 키로도 리뷰 없이 `main`에 merge할 수 있다.

롤백 App은 사람 확인 없이 merge할 수 있으므로 범위를 최소로 둔다.

- 설치 범위는 "Only select repositories"로 INFRA-maemusae 하나다.
- 권한은 Contents(Read and write), Pull requests(Read and write), Metadata(Read)만 준다. Workflows, Administration, Actions, Secrets를 비롯한 다른 권한은 주지 않는다. Webhook은 끈다.
- ruleset 우회는 이 App 하나에만, INFRA `main`에만 준다. 우회 범위를 넓히려면 보안팀 리뷰를 받는다.
- GitHub App 권한은 파일 단위로 좁힐 수 없다. 그래서 OPS 코드가 merge 직전에 PR이 `gitops/apps/images.yaml`의 그 앱 한 줄만 바꾸는지, base가 `main`인지 확인하고, 아니면 merge하지 않는다.
