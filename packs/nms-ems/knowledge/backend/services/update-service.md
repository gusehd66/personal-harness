# update_service (소프트웨어 자동 업데이트)

## 서비스 책임
마이크로서비스 클라이언트(다른 서비스 프로세스)가 기동 시 자신의 JAR 라이브러리 목록을 보내오면, 서버 배포 경로(`DataPath.getUpdateDeployPath()`)의 최신 버전과 비교해 업데이트 필요 파일 목록·버전·기본 로그레벨을 반환하는 지원(support) 서비스. DB 없이 파일 시스템만으로 동작한다.

## 주요 입력
- `request-lib-version-list` POST 요청(서비스명 + 클라이언트 보유 JAR 목록)
- 배포 경로 하위 `{service_name}/{version}/*.jar` 파일과 `filelist.lst`(공용 lib 목록/로그레벨)
- Notification: `service_update` / `not_service_update` (업데이트 파일 복사 중 상태 제어)

## 주요 출력
- 업데이트 대상 `FileInfo`(name/version/size/location) 목록 + 기본 로그레벨 응답

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.update_service/src/main/java/hfr/plugin/my5g/service/update_service/ServiceStart_update_service.java`
- `tServiceName = "update_service"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP 포트 10460 (`home/config/update_service.xml`)
- `CommonMybatisDBConfiguration`은 framework 공통 전달용일 뿐, 서비스 내부에 Mapper/DAO 사용 코드 없음 — **DB 미사용**
- `initialized()`의 서비스 인증 요청(`request_ServiceAuthentication`) 코드가 전체 주석 처리 — session-mgmt 연동 없이 동작하는 특수 서비스

## 아키타입
**D (manualWorkerLoader)** — `../feature-extension.md` 참조.
- ②③ 등록 위치: `manual/manualWorkerLoader.processManualWorkerLoad()` — `aWorkflowMap.setWorker(new <Worker>(aWorkflowMap))` 패턴으로 `Worker4ExecuteRestAPI` / `WorkerLockManager` / `Worker4ProcessFunction` 수동 등록(각각 `isExistWorker()` 중복 체크 후)
- scheduler 없음. product 확장(`manualWorkerLoaderEx_*`) 서브클래스도 없음

## REST contract
근거: `function/RestInvokerFunction_update_service.java` `getContentList()`
- `request-lib-version-list` — POST, `ScopeTypeBook.Inner`. 상수는 `MicroServiceCommonCD.RequestTypeStringBook.Request_Lib_version_List`로 참조하나 실제 정의는 `platformRatCD.RequestTypeStringBook`(상속)에 있음
- 수신 Notification(`getReceive_NotificationTypes()`): `service_update` / `not_service_update` — 파일 복사 중 여부를 `m_UpdateSystem_Running` `UtilCache<String,Boolean>(2, 15)`에 put. 캐시 true(TTL 15초)면 요청 처리 대신 빈 응답 반환(복사 중 배포 경로 접근 차단)

## 주요 Worker/Function 흐름
근거: `worker/Worker_update_service.java`
1. `RestInvokerFunction_update_service.processMicroServiceRestRequest()` — `m_UpdateSystem_Running` 캐시 확인 후 request type 분기
2. `Worker_update_service.processGetLibVersion(serviceName[, requiredVersion])` — `getUpdateDeployPath()/{service}/` 하위 버전 디렉토리 중 최대 버전 선택 → `MFileManager.getFileList()`로 JAR 목록을 `FileInfo` map으로 구성
3. 같은 메서드에서 `filelist.lst`를 파싱해 공용 lib(`/common_path/lib` location) 항목을 map에 병합
4. `processGetDefault_LogLevel(serviceName)` — `filelist.lst`에서 서비스 기본 로그레벨 조회
- `FileInfo` 모델은 `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/FileInfo.java`

## DB/Mongo/파일 흐름
- DB/Mongo 없음. 데이터 접근은 전부 파일 시스템: `DataPath.getUpdateDeployPath()` 하위 `{service_name}/{version}/*.jar` + `filelist.lst`
- `filelist.lst` 파싱 예외는 `catch(Exception e){}` 묵살 — 파일 손상 시 오류 없이 빈/부분 목록 반환

