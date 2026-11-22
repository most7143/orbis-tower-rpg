# 3단계 테스트 보고서: 다이아 캐릭터 뽑기 + 다이아 지급 + 2단계 보완

> 작성: 테스트 에이전트, 2026-11-22
> 기준: `Phase-3-Gacha.md` 7절(T1~T26)과 완료 조건, `Phase-2-Character-UI.md` 끝 "2단계 보완"(3-A) 완료 조건
> 방법: 메이커 새로고침 → 빌드 로그 확인 → 플레이 → `maker_execute_script`(server_main / client)로 조작 → `[VRF]` 로그·Sync 값·스크린샷으로 판정. 버튼은 클라이언트에서 `SendEvent(ButtonClickEvent())`로 눌렀다.
> 코드·UI·데이터는 고치지 않았다. 끝에 저장 데이터를 되돌리고 메이커를 편집 모드로 두었다(8절).

---

## 1. 요약

| 항목 | 결과 |
|---|---|
| 빌드 로그 | 오류 0 (경고 29건은 모두 기존 `LWA-1111`·`LWA-1110` 형식, 2단계와 같은 수) |
| 콘솔 `LEA-` / `rror` | 0건 (테스트 세션 전체) |
| T1~T26 | **26개 모두 통과** (T26은 문구 함수 실측 + 지급 함수 실측, 실제 3번째 상자 수령은 계정 별 부족으로 못 함 — 3-3 참고) |
| 3-A(2단계 보완) | 확인한 항목 모두 통과 (5절) |
| 2단계 회귀 | 레벨업·전직·편성·탑 구성원 외형 정상 |
| 발견한 버그 | 1건(결과 팝업 다이아 아이콘 위치), 범위 밖 기존 문제 2건 |
| UI 관찰 | 6건 (6절) |

---

## 2. 시작 전 기록 (복원 기준)

- 롤스터 덤프: `v=1;dia=3000;book=50,20,10;cc=2;pt=0;awk=0,0;jc=Hero:3,Bishop:1,NightLord:0,Bowmaster:0,Viper:0;ch=Hero:1:1:0:1:0:0|Bishop:1:1:0:1:0:0|NightLord:1:1:0:1:0:0|Bowmaster:0:1:0:1:0:0|Viper:0:1:0:1:0:0;gift=1;gt=0;rf=19;rs=`
- 편성 `Hero,Bishop,NightLord`
- PlayerEngrave: `ClearedFloor=19`, `SelectedStage=1`, `Energy=30`, `AccountLevel=2`, `SectionData=2:1`, `StarData=1:7,2:1,…,18:1,19:7`(20층 없음)

---

## 3. 테스트 결과

### 3-1. 3-B 서버 (T1~T11)

| 번호 | 결과 | 근거 로그 |
|---|---|---|
| T1 표 | 통과 | `[VRF] roster tables jobs=5 outfits=143 gacha=5 banners=2` / `rates b=B001 sum=100.0000 J=5 … J.Hero=1.500 … J.Bowmaster=6.000 … C=1.000` / `rates b= sum=100.0000 J=5 … J.*=2.400` |
| T2 시각·배너 | 통과 | `utc=2026-11-21 17:42`(한국 11-22 02:42), `kst0900=63930902400000.0 dt=63930902400000`(같음), 자동 `B001`, `none` → `[]`, `B001` → `B001` |
| T3 확률(픽업) | 통과 | 아래 3-1-1 |
| T4 확률(상시) | 통과 | 아래 3-1-1 |
| T5 1회 | 통과 | `dia=900 pt=1 gt=1 books=0,1,0 msg=ok_draw res=draw:1:M` |
| T6 10회 부족·성공 | 통과 | `err_dia res=draw:10`(다이아 900 그대로) → `ok_draw draw:10:S,M,S,S,S,S,L,M,S,U.Bowmaster`, `dia=0 pt=11 gt=11 books=6,2,1` (자연 굴림에서 보장이 발동: `[VRF] gacha guarantee J.Bowmaster`) |
| T7 보장 | 통과 | `[VRF] gacha guarantee J.Bishop`, `draw:10:S,S,S,S,S,S,S,S,S,J.Bishop` |
| T8 해금 | 통과 | `draw:1:U.Viper`(카드 Viper:0, `ch=…Viper:1:1:0:1…`) → `draw:1:J.Viper`(Viper:1) |
| T9 5차 전환 | 통과 | `draw:1:X.Hero`, 히어로 카드 3 그대로, `pt=11` |
| T10 교환 | 통과 | `err_points` → `ok_exchange pt=0 res=exchange:Bowmaster:U` → `err_max` → `err_job` |
| T11 저장·로비 | 통과 | stop→play 후 `roster loaded`가 종료 직전 덤프(`dia=700;…;gt=24;rf=19;rs=`)와 같음. 탑 맵에서 클라이언트 `RequestGachaDraw(1)` → `[VRF] gacha reply err_lobby draw:1`, `RequestExchange("Viper")` → `err_lobby exchange:Viper` (실측) |

