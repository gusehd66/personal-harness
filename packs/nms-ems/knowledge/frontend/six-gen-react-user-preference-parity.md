# Six-Gen React User Preference Parity

## 범위

Angular `six-gen-flexible`의 `/user-preference`를 React로 이식할 때 사용한다.
기준 viewport는 `1440x960`이며 모바일은 범위 밖이다.

## Angular 구성 계약

- route component는 `UserPreferenceComponent`이다.
- 기본 UI는 Security의 `ScreenLock`이다.
- Fault provider는 `E_HomeUIID.UserPreference`에 `AlarmSoundConfig` UI를
  추가한다.
- React에서도 Screen Lock 다음에 Alarm Sound가 표시되어야 하며 별도 inner
  title이나 global content toolbar를 추가하지 않는다.

## Request 계약

두 panel 모두 `POST rest-api/system/v1/user-profile`을 사용한다.

- Screen Lock: `type=Security`, `target=ScreenLock`
- Alarm Sound: `type=Fault`, `target=AlarmSoundConfig`
- Get은 Angular UserProfile 기본 필드 `id=0`, `user_id`, `type`, `target`을
  보존한다.
- profile이 없으면 Add, 있으면 Modify를 사용한다.
- 변경되지 않은 Apply는 request를 보내지 않는다.
- header `access_token`과 body `client_session_id`는 모두 WebSocket 첫
  keepalive 이후 활성화된 communication `session_id`이다.

## UI와 runtime 계약

- Screen Lock 선택값은 15, 30, 60, 90, 120분이며 기본값은 30분이다.
- Alarm Sound 선택값은 3, 5, 10초이며 기본값은
  `{ on: false, interval: 3 }`이다.
- Default는 question dialog의 Yes 이후 local draft만 바꾸며 mutation을
  보내지 않는다.
- alarm Occur는 severity별 원본 WAV를 loop 재생한다. Clear는 무시하고 설정
  시간이 지나면 pause 후 `currentTime=0`으로 되감는다.
- 공통 치수와 상태색은 `src/styles.css`의 `--preference-user-*`,
  `--wc-combobox-*`, `--wc-switch-*` token으로 선언한다.
- ComboBox/Switch disabled 상태와 Switch `0.3s` 전환은 Angular WC variable과
  같아야 한다.

## 검증

```powershell
cd D:\sixgen-react
npm run typecheck:react
npx vitest run src/screens/user-preference/screenLockModel.test.ts src/screens/user-preference/alarmSoundModel.test.ts src/screens/user-preference/alarmSoundRuntime.test.ts --maxWorkers=1
npm run build:six-gen-flexible-dev
npm run qa:six-gen-user-preference
npm run qa:six-gen-user-preference-live
npm run qa:six-gen-alarm-sound-runtime-live
```

live gate는 Angular와 React에 순차 로그인해 Get/Add contract, session field,
Default, options, buttons, geometry, computed style, browser error를 비교한다.
mutation은 browser에서 intercept하고 backend preference를 변경하지 않는다.

## 완료 판단

이 gate 통과는 User Preference desktop slice만 검증한다. 모든 활성 메뉴와
inner menu의 route, authority, API, action, dialog/table/form, screenshot parity가
통과하기 전에는 전체 이식 완료나 Angular runtime 제거로 처리하지 않는다.
