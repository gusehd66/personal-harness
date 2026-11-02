# 현재 서버 메뉴별 UI 관찰 기록

## 기준
- 확인일: 2026-06-10
- 대상: Chrome에 열린 `6G Flexible EMS`
- 방식: 좌측 메뉴와 내부 메뉴를 직접 클릭하되, 설정/수정/삭제/저장/적용/다운로드/명령 실행은 하지 않고 조회 화면 구조만 확인

## 좌측 Product Menu 관찰
| 메뉴 | URL | UI 유형 | 주요 관찰 |
| --- | --- | --- | --- |
| `Dashboard > Topology` | `/unms/dashboard/topology/2` | Workspace/Canvas | site/group tree, topology canvas, workspace toolbar, 우측 Detail panel. object 선택 시 `Parent Group`, `Name`, `Description` 표시. |
| `Equipment > Equipment` | `/unms/equipment/equipment` | Table/List | `Total List : 7 rows`, `Group NE Change`, `Manual NE Discovery`, table columns: `Name`, `Status`, `Model`, `Serial No.`, `IP`, `Version`, `Description`, `Action`. |
| `Equipment > ShelfView` | `/unms/equipment/shelf-view` | Workspace/Canvas | site/group/NE tree, shelf 시각화, 우측 Detail panel. unit 정보와 command 관련 항목 표시. |
| `Fault > Active Alarm` | `/unms/fault/current-alarm` | Search + Table | alarm count/status 영역, filter 조건, active alarm table. columns: `Severity`, `Network Group`, `Part`, `HostName`, `Location`, `Alarm Type`, `Reason`, `Creation Time`, `Clear Time`, `Ack Time`, `SA/NSA`, `Category`. |
| `NE Configuration > Configuration` | `/unms/ne-configuration/configuration` | Select Required + Nested Tree | 진입 직후 `Select the NE.`가 표시된다. NE 선택 후 내부 configuration tree와 form/table 화면이 열린다. 세부 내용은 `configuration-workspace.md`를 따른다. |
| `NE Configuration > MEF Service Management` | `/unms/ne-configuration/mef-service-management` | Topology + Table | 좌측 topology 선택 영역, 우측 `Selected NE`, `Slot ID`, `MEF Service List`. columns: `Slot ID`, `Name`, `Category`, `Type`, `Vlan ID`, `Description`, `Action`. |
| `Security > User` | `/unms/security/local-user` | Table/List | `Total List : 8 rows`, user table. columns: `User ID`, `First Name`, `Last Name`, `Role`, `Company Index`, `Department`, `Mobile`, `Email`, `Action`. |
| `Security > User Session` | `/unms/security/user-session` | Table/List | login session table. columns: `Last Login Time`, `User Name`. |
| `History > Alarm History` | `/unms/history/alarm` | Search + Table | alarm history filter와 table. Active Alarm과 유사한 alarm columns를 사용한다. |
| `History > Performance` | `/unms/history/performance` | Search + Table | `Network Type`, `Category`, `Item`, `Unit Type`, `Slot`, `Port`, `Search Period`, `Start`, `End` 조건과 결과 table. 초기 no-data DOM은 `Collect Time`, `Host Name`, `gNB ID`, `Cell ID`를 표시하고 Search 이후 Six-Gen 결과는 `Collect Time`, `Host Name`과 metric columns를 사용한다. `gNB ID`, `Cell ID`는 graph series identity 계산에 사용한다. |
| `Preference` | `/unms/preference/general/ftp-server` | Preference/Form | 내부 설정 tree와 우측 form/table content로 구성된다. 세부 내용은 `preference.md`를 따른다. |

## NE Configuration 내부 관찰
세부 기준 문서: `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/configuration-workspace.md`

NE 선택 후 관찰된 내부 tree:

- `System`
- `PM Report`
- `Port > Port Status`
- `Port > Port Statistics`
- `Port > Unit Inventory`
- `Port > Module Inventory`
- `SyncE`
- `Service Interface`
- `Time Configuration`
- `IEEE 1588v2 > Global-Configuration`
- `IEEE 1588v2 > Clock-Configuration`
- `IEEE 1588v2 > Status & Statistics`
- `Fault > Active Alarm`
- `Performance > Performance`

## Preference 내부 관찰
세부 기준 문서: `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/preference.md`

관찰된 내부 tree:

- `General > FTP Server`
- `General > DB Backup`
- `General > Storage Retention Period`
- `Fault > Alarm Definition`
- `System > Log Level`
- `System > Server Log Download`
- `System > System Info`

## 공통 관찰 원칙
- 메뉴 노출 여부는 메뉴 정의만으로 단정하지 않고 provider import, `ContentsManagerService` filtering, `authorityCheck` 순서로 확인한다.
- table 화면은 toolbar/filter, `Total List`, table header, `Action` column 존재 여부를 함께 기록한다.
- workspace 화면은 좌측 tree, 중앙 canvas/workspace, 우측 detail panel을 분리해서 기록한다.
- `Action` column과 icon-only button은 조회 작업 중 직접 클릭하지 않는다.
- 설정 화면은 label/table 구조만 확인하고 값 변경, 저장, 적용, 다운로드, 명령 실행은 하지 않는다.