#### 3-1-1. T3·T4 확률 재측정 (구현 에이전트 주의점 2)

| 회 | 표본 | 배너 | S | M | L | J | C | pick | other | violations | cardsPerTen |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 20,000 | B001 | 44.1 | 30.6 | 11.9 | 12.4 | 1.1 | 6.0 | 1.6,1.5,1.6,1.7 | 0 | 1.54 |
| 2 | 20,000 | B001 | 44.1 | 30.1 | 12.1 | 12.6 | 1.1 | 6.1 | 1.5,1.8,1.7,1.5 | 0 | 1.57 |
| 3 | 20,000 | B001 | 45.4 | 29.9 | 11.7 | 12.0 | 1.0 | 6.2 | 1.5,1.4,1.4,1.5 | 0 | 1.55 |
| 4 | 50,000 | B001 | 45.1 | 29.8 | 11.9 | 12.2 | 1.0 | 6.0 | 1.5,1.5,1.6,1.6 | 0 | 1.52 |
| 5 | 20,000 | 상시 | 44.9 | 30.2 | 12.0 | 11.8 | 1.0 | - | 2.3,2.3,2.5,2.1,2.6 | 0 | 1.52 |
| 6 | 20,000 | 상시 | 45.0 | 30.7 | 11.5 | 11.8 | 1.0 | - | 2.3,2.5,2.5,2.2,2.4 | 0 | 1.6 |
| 7 | 20,000 | 상시 | 45.4 | 30.0 | 11.9 | 11.7 | 1.0 | - | 2.2,2.4,2.4,2.2,2.6 | 0 | 1.55 |

- 7회 모두 허용 범위 안. 구현 단계의 `L 13.2`는 재현되지 않았다(표본 오차로 판단).
- 난수 함수 자체도 확인: `_UtilLogic:RandomDouble()` 20만 회 평균 0.5001, 10구간 각 9.86~10.07% → 고르다.

### 3-2. 3-C 화면 (T12~T21)

