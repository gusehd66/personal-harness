# Backend 서비스 지식 인덱스

## 목적
Backend 분석은 먼저 서비스 책임을 식별한 뒤, 해당 서비스 문서와 공통 backend 문서를 함께 읽는다.

서비스 문서는 workspace 프로젝트 매핑표가 아니라 NMS 기능 책임 단위다. 실제 구현 근거는 각 문서 안에 `<BACKEND_ROOT>` 기준 경로로 남긴다.

기능 추가/수정 요청은 항상 (1) 어느 서비스, (2) 그 서비스의 아키타입, (3) 5-seam 중 어디에 끼우는지 순서로 분해한다. 공통 절차·아키타입·등록 위치·죽은 코드 목록은 `../feature-extension.md`를 먼저 읽는다.

## 서비스 전체 맵
실제 `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/` 의 NMS-core 서비스 27개를 도메인 family로 묶었다. 아키타입은 `../feature-extension.md`의 A~E 분류(A=unify자동등록, B=코디네이터/디스패처, C=.inf로더, D=manualWorkerLoader, E=외부연동)다. 27개 전 서비스에 문서가 있다. `(stub)`은 skeleton/미구현 서비스의 요약 문서다.

| 서비스 (`my5g.Service.*` / `unms.Service.*`) | 도메인 family | 아키타입 | 문서 |
| --- | --- | --- | --- |
| `BaseData` | 기준데이터·토폴로지 | A | `basedata.md` |
| `Fault` | 장애·알림 | A·E | `fault.md` |
| `Notify_mgmt` | 장애·알림 | C | `notify-mgmt.md` |
| `Performance` | 성능·리포트 | A·E | `performance.md` |
| `Report_mgmt` | 성능·리포트 | D·E | `report-mgmt.md` |
| `Ne_Configuration` | 장비 형상·명령·발견 | E·B | `ne-configuration.md` |
| `NeDiscovery_mgmt` | 장비 형상·명령·발견 | B·E | `nediscovery-mgmt.md` |
| `SoftwareUpgrade_mgmt` | 장비 형상·명령·발견 | E | `softwareupgrade-mgmt.md` |
| `WebCli` | 장비 형상·명령·발견 | E | `webcli.md` |
| `NeBackupRestore_mgmt` | 백업·복원 | A·C | `nebackuprestore-mgmt.md` |
| `NeConfig_Backup` | 백업·복원 | B·C | `neconfig-backup.md` |
| `Database_Backup` | 백업·복원 | C | `database-backup.md` |
| `DBBackup_mgmt` | 백업·복원 | B | `dbbackup.md` |
| `Recovery` | 백업·복원 | B | `recovery.md` |
| `System` | 시스템·운영 | A | `system.md` |
| `SystemManagement` | 시스템·운영 | D·C·E | `system-management.md` |
| `update_service` | 시스템·운영 | D | `update-service.md` |
| `ulog_mgmt` | 시스템·운영 | C | `ulog-mgmt.md` |
| `File_mgmt` | 시스템·운영 | (skeleton) | `file-mgmt.md` (stub) |
| `UserManagement` | 사용자·세션·게이트웨이 | D·C | `user-management.md` |
| `Session_mgmt` | 사용자·세션·게이트웨이 | C | `session-mgmt.md` |
| `Gateway_Uems` | 사용자·세션·게이트웨이 | B·C·E | `gateway-uems.md` |
| `Work_mgmt` | 작업관리 | A | `work-mgmt.md` |
| `vCore_mgmt` | 코어망·특수장비 | E | `vcore-mgmt.md` |
| `embeddedEMS` | 코어망·특수장비 | B | `embedded-ems.md` |
| `apc_mgmt` | 코어망·특수장비 | (skeleton) | `apc-mgmt.md` (stub) |
| `em_emulator` | 코어망·특수장비 | D | `em-emulator.md` (stub) |

> 별도 제품군 `uCM.Service.*`, `License.Portal.*` 은 이 맵 범위 밖이다(NMS-core 우선). 필요 시 동일 기준으로 후속 분석한다.

Product별 장비 모델/protocol/PM/alarm/shelfview 차이는 `../product-profiles.md`에서 확인한다.

## 서비스 문서 작성 기준
서비스별 문서는 아래 항목을 기준으로 짧고 반복 가능하게 작성한다.

- 서비스 책임
- 주요 입력
- 주요 출력
- 시작점과 실행 설정
- 아키타입 (`../feature-extension.md` A~E)과 ②③ 등록 위치
- WorkflowMap 등록
- REST contract
- 주요 Worker/Function 흐름
- DB/Mongo/파일 흐름
- scheduler/queue/background 흐름
- 기능 확장점(feature seams) — 5-seam 중 이 서비스에서 자주 쓰는 추가 지점 표
- product 확장점
- 죽은/미구현 코드 (있으면)
- mock/fake 지점
- fixture 기준
- 변경 위험
- 근거 파일

## 분석 순서
1. 요청이 가리키는 서비스를 위 전체 맵에서 찾고, 해당 서비스 문서(있으면)를 먼저 읽는다.
2. 기능 추가/수정이면 `../feature-extension.md`로 아키타입·등록 위치·5-seam·죽은 코드를 확인한다.
3. 공통 시작/등록 흐름은 `../service-startup.md`, `../worker-process-flow.md`, `../rest-content-flow.md`를 함께 본다.
4. DB/Mongo 변경은 `../db-and-migration.md`를 함께 본다.
5. scheduler/PM/alarm/sync 변경은 `../scheduler.md`와 관련 서비스 문서를 같이 본다.
6. product별 차이는 `../product-profiles.md`에서 먼저 찾고, 실제 product source로 다시 확인한다.

## 공통 주의
- 실제 product source를 수정하지 않는 하네스 작업에서는 `<BACKEND_ROOT>`를 읽기 전용으로 취급한다.
- 공통 서비스 변경은 my5G, openhaul, sixgen-ems 전체 side effect를 먼저 확인한다.
- product별로 나눌 수 있으면 공통영역 직접 수정 전에 product function/worker/parser override를 우선 검토한다.
