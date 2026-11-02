# NE Configuration 화면 구조

## 사용 시점
- `NE Configuration > Configuration` 화면을 설명하거나 매뉴얼 캡처를 갱신할 때 사용한다.
- NE 선택 후 나타나는 내부 configuration tree, table, form, performance/alarm 하위 화면을 확인할 때 사용한다.
- configuration UI 변경 시 어떤 화면 유형이 영향을 받는지 판단할 때 사용한다.

## 기본 구조
`Configuration` 화면은 진입 직후 바로 설정 항목이 보이지 않고, 먼저 NE 선택이 필요하다.

```text
Content Title: Configuration
Top NE Selector
Main Panel
  - before NE selection: "Select the NE."
  - after NE selection:
    - left configuration tree
    - right selected configuration content
```

## 현재 서버 관찰 기준
- 확인일: 2026-06-10
- 기준 화면: `6G Flexible EMS`
- 기준 NE: `SEBU1_NEW_COT`
- 조사 방식: 조회 기준으로만 확인. 저장, 적용, 삭제, 초기화, discovery, command 실행은 하지 않음.

## NE 선택 흐름
NE selector에는 `Model` 선택 영역과 site/group/NE tree가 함께 표시된다.

관찰된 항목:
- `A001001`
- `1`
- `2`
- `3`
- `SEBU1_NEW_COT`
- `SEBU1_NEW_RT1`
- `SEBU1_NEW_RT2`

## 내부 Configuration Tree
NE 선택 후 관찰된 내부 tree:

- `System`
- `PM Report`
- `Port`
  - `Port Status`
  - `Port Statistics`
  - `Unit Inventory`
  - `Module Inventory`
- `SyncE`
- `Service Interface`
- `Time Configuration`
- `IEEE 1588v2`
  - `Global-Configuration`
  - `Clock-Configuration`
  - `Status & Statistics`
- `Fault`
  - `Active Alarm`
- `Performance`
  - `Performance`

## 화면 유형별 관찰
| 내부 메뉴 | UI 유형 | 관찰 기준 |
| --- | --- | --- |
| `System` | Form/Info | `Hostname`, `Description`, `Type`, `Phys Address`, `Serial Num`, `MFG Name`, `Model Name`, `MFG Date`, CPU/Memory/Fan 상태 항목을 표시한다. |
| `PM Report` | Toggle/Form | `ON`, `OFF` 선택 값이 보인다. 값 변경은 하지 않는다. |
| `Port Status` | Table | `Port ID`, `Admin Status`, `Port Type`, `Port Alarm Status`, `Equip`, `Wavelength`, `Rx Power`, `Tx Power`, `Description`. |
| `Port Statistics` | Table | `Name`, ingress/egress octets, packets, discard, error 계열 통계 컬럼을 표시한다. |
| `Unit Inventory` | Table | `Slot ID`, `Name`, 제조사, 모델, serial, 제조일, hardware/software revision. |
| `Module Inventory` | Table | `Port ID`, 거리, wavelength, Tx/Rx power, bias current, voltage, temperature, manufacturer, serial/part/product code. |
| `SyncE` | Toggle + Table | `Enable`, `Disable`, `Slot ID`, `Name`, `SyncE Status`, `ESMC Rx/Tx Status`, `Priority`, `Tx-QL`, `Rx-QL`. |
| `Service Interface` | Table | UNI/NNI 서비스 인터페이스 table을 표시한다. `Action` column은 조회 중 클릭하지 않는다. |
| `Time Configuration` | Table | `Name`, `Association Type`, `Prefer`, `Status`, `Action`. |
| `Global-Configuration` | Toggle/Form | `Enable`, `Disable` 및 PTP global 설정 field를 표시한다. |
| `Clock-Configuration` | Table | `Slot Id`, `Port Number`, VLAN, interval, port type, time sync 관련 table을 표시한다. |
| `Status & Statistics` | Table | `Slot Id`, `Port Number`, `Out Pkts`, `In Pkts`, `In Discards`. |
| `Fault > Active Alarm` | Table | `Alarm Severity`, `Resource Name`, `Alarm Type`, `Cause`, `Time Created`. |
| `Performance > Performance` | Table | port/interface performance counter와 interval/time 기반 성능 데이터를 표시한다. |

## 변경 전 확인
- generic configuration component만 보지 말고 active site의 provider, route, file replacement를 먼저 확인한다.
- table column 변경은 매뉴얼, API response, backend contract, export/download 기능까지 영향을 줄 수 있다.
- toggle/form 화면은 조회 중에도 값 변경이 쉬우므로 매뉴얼 캡처 작업에서는 클릭 범위를 제한한다.
- `Action` column의 icon button은 기능을 단정하지 말고 코드 또는 tooltip 근거로 확인한다.

## 금지
- 매뉴얼/UI 조사 중 `Save`, `Apply`, `Delete`, `Initialize`, `Discovery`, `Command`, `Upgrade`, download/action 실행 버튼을 누르지 않는다.
- 설정값을 임의로 바꾸지 않는다.