| 번호 | 결과 | 근거 |
|---|---|---|
| T12 진입 | 통과 | 로비 `BtnGacha/Label` = `캐릭터 뽑기`(크기 32, BestFit 켜짐), 스크린샷에서 한 줄로 버튼 안에 들어감. 클릭 후 `CharGachaUI/Root` true, `[VRF] chargacha open banner=B001`, `/ui/GachaUI`의 `Dim`·`Panel` 모두 꺼짐 |
| T13 표시 | 통과 | `DiaText 1,000`, `기간 픽업` / `보우마스터 픽업` / `11월 22일 00:00 ~ 12월 6일 00:00 (한국 시간)` / `보우마스터 · 궁수`, `교환 포인트 12 / 30`, `Empty` 보임, `Desc` 3줄 계산값(6%, 1.5%). 스크린샷 정렬 정상 |
| T14 다이아 부족 차단 | 통과 | 다이아 50에서 `BtnOne`·`BtnTen` `Enable=false`, 회색 Disabled 이미지(스크린샷), 두 번 클릭 후 `GachaSeq 0 → 0`, `chargacha request` 로그 없음 |
| T15 1회만 가능 | 통과 | 다이아 100: `one=true ten=false` |
| T16 1회 해금 연출 | 통과 | `request draw 1` → `result ok_draw draw:1:U.Viper` → `reveal i=1 U.Viper`. `Cell1`만 (0,60), 아이콘 켜짐·글자 끔, `바이퍼\n전용 카드`, `해금!`, `Msg 바이퍼 해금!`, 다이아 0, 띠 색 `#D18014`. 요청 직후 뽑기·교환 버튼 모두 비활성 |
| T17 10회 연출 | 통과 | `Cell1` (-440,200)으로 복귀. 칸 1·6·7·9·10 소 회색 `(0.36,0.40,0.46)`, 칸 2·8 중 파랑, 칸 3 대 보라 `(0.45,0.28,0.70)`, 칸 4 `공용\n카드` 진홍 `(0.80,0.18,0.30)`, 칸 5 히어로 아이콘 + `+10 포인트`(히어로가 T9로 5차라 `X.Hero`). 연출 중(0.6초) 두 버튼 비활성, 끝난 뒤 다이아 0이라 비활성 유지, 교환 포인트 33이 되어 `카드 교환` 활성(스크린샷) |
| T18 확률 창 | 통과 | `Modal`·`Rate` 켜짐, `Dim` RaycastTarget true, 본문 `보우마스터 (픽업)  6%` / `히어로 · 비숍 · 나이트로드 · 바이퍼  각 1.5%`. 닫기 후 `Modal=false`. `none` 후 다시 열면 `모든 직업  각 2.4%`, `상시 뽑기` / `통합 뽑기` / `언제나 뽑을 수 있습니다`, `IconBox=false` |
| T19 교환 | 통과 | `Info` `보우마스터 전용 전직 카드 1장 — 교환 포인트 30 사용\n잠긴 직업은 이 카드로 바로 해금됩니다`, `BtnOk` 활성, `chargacha result ok_exchange exchange:Bowmaster:U`, 창 닫힘, `Msg 보우마스터 해금!`, `교환 포인트 0 / 30`, `BtnExchange` 비활성 |
| T20 교환 차단 | 통과 | 히어로(5차 Lv30) 선택 시 `BtnOk Enable=false`, 클릭 후 `GachaSeq 3 → 3`, `Info` 둘째 줄 `5차를 마친 직업은 교환할 수 없습니다` 빨강. Job1만 Selected 이미지(`1e130afe…`), 나머지 Normal. 스크린샷 확인 |
| T21 닫기·로비 이탈 | 통과 | `Root/Close` → Root false, `[VRF] chargacha close`. 다시 열면 결과 칸 비고 `Empty` 표시. **탑 이동 시 자동 닫힘 실측**: 열린 상태로 탑 이동 → `CharGachaUILogic.OnUpdate`에서 `[VRF] chargacha close`, Root false |

### 3-3. 3-D 다이아 (T22~T26)

| 번호 | 결과 | 근거 |
|---|---|---|
| T22 소급 첫 클리어 | 통과 | F=20(T25로 20이 됨). `DebugSetRewardMarks(0,"")` → `[VRF] dia first clear floors 1-20 +820 total=820`(1초 확인 `CheckDiamondRewards`), 기대 820 일치, 덤프 `rf=20`. 2.5초 더 기다려도 820 그대로 |
| T23 함수 중복 방지 | 통과 | `first=60 second=0 delta=60 expect=60` |
| T24 별 상자 | 통과 | `DebugCheckSection("1:7,2:3")` 첫 300, 두 번째 0, `delta=300`, 덤프 `rs=1` |
| T25 탑 결과 표시 | 통과 (실측) | 20층(미클리어)을 실제 판으로 진행. 시간 단축을 위해 판 시작 직후 `Stage = StagesPerFloor`(10)로 바꾸고 서버에서 `ApplyMonsterDamage(999999)`를 반복해 보스 처치 경로를 그대로 탔다. `[VRF] dia first clear floors 20-20 +60 total=60` 호출 위치가 `TowerRunManager.OnUpdate`(1초 확인보다 먼저), `RunDiamonds=60`, 결과 팝업 전투 보상 맨 앞 `다이아 x60`. **같은 층 재클리어**: `RunDiamonds=0`, 롤스터 다이아 `60 → 60`, 결과 팝업에 다이아 항목 없음(스킬북·깃털·주문서만) |
| T26 별 상자 안내 | 통과 (부분 실측) | 계정의 2구간 별이 14/30이라 실제 3번째 상자 수령은 불가. 대신 클라이언트에서 실제 함수 호출: `_RegionSelectLogic` `claimTier=3` + `보상 획득! …` → `… · 다이아 +300`, 실패 문구는 그대로, `claimTier=2`는 덧붙이지 않음, `_ResultPopupLogic`도 같은 결과. `PlayerEngrave.RequestClaimSection`의 성공 문구가 `보상 획득! 스킬북 +…`로 시작하는 것을 코드로 확인. 실제 지급은 T24(`GrantSectionDiamonds`)로 확인 |

