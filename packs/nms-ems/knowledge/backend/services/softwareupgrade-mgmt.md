# SoftwareUpgrade_mgmt Service

## 서비스 책임
NE(장비) 소프트웨어 업그레이드 전 주기를 관리하는 core 서비스다.
- 소프트웨어 이미지 업로드/검증/삭제
- 업그레이드 실행/상태 조회/취소
- NE Reboot(업그레이드 후 분리 스텝)
- NE 초기화 시 자동 소프트웨어 업그레이드 트리거

주요 입력: REST request(이미지/업그레이드/검증), NE 초기화 이벤트, 하위 EMS/NE 상태 응답.
주요 출력: 하위 EMS/NE로의 executeRestAPI 위임 호출, WebSocket push 알림(WorkerNotifyClient), operation log.

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SoftwareUpgrade_mgmt/src/main/java/hfr/plugin/my5g/service/SoftwareUpgrade_mgmt/ServiceStart_SoftwareUpgrade_mgmt.java`

- `tServiceName = "softwareupgrade_mgmt"`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 서비스 전용 ServerConfig/mapper 없음 — DB는 공통 `CommonMybatisDBConfiguration` 주입.
- Worker 획득은 프레임워크 `WorkflowMap` 자동 스캔: `RestInvokerFunction_SoftwareUpgrade_mgmt` 초기화 시 `aWorkFlowMap.getWorker(Worker_SoftwareUpgrade_mgmt.class)`, `getWorker(WorkerLocalSystemData.class)`, `getWorker(WorkerNotifyClient.class)` 등으로 획득한다.

## 아키타입
**E (외부연동/southbound adapter)** — `../feature-extension.md` 참조.
- ② Worker 등록: 프레임워크 WorkflowMap 자동 스캔(플러그인 구조). 신규 Worker 클래스는 기존 `Worker_SoftwareUpgrade_mgmt`처럼 스캔 대상 패키지에 두고 `getWorker()`로 획득되는지 확인.
- ③ Scheduler: 별도 ScheduleAdapterRat job 없음. 유일한 주기 작업은 `RestInvokerFunction` 내부의 1시간 주기 aging timer(`checkUpgradeStateAging()`, SystemTimer 직접 등록)뿐이다.
- unify_* / .inf / manualWorkerLoader 미사용.

## REST contract
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SoftwareUpgrade_mgmt/src/main/java/hfr/plugin/my5g/service/SoftwareUpgrade_mgmt/function/RestInvokerFunction_SoftwareUpgrade_mgmt.java` (getContentList() L1081-1084)
- 상수 선언: `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` (RequestTypeStringBook L225-228)

| 상수 (RequestTypeStringBook) | URL 문자열 | 용도 |
| --- | --- | --- |
| `software_upgrade` | `upgrade` | 업그레이드 실행/상태조회/취소/Reboot (WorkType으로 분기) |
| `software_image` | `image` | 이미지 목록/삭제 |
| `software_image_upload` | `image-upload` | 이미지 파일 업로드 (InputStream 경로: `processMicroServiceRestRequestUsingInputStream()`) |
| `software_image_verify` | `image-verify` | 이미지 검증 (`SoftwareImageVerification`) |

- 4개 모두 `HttpMethodBook.POST`, `"v1"`, `ScopeTypeBook.Inner`(내부 전용).
- 실제 동작 분기는 request type + `CommunicatorCD.WorkTypeBook`(Check/Get/Add/Delete 등) 조합. WorkType 상수 보조 정의: `common/data/SoftwareUpgrade_mgmtCD.java` (DB 모델 아님).

## 주요 Worker/Function 흐름
근거:
- `.../worker/Worker_SoftwareUpgrade_mgmt.java`
- `.../function/RestInvokerFunction_SoftwareUpgrade_mgmt.java`
- `.../function/UpgradeStateChecker.java`, `.../function/LocalSoftwareUpgradeState.java`, `.../function/RebootStateChecker.java`

핵심 메서드 (Worker_SoftwareUpgrade_mgmt):
- `checkSoftwareState(aEMSID, aProductCode, aEMSNeIDs, tNeID, moduleType)` — NE 현재 소프트웨어 상태 조회
- `upgradeSoftware(aEMSID, aProductCode, aNes, aUpgradeWork, aFtpServerConfig, aSoftwareImage)` — 업그레이드 명령 위임
- `checkUpgradeState(...)` — 진행 상태 조회
- `rebootNEs(aEMSID, aProductCode, aNes, aImageFile, moduleID)` / `checkRebootState(...)` — Reboot 분리 스텝
- `isUseFTP(neID)` — NE 모델별 FTP 사용 여부

