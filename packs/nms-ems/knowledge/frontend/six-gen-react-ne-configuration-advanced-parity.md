# Six-Gen React NE Configuration Advanced Parity

## 범위

React 이식 대상은 다음 desktop 화면이다.

- `SyncE`: `/ne-configuration/configuration/sixgen_synce`
- `Service Interface`: `/ne-configuration/configuration/sixgen_service_interface`
- `Time Configuration`: `/ne-configuration/configuration/sixgen_time_configuration`

모바일은 이 parity 범위에서 제외한다. 기준 viewport는 `1440x960`이다.

## Slot 규칙

- SyncE와 Service Interface는 현재 Angular module과 같이 I200 Slot 패널을 표시한다.
- Time Configuration은 MCU 전용이므로 Slot 패널을 표시하지 않는다.
- Time은 `455 SlotList`에서 MCU slot을 찾고 없으면 `0`을 사용한다.
- NE/Slot 전환 뒤 도착한 이전 조회·설정 응답은 generation이 다르면 화면에 반영하지 않는다.

## Command 계약

| 화면 | Get | Set/Delete |
| --- | --- | --- |
| SyncE | `442 SyncE`, `443 SyncEPort` | `442`, `443` 변경 field만 전송 |
| Service Interface | `410 UNIInterface`, `409 NNIInterface` | create/modify `410`, `409`; delete `440`, `439` |
| Time | `469 DateTimeState`, `468 TimeZoneConfiguration`, `467 NtpConfiguration` | timezone `468`, date `470`, NTP create/delete `467` |

- SyncE operation metadata는 `Configuration / SyncE`다.
- Service editor Set은 Angular처럼 빈 operation metadata를 유지한다.
- Service delete는 `Configuration / Service Interface`다.
- NTP delete는 `fieldOperations: { server: 'Delete' }`를 유지한다.

## Preflight 규칙

- Service create는 shelf-only `410` 또는 `409`로 ID 중복을 확인한다.
- Service duplicate 조회 실패는 Angular 경고를 표시한 뒤 create를 재개한다.
- UNI/NNI의 `num-of-services > 0`이면 delete를 차단한다.
- Date 변경 시 `467`로 NTP 존재 여부를 먼저 확인한다.
- NTP가 있고 Date만 변경하면 차단한다.
- NTP가 있고 Date와 timezone을 함께 변경하면 확인 후 timezone만 적용한다.
- NTP preflight 조회 실패는 fail-open으로 Date Set을 장비에 전달한다.
- NTP create의 duplicate preflight 조회 실패도 fail-open으로 create를 진행한다.

## UI 계약

- Service editor 폭은 `600px`, NTP editor 폭은 `700px`다.
- UNI/NNI Action 컬럼은 Angular `isFixedColumn`처럼 오른쪽에 고정한다.
- visible/enabled/disabled 상태는 `NeConfiguration` authority의 `Editable` 여부를 따른다.
- advanced 화면 색상·표·editor 치수는 `DESIGN.md`의 `--ne-configuration-*` token을 사용한다.

## 검증

- 모델/API 테스트: `src/screens/ne-configuration/*Model.test.ts`, `advancedConfigurationApi.test.ts`
- 브라우저 QA: `npm run qa:six-gen-ne-configuration-advanced`
- 브라우저 QA는 command 순서, changed fields, confirmation, preflight 503 fallback, authority, stale response, fixed Action 컬럼, editor 폭을 검증한다.
- destructive action은 mock backend의 payload preflight까지만 실행한다.