- 소급 지급 중복(주의점 7): 재실행(stop→play) 2회 모두 `rf`·`rs`가 저장값 그대로 읽혀 다이아 변화 없음(`dia=700` 유지, 마지막 재실행 `dia=3000` 유지).

---

## 4. 버그

### B1. 탑 결과 팝업의 다이아 아이콘이 칸 위로 밀려 잘려 보임 (UI, 중간)

- 재현: 미클리어 층을 클리어 → 결과 팝업 `전투 보상` 첫 칸.
- 결과: 다이아 아이콘이 칸 안 가운데가 아니라 칸 위쪽 테두리 밖으로 올라가 아래 뾰족한 끝만 보인다. 같은 줄의 스킬북·깃털 아이콘은 정상.
- 확인값: `/ui/ResultPopup/Root/Result2/LootView/Loot0/Loot0Icon` — `ImageRUID 81462aeb…`, `Type Simple`, `PreserveSprite AspectOnly`, 크기 64×64, 위치·렌더러 오프셋 0 — 스킬북 칸(`Loot1Icon`)과 값이 같다.
- 원인 추정: 다이아 스프라이트의 피벗이 가운데가 아니라서 `AspectOnly`(스프라이트 피벗 반영)에서 위로 밀린다. 캐릭터 뽑기 화면 `DiaIcon`(PreserveSprite None)은 정상.
- 고칠 곳(제안): `Tower/ResultPopupLogic.mlua` 보상 칸 채우기(186행 부근)에서 다이아일 때만 아이콘 렌더러를 `PreserveSprite None`으로 두거나, `ui/ResultPopup`의 `Loot*Icon` 전체를 None으로(다른 아이콘 비율 확인 필요).

### 범위 밖 기존 문제 (3단계 변경과 무관, 기록만)

1. **결과 팝업 미션 상태가 첫 달성에도 `기존 달성`**: 20층 첫 클리어(이전 별 0)인데 `[VRF] result fill floor=20 clear=true prev=7 now=7` → 세 줄 모두 `기존 달성`, `신규 별` 배지 없음. 클라이언트가 `prev`를 읽을 때 이미 갱신된 `StarData`를 읽는 순서 문제로 보인다(`ResultPopupLogic.Fill`).
2. **메인 바이퍼로 탑 시작 시 HUD 1번이 `나이트로드`**: 2단계 보고서 관찰과 같음(4단계 보류 항목). 외형·너클은 정상.
- 참고: 테스트 중 서버에서 `ApplyMonsterDamage`를 0.4초마다 넣었을 때 `skill stone stage clear` 로그가 같은 초에 2~3번 찍혔다. 인위적 피해 주입 때문일 수 있어 버그로 보지 않았다.

---

## 5. 3-A(2단계 보완)·구현 주의점 확인