흐름 요약:
1. REST 요청 → `RestInvokerFunction.processMicroServiceRestRequest()`에서 request type/WorkType 분기.
2. Worker 메서드가 `m_WorkerRestContentManager.executeRestAPI(class, productCode.getServiceName(), requestType, json)`로 하위 EMS/NE 서비스에 위임(직접 장비 통신 없음).
3. 업그레이드 진행 상태는 메모리 내 `upgradeStates` EHashtable + `UpgradeStateChecker`/`RebootStateChecker`가 폴링 관리, `LocalSoftwareUpgradeState`가 로컬 상태 보관.
4. 상태 변경 시 `WorkerNotifyClient.sendNotifyToClient()`로 WebSocket push.
5. observer 패턴으로 동일 NE에 대한 다중 사용자 업그레이드 충돌 처리(findConflictingUpgrade/addObserver).

NOC bypass: `isManualNOCBypassRequest()` override(L180)와 `processBypassManualRequestHandler()`(L1369)가 **software_upgrade request type에만** bypass를 적용한다.

### NE 초기화 자동 업그레이드
근거: `.../function/NeInitializeProcess.java`, `.../function/NeInitializeWorkItem.java` (L86-104)

`NeInitializeWorkItem.processNeInitialize()`가 `checkNeedToUpgrade()` 후 NeCodeBook 모델 분기(MY5GCUDU/MY5GCU/MY5GDU/MY5GT1/MY5GT2/MY5GBOX_T2_OD, RU 모델, MY5GFHM)로 `processSoftwareUpgrade()`를 호출 → NE 초기화 → 자동 SW 업그레이드 → Reboot 연계.

## DB/Mongo/파일 흐름
- 전용 DB mapper/Mongo 클라이언트 **없음**. 공통 `CommonMybatisDBConfiguration` 사용.
- 기록은 `WorkerLocalBaseDataCommon.writeOperationLog_User/writeOperationLog_System` 경유(공통 프레임워크).
- 이미지 파일: `HttpFileReceiveProcess`가 업로드 수신 → `serverRunningConfig.getSoftwareImageLocalPath()` 하위에 저장. FTP 설정은 `FTPServerConfigProvider.getFTPServerConfig()`(static).
- 업그레이드 진행 상태는 DB가 아닌 메모리(EHashtable) — 아래 변경 위험 참조.

## scheduler 흐름
독립 scheduler job 없음. `RestInvokerFunction` 초기화 시 SystemTimer에 1시간 주기 `checkUpgradeStateAging()`(L1053)을 직접 등록해 오래된 업그레이드 상태를 정리한다.

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치 / 근거 |
| --- | --- | --- |
| ① REST request type | `MicroServiceCommonCD.RequestTypeStringBook`에 상수 선언 → `getContentList()`에 `new RestContent(this, <상수>, HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner, true)` 추가 → `processMicroServiceRestRequest()`(또는 파일 업로드면 `processMicroServiceRestRequestUsingInputStream()`)에 else-if 분기 추가. **bypass가 필요하면 `isManualNOCBypassRequest()`/`processBypassManualRequestHandler()`에도 분기 추가**(누락 흔함) | `function/RestInvokerFunction_SoftwareUpgrade_mgmt.java` L1081-1084, L167-267 |
| ② Worker/Function | `Worker_SoftwareUpgrade_mgmt`에 public 메서드 추가 후 RestInvokerFunction에서 `this.worker.newMethod()` 호출. 신규 Worker 클래스가 필요하면 WorkflowMap 등록 확인 필수 | `worker/Worker_SoftwareUpgrade_mgmt.java` L85-473 |
| ⑤ product override (신규 NE 모델) | 구조적 분리 없이 **인라인 NeCodeBook 분기** 패턴. 신규 모델 추가 시 (a) `Worker_SoftwareUpgrade_mgmt.checkSoftwareState()`/`upgradeSoftware()`/`isUseFTP()`, (b) `RestInvokerFunction.upgradeSoftware()`, (c) 자동 업그레이드 대상이면 `NeInitializeWorkItem.processNeInitialize()` — 3곳에 NeCodeBook.NEW_MODEL 케이스를 함께 추가(이미지경로 패턴·FTP사용여부·moduleType) | `function/RestInvokerFunction_SoftwareUpgrade_mgmt.java` L517-560, `worker/Worker_SoftwareUpgrade_mgmt.java` L97-125, L464-468, `function/NeInitializeWorkItem.java` L93-98 |
| data-model | `common/data/` 하위에 Request/Response/Work/Result 클래스 추가 — 기존 `SoftwareUpgradeWork`, `SoftwareUpgradeNeState`, `SoftwareCheckWork` 패턴. RestInvokerFunction에서 `JSON.jsonStringToRootObject`/`getFieldObject`로 역직렬화 | `common/data/` (SoftwareUpgradeWork.java 등) |
| external-adapter (신규 NE 프로토콜) | `m_WorkerRestContentManager.executeRestAPI(class, productCode.getServiceName(), requestType, json)` 호출 분기 추가 + 대상 서비스 측 requestType 상수(platformRatCD.RequestTypeStringBook) 추가 | `worker/Worker_SoftwareUpgrade_mgmt.java` L86-88, L133, L309, L338, L429 |

