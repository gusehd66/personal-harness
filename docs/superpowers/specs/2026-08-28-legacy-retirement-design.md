# Legacy Harness 제거 및 새 구조 정착 설계

## 1. 목표

`C:\khd\ai-harness\harness\`와 `C:\khd\ai-harness\nms-knowledge\`를 최종 구조에서 제거한다. 필요한 범용 내용은 `core/`로, NMS/EMS 전용 절차·템플릿·지식은 `packs/nms-ems/`로 실제 이관한다.

최종 구조에서 legacy 폴더는 호환 경로로도 유지하지 않는다.

```text
C:\khd\ai-harness\
├─ core\
│  ├─ policy\
│  ├─ playbooks\
│  └─ templates\
├─ packs\
│  └─ nms-ems\
│     ├─ policy\
│     ├─ playbooks\
│     ├─ templates\
│     └─ knowledge\
├─ skills\
└─ docs\
```

## 2. 소유권

| 기존 내용 | 최종 위치 | 처리 |
| --- | --- | --- |
| `harness/policy/approval.md` | `core/policy/approval.md` | 현재 Core 문서를 canonical으로 사용 |
| `harness/policy/security.md` | `core/policy/security.md` + `packs/nms-ems/policy/domain-safety.md` | 범용 보안과 NMS 인증·세션 위험으로 분리 |
| `harness/policy/global.md` | `core/policy/global.md` + `packs/nms-ems/PACK.md` | 범용 원칙과 NMS 라우팅으로 분리 |
| `harness/playbooks/code-review.md` | `core/playbooks/code-review.md` | Core 문서를 canonical으로 사용 |
| `harness/playbooks/long-term-learning.md` | `core/playbooks/long-term-learning.md` | Core 문서를 canonical으로 사용 |
| `harness/playbooks/bugfix.md` | `packs/nms-ems/playbooks/bugfix.md` | NMS 위험·결과 형식을 새 경로로 이관 |
| `harness/playbooks/angular-ui-harness.md` | `packs/nms-ems/playbooks/angular-ui-harness.md` | 실제 파일 이동 |
| `harness/playbooks/backend-harness.md` | `packs/nms-ems/playbooks/backend-harness.md` | 실제 파일 이동 |
| `harness/playbooks/frontend-feature.md` | `packs/nms-ems/playbooks/frontend-feature.md` | 실제 파일 이동 |
| `harness/playbooks/angular-react-framework-modules-mapping.md` | `packs/nms-ems/playbooks/angular-react-framework-modules-mapping.md` | 실제 파일 이동 |
| `harness/templates/result-report.md` | `core/templates/result-report.md` | Core 문서를 canonical으로 사용 |
| `harness/templates/angular-harness-report.md` | `packs/nms-ems/templates/angular-harness-report.md` | 실제 파일 이동 |
| `harness/templates/backend-harness-report.md` | `packs/nms-ems/templates/backend-harness-report.md` | 실제 파일 이동 |
| `nms-knowledge/**` | `packs/nms-ems/knowledge/**` | 전체 트리 구조를 보존한 실제 이관 |

## 3. 내부 링크와 소비자 경로

모든 현재 사용 문서는 다음 규칙을 따른다.

- `nms-knowledge/...` 또는 `.ai/nms-knowledge/...` → `<HARNESS_ROOT>/packs/nms-ems/knowledge/...`
- `harness/playbooks/<nms-file>.md` → `<HARNESS_ROOT>/packs/nms-ems/playbooks/<nms-file>.md`
- `harness/templates/angular-harness-report.md`와 `backend-harness-report.md` → `<HARNESS_ROOT>/packs/nms-ems/templates/...`
- 범용 policy, code review, long-term learning, result report → `<HARNESS_ROOT>/core/...`

`C:\khd\playwright\AGENTS.md`, `C:\khd\allproject\uems_fe\AGENTS.md`, `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md`는 위 경로만 사용하도록 갱신한다.

`docs/superpowers/specs/`, `docs/superpowers/plans/`, `.harness/runs/`의 이전 경로 언급은 작업 이력이다. 이력 문서의 문자열은 살아 있는 링크로 취급하지 않으며, 역사 기록을 바꾸기 위해 수정하지 않는다.

프로젝트 로컬 `.ai/manual/`은 운영 매뉴얼 산출물이므로 이관 또는 삭제 대상이 아니다.

## 4. 민감정보 처리

legacy 지식을 새 Pack으로 복사하거나 이동하기 전에 값 형식 인증정보 후보를 값 노출 없이 점검한다.

- 실제 credential, cookie, private key, 운영 연결 문자열이면 마스킹한다.
- 유효한 실제 credential이면 문서 마스킹만으로 충분하지 않으며 별도 교체 권한과 절차가 필요하다.
- 코드 식별자, field name, 환경 변수 이름, 예시 placeholder이면 실제 값이 아님을 확인한 뒤 유지할 수 있다.
- 판정에는 파일·행·분류만 기록하고 값 자체는 실행 기록이나 결과 보고에 남기지 않는다.

민감정보 후보가 미해결이면 해당 파일은 Pack으로 이동하지 않고 이관을 중단한다.

## 5. 안전한 이관 순서

1. legacy 파일 인벤토리와 SHA-256 기준을 생성한다.
2. 민감정보 후보를 파일·행 단위로 판정한다.
3. 새 최종 경로에 복사해 파일 수와 hash가 맞는지 검증한다.
4. 새 경로 기준으로 skills, README, 세 프로젝트 `AGENTS.md`, 현재 사용 문서의 링크를 수정한다.
5. 현재 사용 경로에서 legacy 참조가 없는지 검사하고, 새 링크 대상이 모두 존재하는지 검사한다.
6. 현재 하네스의 static 검증과 각 skill validator를 실행한다.
7. 위 검증이 통과할 때만 `harness/`와 `nms-knowledge/`를 삭제한다.
8. 삭제 후 같은 경로·링크·skill 검증을 다시 실행한다.

삭제는 7단계 이전에 수행하지 않는다. 현재 하네스가 정상 Git 저장소가 아니므로, 1단계의 hash 기준은 삭제 전후 동일성을 확인하는 최소 복구 근거다.

## 6. 오류 처리

- 파일 수 또는 SHA-256이 다르면 삭제하지 않고 차이를 기록한다.
- 새 링크 대상이 없으면 해당 링크를 복구하거나 경로를 수정한 뒤 다시 검사한다.
- 민감정보 후보가 실제 값인지 판정할 수 없으면 삭제·이관을 중단하고 사용자에게 판정 권한을 요청한다.
- validator, 문서 경로 검사, placeholder 검사가 실패하면 legacy를 삭제하지 않는다.
- 제품 build·브라우저·API 검증은 이관이 문서·skill 경로만 바꾸는 범위에서는 필수 조건이 아니며, 실행하지 않은 사실을 결과에 남긴다.

## 7. 완료 기준

- `harness/`와 `nms-knowledge/`가 존재하지 않는다.
- 모든 이관 대상은 `core/` 또는 `packs/nms-ems/`에 하나의 canonical 원본만 갖는다.
- 현재 사용 문서와 세 소비자 `AGENTS.md`에서 legacy 경로가 발견되지 않는다.
- 새 경로의 파일 수와 SHA-256이 이관 기준과 일치한다.
- 신규·갱신 skill validator가 통과한다.
- 값 형식 민감정보 후보는 이동 전 판정 또는 마스킹 완료 상태다.
- 기존 `.ai/nms-knowledge` 참조는 현재 사용 문서에서 발견되지 않는다.

## 8. 비범위

- 프로젝트 로컬 `.ai/manual/` 삭제 또는 이동
- 제품 소스 코드 변경
- 외부 프로젝트 전체 탐색이나 알려지지 않은 소비자의 자동 수정
- Git 원격 생성, push, commit
- 실제 credential 교체. 실제 값이 확인되면 별도 승인 작업으로 다룬다.