| 항목 | 결과 | 근거 |
|---|---|---|
| (a) 5차 이름 중복 | 통과 | 히어로 5차 `TierName`·`PreviewName` = `히어로 (5차)`. 나이트로드 4차 `PromoteTitle` = `전직 — 다음 차수: 나이트로드 (5차)` |
| (b) 잠긴 직업 미리보기 (주의점 3) | 통과 | 바이퍼 잠김: `Detail/PreviewDim=true`, 스크린샷에서 아바타가 어둡게 + 자물쇠. 같은 화면에서 서버로 해금(5차 Lv30) → `PreviewDim=false`, 밝기 정상 복귀(스크린샷). 구현은 문서의 `Avatar/Dim` 대신 `Detail/PreviewDim` + `SetAvatarPartColor`(0.45) 방식 |
| (c) 편성 띠 | 통과 | `편성  1 히어로 · 2 비숍 · 3 나이트로드`, 한 줄 |
| (d) 위쪽 띠 비침 | 통과 | 캐릭터 화면 스크린샷에서 로비 HUD(닉네임·레벨) 비침 없음 |
| (e) 바이퍼 5차 너클 (주의점 5) | 통과 | `OutfitFor("Viper",5).TwoHandedWeapon = 573d7ad7…`, 미리보기 `GetEquip` 같은 값, 미리보기·탑 스크린샷 모두 손에 붉은 너클 |
| (f) 히어로 stand2 (주의점 4) | 통과로 판단 | 히어로 5차 미리보기 스크린샷에서 두손검을 몸 앞에 세워 든 자세(2단계의 "뒤로 꽂힌 듯한" stand1 모습과 다름). `PreviewStand2Jobs="Hero"` |
| (g) 넘침 꼬리 | 통과 | 비숍 Lv1·책 50/20/10: `최대  Lv.1 → 7  ·  소 4 · 중 2 · 대 0  (사라지는 경험치: 10)` |
| 2단계 회귀 (주의점 8) | 통과 | `ok_level level:Bishop:1:7:7`, `ok_promote promote:Bishop:2:1:0`, `ok_party party:Viper,Bishop,NightLord`. 탑 구성원 외형: 히어로 5차 보라 아케인 세트·비숍 파란 모자·나이트로드 검정 털모자(20층 스크린샷), 메인 바이퍼 5차 세트(1층 스크린샷) |

---

## 6. UI 관찰 (버그 아님, 판단 참고)

1. **결과 칸 이름 글자가 작음**: `Cell{n}/Name`(184×56, 22, 2줄, BestFit)이 `바이퍼\n전용 카드`에서 약 16 크기로 줄어 칸 안에서 작게 보인다. 넘치지는 않는다. 칸 높이 여유(250)가 있어 Name 높이를 70 정도로 늘리면 22를 유지할 수 있다.
2. **상시 뽑기일 때 왼쪽 영역 가운데가 비어 보임**: `IconBox`·`JobName`이 꺼져 제목 아래 약 220px가 빈 공간. 동작은 문서대로.
3. **교환 창 5차 직업 안내가 두 줄 모두 빨강**: 문서 4-4는 둘째 줄을 부족색으로 적었는데 `Info` 전체가 빨강이라 첫 줄(`… 교환 포인트 30 사용`)도 빨강.
4. **잠긴 직업 미리보기 자물쇠가 얼굴 위에 겹침**: 덮개 가운데(0,40) 자물쇠가 아바타 얼굴과 겹친다. 의도된 배치라면 문제없음.
5. **다이아 표시 위치**: 뽑기 화면 위쪽 띠의 `DiaIcon`·`DiaText`가 오른쪽(닫기 버튼 왼쪽 아래)에 떠 있다. 문서 좌표 그대로이며 정렬은 정상.
6. **로비 `캐릭터 뽑기` 글자**: BestFit으로 다른 버튼(`성장`, `캐릭터`)보다 살짝 작다. 한 줄로 영역 안에 들어감.

---

## 7. 자료 조사 에이전트에게 넘길 질문

