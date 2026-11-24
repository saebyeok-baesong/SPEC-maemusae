# SPEC-maemusae

새벽배송 팀의 공용 문서 레포입니다. 코드는 아래 세 레포에서 개발하고, 이 레포에는 세 팀(풀스택, 인프라, 보안)이 함께 보는 기준 자료를 둡니다.

| 레포 | 담당 | 내용 |
| --- | --- | --- |
| [SERVICE-maemusae](https://github.com/saebyeok-baesong/SERVICE-maemusae) | 풀스택 | 매무새 앱 (SNACKCUP/maemusae 포크, Next.js + FastAPI) |
| [OPS-maemusae](https://github.com/saebyeok-baesong/OPS-maemusae) | 인프라 (보안 공동) | AI 장애 대응 자동화 (webhook, LangGraph 워커, 승인) |
| [INFRA-maemusae](https://github.com/saebyeok-baesong/INFRA-maemusae) | 인프라 (보안 공동) | Terraform, GitOps(Argo CD), IAM |
| SPEC-maemusae | 세 팀 공동 | 이 레포 |

## 들어 있는 것

| 경로 | 내용 |
| --- | --- |
| `adr/` | 설계 결정 기록. 세 팀에 영향이 있는 결정은 여기에 번호를 붙여 남깁니다 |
| `contracts/` | 레포 사이의 계약 (`images.yaml` 형식, 보안 스캔 S3 업로드, 알림 payload) |
| `fixtures/` | 여러 레포가 테스트에 함께 쓰는 예시 데이터 |

## 결정 기록

- [0001 레포 구조와 팀별 소유](adr/0001-repo-structure.md)