## product 확장점
product별 별도 폴더/서브클래스 없음. NeCodeBook 인라인 분기가 유일한 product 분기 수단이다(위 ⑤ seam). vran/gpon 전용 enum만 `common/vran/E_RDLCardType.java`, `common/gpon/E_RDLCardType_gpon.java`로 분리되어 있다. 지원 모델군: MY5G 계열(CUDU/T1/T2/BOX_T2_OD/FHM), POLAN 계열(OLT/ONU/AP), N_DAS 계열(TU/SU/RU), AR_WF7/WF6E 등.

## 죽은/미구현 코드
`function/old_LocalSoftwareUpgradeState.java`, `function/old_UpgradeStateChecker.java`, `worker/old_Worker_SoftwareUpgrade_mgmt.java` — 구버전 사본. 수정 요청 시 `old_` 없는 현행 파일만 대상으로 한다.

## mock/fake 지점
- `Worker_SoftwareUpgrade_mgmt.checkSoftwareState()/upgradeSoftware()/rebootNEs()` — `WorkerRestContentManager.executeRestAPI` 경유 → executeRestAPI를 mock하면 장비 없이 전 흐름 테스트 가능
- `FTPServerConfigProvider.getFTPServerConfig()` — static 메서드 (모킹 까다로움)
- `HttpFileReceiveProcess` — 파일 수신/저장, `serverRunningConfig.getSoftwareImageLocalPath()` 경로 의존
- `WorkerNotifyClient.sendNotifyToClient()` — WebSocket push

## 변경 위험
- **NeCodeBook 분기 분산**: 신규 모델 추가 시 RestInvokerFunction + Worker + NeInitializeWorkItem을 동시에 수정해야 함 — 한 곳 누락 시 특정 경로에서만 실패.
- **상태 휘발성**: `UpgradeStateChecker`가 메모리 내 EHashtable로 상태 관리 → 서버 재시작 시 진행 중 업그레이드 상태 소실.
- **NOC bypass 편중**: bypass 로직이 software_upgrade에만 적용 → 신규 request type에 bypass 누락 주의.
- **테스트성**: 1시간 aging timer가 SystemTimer 직접 등록이라 모킹 어려움.
- 다중 사용자 충돌 처리(observer)·Reboot 분리 스텝 등 1500라인+ function 클래스의 복잡도 자체가 회귀 위험.

## 근거 파일
`<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SoftwareUpgrade_mgmt/src/main/java/hfr/plugin/my5g/service/SoftwareUpgrade_mgmt/` 기준:
- `ServiceStart_SoftwareUpgrade_mgmt.java`
- `function/RestInvokerFunction_SoftwareUpgrade_mgmt.java`
- `worker/Worker_SoftwareUpgrade_mgmt.java`
- `function/NeInitializeProcess.java`, `function/NeInitializeWorkItem.java`
- `function/UpgradeStateChecker.java`, `function/RebootStateChecker.java`, `function/LocalSoftwareUpgradeState.java`
- `function/SoftwareImageVerification.java`
- `common/data/SoftwareUpgrade_mgmtCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