| 번호 | 질문 | 배경 |
|---|---|---|
| Q1 | 결과 팝업 다이아 아이콘 표시 방식: 다이아만 PreserveSprite None? 보상 아이콘 전체 None? | 버그 B1 |
| Q2 | 결과 칸 `Name` 높이(56)를 늘려 글자 22를 유지할지 | 관찰 1 |
| Q3 | 교환 창 5차 안내 색: 둘째 줄만 빨강으로 할지(두 텍스트로 나누거나 리치 텍스트) | 관찰 3 |
| Q4 | 상시 뽑기 때 비는 왼쪽 가운데 영역에 무엇을 둘지(예: 직업 5종 작은 아이콘 줄) | 관찰 2 |
| Q5 | 결과 팝업 `기존 달성` 순서 문제를 어느 단계에서 고칠지 | 범위 밖 1 (PlayerEngrave·ResultPopup 영역) |
| Q6 | 테스트용 계정 진행 데이터를 원래 19층 상태로 되돌릴지, 20층 진행을 그대로 둘지 | 8절 |

---

## 8. 마무리 상태와 테스트로 바뀐 계정 데이터

### 8-1. 복원한 것

- 롤스터: `DebugReset()` → `DebugSetBanner("")`(자동) → `DebugForceNext("")`(강제 결과 비움) → `DebugSetRewardMarks(20, "")` → `DoSetParty("Hero","Bishop","NightLord")` → `ok_party`, `RosterDirty=false`.
- 최종 덤프(재실행 후 `roster loaded`로 확인): `v=1;dia=3000;book=50,20,10;cc=2;pt=0;awk=0,0;jc=Hero:3,Bishop:1,NightLord:0,Bowmaster:0,Viper:0;ch=Hero:1:1:0:1:0:0|Bishop:1:1:0:1:0:0|NightLord:1:1:0:1:0:0|Bowmaster:0:1:0:1:0:0|Viper:0:1:0:1:0:0;gift=1;gt=0;rf=20;rs=`
- 시작 전 덤프와 다른 값은 **`rf=19 → 20` 하나**. 이유: T25로 `ClearedFloor`가 20이 되었다. `rf=19`로 두면 다음 접속 1초 안에 20층 몫 60이 자동 지급되어 결국 `rf=20`·다이아 3,060이 된다. 실제 진행(20층)과 맞추려고 20으로 두었다.
- `PlayerEngrave.SelectedStage` 1로 되돌림. 입장 에너지 30 그대로(탑 이동을 서버 텔레포트로 해서 소모 없음).
- 메이커: 편집 모드(Lobby).

### 8-2. 되돌리지 않은 것 (탑 실측 T25로 바뀜)

| 항목 | 원래 | 지금 |
|---|---|---|
| `PlayerEngrave.ClearedFloor` | 19 | 20 |
| `StarData` | `…,19:7` | `…,19:7,20:7` (2구간 별 11 → 14) |
| 계정 레벨 | 2 | 3 (계정 EXP +173 × 2판) |
| 스킬북·깃털·주문서(판 보상) | (시작값 기록 못 함) | 스킬북 +250(100+150), 깃털 +122(1+49+72), 주문서 +2 (실패 1판 + 클리어 2판 결과 팝업 기준) |
| `SectionData` | `2:1` | `2:1` (변화 없음) |

- 20층 진행을 되돌리려면 server_main에서(PlayerEngrave 코드 수정 없이 데이터만):
  ```lua
  local p = nil
  for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
  local eng = p:GetComponent("script.PlayerEngrave")
  local r = p:GetComponent("script.PlayerRoster")
  eng.ClearedFloor = 19
  eng:MarkDirty()
  eng:Flush()
  eng._T.stars[20] = nil
  eng:SyncStars()
  eng:SaveStars()
  r:DebugSetRewardMarks(19, "")
  ```
  (`_T.stars`를 밖에서 고치는 방식은 실측하지 않았다. 계정 EXP·스킬북·깃털·주문서는 시작값을 기록하지 못해 정확히 되돌릴 수 없다.)

### 8-3. 변경 파일 (`git status` 기준)

- 이 테스트에서 고친 파일 없음(이 보고서만 새로 씀).
- `Engrave/PlayerEngrave.mlua` diff는 유물 연출 세션의 `ReceiveRelicDraw` 추가(8줄)뿐, 3단계 변경 없음.