## scheduler 흐름
없음.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, "새_type", HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner, false)` 추가 + `processMicroServiceRestRequest()` if-else 분기 추가. 상수는 `MicroServiceCommonCD.RequestTypeStringBook` 또는 `platformRatCD.RequestTypeStringBook`에 선언 | `function/RestInvokerFunction_update_service.java` |
| ② worker-function (아키타입 D) | 간단 확장: `Worker_update_service`에 `processXxx()` public 메서드 추가 후 RestInvokerFunction에서 `m_Worker.processXxx()` 호출. 신규 Worker: `abstractWorker` 상속 작성 후 `manualWorkerLoader.processManualWorkerLoad()`에 `aWorkflowMap.setWorker(new NewWorker(aWorkflowMap))` 등록 — **등록 누락이 흔한 실수** | `worker/Worker_update_service.java`, `manual/manualWorkerLoader.java` |
| notification-type | `getReceive_NotificationTypes()`에 `MicroServiceCommonCD.NotifyTypeStringBook` 상수 추가 + `received_NotificationData()`에 처리 분기 추가(비교는 `platformRatCD.NotifyTypeStringBook` 상수 사용에 주의) | `function/RestInvokerFunction_update_service.java` |
| data-model | `update_serviceCD.WorkTypeBook`에 WorkType 상수 추가(현재 완전히 비어 있음 — 확장 준비 상태). 파일 메타 구조 변경은 `uEMS.plugin.my5g.Common`의 `FileInfo` 확장 + `processGetLibVersion()` 반영 | `common/data/update_serviceCD.java` |

## product 확장점
없음. `RestInvokerFunction_update_service`는 `AbsRestInvokerFunction_Service` 단일 구현이며 product 폴더(my5G/openhaul/sixgen-ems) 분기 없음. EmsTypecode는 `productCD.ProductTypeCode.plugin_comm` 고정.

## 죽은/미구현 코드
- `initialized()` 내 서비스 인증 요청 코드 전체 주석 처리 — 코드 의도와 실제 동작 간 drift. 인증 흐름을 살리는 수정은 전체 재설계 수준
- `update_serviceCD.WorkTypeBook` — 완전히 빈 클래스(상수 0개)

## mock/fake 지점
파일 시스템 real 접근 없이 테스트하려면:
1. `DataPath.getUpdateDeployPath()` — 테스트용 임시 경로로 대체
2. `MFileManager.getFileList()` / `getDirectoryList()` — `processGetLibVersion()`의 파일 목록 호출 목킹
3. `filelist.lst` `FileInputStream` — `processGetDefault_LogLevel()` 입력 목킹

## fixture 기준
서비스명·requiredVersion 입력 / 배포 경로의 버전 디렉토리·JAR 목록 / `filelist.lst` 내용(정상·손상·부재) / `m_UpdateSystem_Running` 캐시 상태(true→빈 응답 케이스) / expected `FileInfo` map(버전 비교·공용 lib 병합 결과).

## 변경 위험
- `processGetLibVersion()`은 `DataPath.getUpdateDeployPath()` 경로에 완전 의존 — 경로 규칙 변경 시 전 서비스 업데이트 불가
- `m_UpdateSystem_Running` 캐시 TTL 15초 내 요청은 빈 응답 — 빠른 재시도 시나리오에서 클라이언트가 "업데이트 없음"으로 오판 가능
- `filelist.lst` 파싱 예외 묵살 — 파일 손상이 조용한 목록 누락으로 이어짐
- 인증 주석 처리 전제(무인증 Inner API)가 변경되면 전체 흐름 재설계 필요

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.update_service/src/main/java/hfr/plugin/my5g/service/update_service/`
- `ServiceStart_update_service.java`
- `function/RestInvokerFunction_update_service.java`
- `worker/Worker_update_service.java`
- `manual/manualWorkerLoader.java`
- `common/data/update_serviceCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.update_service/home/config/update_service.xml` (port 10460)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/FileInfo.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/common/data/platformRatCD.java` (`Request_Lib_version_List = "request-lib-version-list"`)
