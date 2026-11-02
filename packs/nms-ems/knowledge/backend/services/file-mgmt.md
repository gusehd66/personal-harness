# File_mgmt Service (skeleton — 미구현)

## 서비스 책임
파일 관리(File Management) 담당으로 예약된 마이크로서비스 **골격(skeleton)**. 현재 REST 처리 로직과 WorkType 정의가 모두 비어 있어 실질 기능이 없다. `processMicroServiceRestRequest()`는 무조건 `null` 반환, `File_mgmtCD.WorkTypeBook`은 전부 주석 상태다. 기능 요청이 오면 "수정"이 아니라 "최초 구현"으로 접근한다.

## 시작점과 실행 설정
- `ServiceStart_File_mgmt.main()` → `tServiceName="file_mgmt"` → `runMicroService.run("file_mgmt", new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP 포트: `home/config/file_mgmt.xml`의 `<port>10453</port>`
- 별도 ServerConfig 구현 없음 — config XML 기반으로 runMicroService가 플러그인/설정 자동 로드

## 아키타입
분류 불가(skeleton). ②③ 등록 메커니즘이 아직 정해지지 않았다. 구현 시 `../feature-extension.md`의 아키타입 중 선택 — DB CRUD 중심이면 A(unify) 패턴, 단순 Worker면 유사 구조 서비스(`my5g.Service.SoftwareUpgrade_mgmt`)를 참고한다.

## REST contract
request type 없음. `getContentList()`는 `super.getContentList()`만 반환(TODO stub). 상속: `RestInvokerFunction_File_mgmt` → `AbsRestInvokerFunction_Service` 직접 상속 — my5G 공통 `AbsRestInvokerFunction_my5GService` 미사용(구현 시 변경 검토).

## 주요 Worker/Function 흐름
`Worker_File_mgmt`만 존재. `initialize()`에서 `WorkerLocalRestSessionManager`, `WorkerRestContentManager` 두 워커를 WorkflowMap에서 조회할 뿐 비즈니스 로직 없음.

## DB/Mongo/파일 흐름
없음. `CommonMybatisDBConfiguration`은 주입되어 있으나 mapper XML/데이터 모델/Mongo 코드가 전무하다.

## scheduler 흐름
없음.

## 기능 확장점 (feature seams) — 구현 시 절차
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| data-model | `File_mgmtCD.WorkTypeBook`에 WorkType 상수(예: `public static String Upload = "Upload";`) 정의 — 현재 전부 주석 | `common/data/File_mgmtCD.java` |
| ① rest-request-type | `getContentList()`에서 `super.getContentList()` 결과에 새 `RestContent` 추가 + `processMicroServiceRestRequest()`에 `aRequestTypeCode` 분기 추가(현재 null 반환 — `httpJsonResponseData` 반환 로직 필수) | `function/RestInvokerFunction_File_mgmt.java` |
| ② worker-function | `Worker_File_mgmt.initialize()`에 `m_WorkflowMap.getWorker(...)` 의존 추가, 또는 `abstractWorker` 상속 신규 Worker 작성 후 plugin.inf/ServerConfig 등록 | `worker/Worker_File_mgmt.java` |
| ④ db-mapper | mapper 신규 작성(MyBatis 인터페이스 + XML). `CommonMybatisDBConfiguration`이 이미 전달되어 있어 EmpContext로 사용 가능. 패턴은 `my5g.Service.SoftwareUpgrade_mgmt` 참고 | `ServiceStart_File_mgmt.java` |

## product 확장점
없음. product 폴더(my5G/openhaul/sixgen-ems)에 대응 구현 없음.

## mock/fake 지점
- `processMicroServiceRestRequest()` — null 반환 중이라 mock 없이 직접 stub 가능
- `WorkerLocalRestSessionManager` / `WorkerRestContentManager` — WorkflowMap mock 주입으로 대체

## 변경 위험
- 클라이언트 호출 시 무응답(null) — 첫 구현 시 응답 반환 로직 필수
- `Worker_File_mgmt.initialize()`가 WorkflowMap에서 두 워커를 조회 — 로컬 테스트 시 미등록이면 NPE
- mapper 없는 상태에서 DB 연결 설정 오류 시 기동 실패 가능

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.File_mgmt/`
- `src/main/java/hfr/plugin/my5g/service/File_mgmt/ServiceStart_File_mgmt.java`
- `src/main/java/hfr/plugin/my5g/service/File_mgmt/function/RestInvokerFunction_File_mgmt.java`
- `src/main/java/hfr/plugin/my5g/service/File_mgmt/worker/Worker_File_mgmt.java`
- `src/main/java/hfr/plugin/my5g/service/File_mgmt/common/data/File_mgmtCD.java`
- `home/config/file_mgmt.xml` (port 10453)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/abc/AbsRestInvokerFunction_Service.java`
