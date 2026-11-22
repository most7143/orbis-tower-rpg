# 2단계 테스트 보고서: 로비 캐릭터 화면 + 차수 외형

> 작성: 테스트 에이전트, 2026-11-22
> 기준: `Phase-2-Character-UI.md` 6절(T1~T22)과 완료 조건, `Phase-1-Character-Core.md` 끝 "1단계 보완" 완료 조건 1~3
> 방법: 메이커 새로고침 → 빌드 로그 확인 → 플레이 → `maker_execute_script`(server_main / client)로 조작 → `[VRF]` 로그·Sync 값·스크린샷으로 판정. 버튼은 클라이언트에서 `SendEvent(ButtonClickEvent())`로 눌렀다.
> 코드·UI·데이터는 고치지 않았다. 끝에 저장 데이터를 첫 지급 상태 + 편성 히어로·비숍·나이트로드로 되돌리고 메이커를 편집 모드로 두었다.

---

## 1. 요약

| 항목 | 결과 |
|---|---|
| 빌드 로그 | 오류 0 (경고 29건은 모두 기존 `LWA-1111`·`LWA-1110` 형식 경고이고 Character 폴더 파일에서 나온 것은 없음) |
| 콘솔 `LEA-` | 0건 (두 번째 플레이 세션 전체 검색). 첫 세션은 중간 로그 정리 전에 `rror` 검색 0건 |
| 1단계 보완(2-A) 완료 조건 1~3 | **3개 모두 통과** |
| T1~T22 | **20개 통과, 2개 조건부 통과(T20, T21)**. 막히는 실패 없음 |
| 발견한 버그 | 3건 (5차 이름 "(5차)" 중복 표시, 잠긴 직업 미리보기 어둡게 안 됨, 편성 띠 글자 줄바꿈) |
| UI·외형 관찰 | 6건 (아래 4절) |

---

## 2. 1단계 보완(2-A) 확인

| 번호 | 절차 | 결과 | 근거 로그 |
|---|---|---|---|
| 1 | `r:DebugSetCurrency(0,0,0,0,0)` | 통과 | `[VRF] roster debug currency dia=0 books=0,0,0 common=0` |
| 2 | 히어로 Lv1, 서 소 1권만(`DebugSetCurrency(3000,1,0,0,2)`) → `r:DoLevelTo("Hero", 7)` | 통과 | `[VRF] roster books job=Hero used=1,0,0 lv 1->1 exp=100` / `[VRF] roster reply ok_level level:Hero:1:1:7` |
| 3 | 탑 맵(OrbisTower)에서 클라이언트가 `RequestPromote("Hero", false)` | 통과 | `[VRF] roster reply err_lobby promote:Hero` (같이 보낸 `RequestLevelTo` → `err_lobby level:Hero`, `RequestSetParty` → `err_lobby party:Hero,Bishop,NightLord` 도 확인) |

---

## 3. 테스트 케이스 결과

| 번호 | 결과 | 근거 |
|---|---|---|
| T1 진입 | 통과 | 로비 버튼 글자 `캐릭터`, 클릭 후 `CharacterUI/Root` enable true, `[VRF] charui open` |
| T2 목록 | 통과 | `검사 / 매지션 / 로그`, `전사 · Lv.1 · 1차`, 카드 `3장 / 1장 / 0장`; Job4·5 덮개 켜짐 + `궁수 · 잠김`, `해적 · 잠김`; PartyMark 1·2·3 |
| T3 상세 | 통과 | `검사 (1차)`, `Lv.1  /  관문 Lv.7`, `공격력 10 / 체력 120 / 방어력 1`, 서 `50권 / 20권 / 10권` |
| T4 미리보기 | 통과 | `1레벨업  Lv.1 → 2  ·  소 2 · 중 0 · 대 0`, `최대  Lv.1 → 7  ·  소 4 · 중 2 · 대 0  (넘치는 경험치 10은 사라집니다)` — 아래 5-1 판단 참고 |
| T5 1레벨업 | 통과 | `ok_level level:Hero:1:2:2`, Msg `Lv.1 → Lv.2 달성!`, 소 48권 |
| T6 최대 | 통과 | `ok_level level:Hero:2:7:7`, 두 레벨업 버튼 비활성, `PreviewOne` `관문 도달 — 전직이 필요합니다`, `ExpText` `관문 도달 — 전직하면 계속 성장합니다` |
| T7 비활성 차단 | 통과 | 비활성 `BtnLv1`·`BtnLvMax` 클릭 후 `RosterSeq` 3 → 3 (요청 없음) |
| T8 전직 | 통과 | `ok_promote promote:Hero:2:1:0`, `파이터 (2차)`, 카드 2장, `공격력 24 / 체력 211 / 방어력 9`, `[VRF] outfit job=Hero tier=2 ... n=6` (1초 안) |
| T9 공용 대체 | 통과 | `BtnPromote` 비활성, `BtnPromoteCommon` 활성 `공용 카드로 전직 (1장)` |
| T10 | 통과 | `ok_promote promote:Bishop:3:1:1`, Msg `프리스트로 전직했습니다! (공용 카드 1장 사용)`, 공용 2 → 1, Currency 갱신 |
| T11 둘 다 부족 | 통과 | 두 전직 버튼 비활성, `PromoteInfo` 색 `RGBA(0.78,0.20,0.20)` |
| T12 해금 | 통과 | `ok_unlock unlock:Viper`, 덮개 꺼짐, `해적 / 해적 · Lv.1 · 1차`, 공용 0, `BtnUnlock` 숨김, Msg `바이퍼 해금!` |
| T13 잠긴 직업 편성 차단 | 통과 | Job4 선택 시 `Party1~3` 모두 비활성, 클릭 3번 후 `RosterSeq` 6 → 6 |
| T14 편성 | 통과 | `ok_party party:Viper,Bishop,NightLord`, `MainJob=Viper`, Job5 PartyMark `1`, Job1 PartyMark 숨김, `PartyText` 갱신, `[VRF] outfit job=Viper tier=1 ... n=4` |
| T15 책 없음 | 통과 | 두 레벨업 버튼 비활성, `PreviewOne/Max` `경험의 서가 없습니다`, `Book*/Count` 부족색 |
| T16 스킬 탭 편성 | 통과 | `[VRF] roster reply err_party party:Bowmaster,Bishop,NightLord`, `MainJob` 변화 없음 (`_HeroSkillUILogic:SelectJob("Bowmaster")` 후 `OnFormation(1)` 호출로 확인) |
| T17 외형 표 | 통과 | `Cap=b72cc1ef… Cape=54c1be1e… Glove=dfd4da84… Longcoat=e93e89b8… Shoes=ce9c38b1… TwoHandedWeapon=f0c8f180… family=T`, `[VRF] roster tables jobs=5 outfits=143` |
| T18 로비 외형 | 통과 (로그) | `[VRF] outfit job=Hero tier=5 ... n=6`, 플레이어 Cap `927bc02e…`, Longcoat `f0d982e5…`, TwoHanded `e1ee66e2…`. 로비 화면에는 플레이어 캐릭터가 보이지 않아 스크린샷 대신 캐릭터 화면 미리보기로 아케인셰이드 나이트 세트를 확인 |
| T19 탑 구성원 | 통과 | 히어로 5차·비숍 3차·나이트로드 1차로 판 시작 → `[VRF] outfit job=Hero tier=5`, `job=Bishop tier=3 target=Companion n=6`, `job=NightLord tier=1 target=Companion2 n=4`. 스크린샷에서 세 명이 각각 다른 세트(보라 아케인 / 파란 모자·로브 / 검정 털모자) |
| T20 스킬 동작 무기 | 조건부 통과 | 아래 3-1 |
| T21 미리보기 | 조건부 통과 | 직업·차수마다 옷이 바뀜(스크린샷 확인). 다만 잠긴 직업 어둡게 표시가 안 됨 → 버그 B2 |
| T22 재접속 | 통과 | stop → play 후 `roster loaded`가 종료 직전 `DebugDump`와 같은 문자열, 편성 `Viper,NightLord,Bowmaster` 유지, 로비 외형 Cap `db33b98e…`(아케인셰이드 파이렛햇)·TwoHanded `a5602ea4…`(바이퍼 5차 세트), 캐릭터 화면 목록도 같은 값 |

추가 확인:
- 연타 방지: `BtnPromoteCommon` 클릭 직후(같은 프레임) 전직·레벨업·편성 버튼이 모두 비활성 → 결과 도착 후 다시 계산. 통과.
- 5차 Lv30: `Lv.30 (최고)`, `ExpText` `최고 레벨·최고 차수입니다`, `PreviewOne` `최고 레벨입니다`, `PreviewMax` 빈칸, `PromoteTitle` `최고 차수 달성`, 전직 버튼 숨김, 경험치 막대 840 꽉 참. 통과.
- 닫기 X: 클릭 후 Root enable false. 로비를 떠나면(탑 이동) `[VRF] charui close`로 자동 닫힘. 통과.
- 탑에서 로비로 돌아오면 1초 안에 로비 외형을 다시 입힘(`RefreshLobbyOutfit`). 통과.

### 3-1. T20 상세

- 실제 전투 중 스킬 프레임은 0.3~0.6초라 스크린샷으로 잡기 어려워, 클라이언트에서 `TowerPlayer:ShowActionFrame(동작, 프레임)`(스킬 재생이 쓰는 같은 함수)을 반복 호출해 자세를 고정한 뒤 찍었다. 동작 이름은 `SkillActionTable.csv`에서 직업별로 실제 쓰는 것을 골랐다.
- 1회차(메인 히어로 5차, 비숍 3차, 나이트로드 1차): 히어로 `swingT1` 2프레임 두손검 보임, 나이트로드 `swingO1` 2프레임 손에 아대 보임, 비숍 `swingO1` 완드 보임.
- 2회차(메인 바이퍼 5차, 나이트로드 4차, 보우마스터 2차): 바이퍼 `stabO1` 무기 앞으로 뻗은 모습 보임, 보우마스터 `shoot1` 2프레임 활 보임, 나이트로드 `swingO2` 손 쪽 무기 확인(작아서 식별이 어려움).
- 결론: 무기 사라짐은 보지 못했다. `WeaponFamily`는 메인·구성원 모두 표대로(히어로 T, 비숍·나이트로드·바이퍼 O, 보우마스터 T) 들어갔다. 표 값을 바꿀 필요는 없어 보인다.
- 한계: 고정 자세 확인이며 실제 자동 전투 중 프레임 전환은 눈으로 따라가지 못했다. 사람이 판 1회를 보며 최종 확인하는 것을 권한다.
- 관찰(2단계 범위 밖): 메인을 바이퍼로 판을 시작하면 HUD 1번 칸이 `1 나이트로드 Lv.1`, 자원 `표식 0/100`으로 나오고 첫 스킬 칸 아이콘이 비어 있다. 플레이어 엔티티의 `JobCombat`이 시작부터 `job=NightLord`로 떠 있어서(`[VRF] JobCombat begin job=NightLord`) 생기는 기존 탑 구조 문제로 보인다. 외형은 바이퍼 세트로 정상. 4단계(판 전달 교체)에서 같이 확인할 것.

---

## 4. 발견한 버그·이상

### B1. 5차 이름에 "(5차)"가 두 번 붙음 (실패, 중간)

- 재현: 서버 `r:DebugSetChar("Hero",1,25,5)` → 캐릭터 화면에서 히어로 선택. 또는 4차 직업 선택.
- 결과: `TierName`·`PreviewName` = `히어로 (5차) (5차)`, 4차의 `PromoteTitle` = `전직 — 다음 차수: 나이트로드 (5차) (5차)`.
- 원인: `CharJobTable.csv`의 `Name5` 값이 이미 `히어로 (5차)` 형식인데 `Character/CharacterUILogic.mlua`의 `RefreshDetail`이 `tierName .. " (" .. tier .. "차)"`로, `PromoteTitle`도 `TierName(job, nextTier) .. " (" .. nextTier .. "차)"`로 차수를 또 붙인다.
- 고칠 곳(제안): 표의 `Name5`를 그대로 두고 UI에서 "이름에 '차)'가 이미 있으면 붙이지 않기", 또는 `Name5`를 별도 이름(자료 조사 결정 필요, 6절 Q1)으로 바꾸기. 목록 카드 이름(`히어로 (5차)`)·편성 띠는 한 번만 붙어 정상.

### B2. 잠긴 직업 미리보기 아바타가 어두워지지 않음 (조건부, 낮음) — 구현 에이전트 주의점 4

- 재현: 잠긴 직업(바이퍼·보우마스터) 선택.
- 결과: `AvatarGUIRendererComponent.Color`는 `RGBA(0.45,0.45,0.45,1)`로 들어가 있으나 화면은 그대로 밝다. 색을 1 → 0.45로 다시 넣어도 화면 픽셀 평균이 바뀌지 않았다(스크린샷 비교).
- 원인 추정: `AvatarGUIRendererComponent.Color`가 실제 부위 렌더러에 전달되지 않는 엔진 동작으로 보인다. 같은 컴포넌트의 `SetAvatarPartColor(category, r,g,b,a)`는 실측에서 **몸(피부)·얼굴은 회색으로 바뀌었고 모자는 바뀌지 않았다**(입은 아이템 부위까지 먹는지는 불확실).
- 제안: (1) 미리보기 칸 위에 반투명 어두운 덮개 스프라이트(목록 카드 `Lock`과 같은 방식, Raycast false)를 잠긴 직업일 때만 켜기 — 가장 확실. (2) 또는 `SetAvatarPartColor`를 9개 부위+몸·머리·얼굴 전부에 적용하고 해금 직업에서 1로 되돌리기(옷 부위 적용 여부 추가 확인 필요). 파일: `Character/CharacterUILogic.mlua` `ShowPreview`, 덮개 방식이면 `ui/CharacterUI`에 엔티티 추가.
- 테스트 중 넣은 부위 색은 같은 세션에서 1로 되돌렸다.

### B3. 편성 띠 글자가 길면 두 줄로 넘침 (UI, 낮음)

- 재현: 편성을 5차 직업 3개(예: 나이트로드 5차·보우마스터 5차·바이퍼 5차)로 둔다.
- 결과: `PartyText` = `편성  1 나이트로드 (5차) · 2 보우마스터 (5차) · 3 바이퍼 (5차)`가 600px 칸에서 두 줄로 줄바꿈되어 둘째 줄이 파란 구분선에 붙는다.
- 제안: `PartyText`에 `BestFit`(최소 16) 켜기 또는 차수 표기를 빼고 이름만 쓰기. B1을 고치는 방식과 함께 정하면 된다. 파일: `ui/CharacterUI` `Root/PartyText`.

### 관찰 (버그 아님, 판단 참고)

1. **두손 무기 미리보기 자세**: 미리보기 아바타는 항상 한손 대기 자세(stand1)라 두손검(파이터 그륜힐, 히어로 5차 투핸드소드)이 비스듬히 뒤로 꽂힌 듯 보인다. 탑에서는 정상 자세. 크게 어색하지는 않지만 미리보기에 `stand2`를 주는 방법(가이드의 `StateComponent`+`AvatarStateAnimationComponent` 추가)을 검토할 수 있다.
2. **아대·너클 미리보기**: 나이트로드 1차 가니어, 4차 리스크홀더는 대기 자세에서 거의 보이지 않는다(아이템이 작음). 공격 자세에서는 보인다. 문제 아님.
3. **바이퍼 5차 무기 이름 확인**: `CharOutfitTable`의 바이퍼 5차 `TwoHandedWeapon`이 `아케인셰이드 클로`이다. 공격 자세에서 칼날처럼 앞으로 뻗어 보여 아대(클로)인지 너클인지 육안으로 애매했다. 메이플 한국어 이름 기준으로 너클이 맞는지 자료 조사 확인을 권한다(6절 Q3).
4. **잠긴 직업 상세의 레벨·경험치**: 잠긴 직업도 `Lv.1  /  관문 Lv.7`, `경험치 0 / 120`이 그대로 보인다. 문서에 정해진 문구가 없어 버그로 보지 않았다.
5. **배경 반투명**: 캐릭터 화면 위쪽 띠 뒤로 로비 HUD(닉네임·레벨 2·에너지)가 희미하게 비친다. 성장창과 같은 `panel_outer` 반투명 값이라 의도된 것으로 보이나, 제목 `캐릭터` 옆에 `감자`·`레벨 2`가 겹쳐 보인다.
6. **스킬 탭 편성 실패 안내 없음**: T16처럼 스킬 탭에서 잠긴 직업 편성이 막혀도 스킬 탭 자체에는 문구가 뜨지 않는다(캐릭터 화면이 열려 있을 때만 그 화면 `Msg`에 뜸). 6단계 정리 전까지는 큰 문제 아님.

---

## 5. 구현 에이전트 주의점 판단

1. **T4 `(넘치는 경험치 10은 사라집니다)`** — **맞는 표시.** Q11 규칙은 "책 경험치 합 - plan.gain > 0이면 붙인다"이고, 관문 Lv7까지 필요 1,390에 책 1,400(중 2 + 소 4)을 쓰므로 10이 관문에서 버려진다. 6절 T4 기대 문구에 이 꼬리가 빠진 것은 문서 쪽 누락이다. 1레벨업 미리보기는 넘친 경험치가 다음 레벨로 이어져 `gain`에 포함되므로 꼬리가 붙지 않는 것도 맞다. 작은 문제: 조사가 `은`으로 고정이라 숫자 끝소리가 받침 없는 값(예: 2·4·5·9로 끝나는 수)이면 `2은`처럼 어색해진다(`2는`이 맞음). 지금 나온 값(10·50)은 정상.
2. **잠김/관문/최고 안내는 `PreviewOne`에만, `PreviewMax` 빈칸** — 2-4 표는 "PreviewOne/Max = 문구"라 두 줄 모두에 같은 문구로 읽을 수도 있으나, 같은 문장을 두 번 쓰는 것보다 한 줄이 깔끔해 **수용 가능**. 스크린샷에서도 어색하지 않다.
3. **잠긴 직업 `PromoteNote` 해금 안내, 5차 30레벨 `ExpText`** — 둘 다 문서에 없는 추가지만 상황 설명으로 적절. `PromoteNote` `공용 전직 카드 1장으로 바로 해금할 수도 있습니다`, `ExpText` `최고 레벨·최고 차수입니다` 확인.
4. **잠긴 직업 아바타 어둡게 안 됨** — 확인됨. B2 참고(엔진 동작 추정, 덮개 방식 권장).
5. **`ui/LobbyUI` diff** — git 원본과 현재 파일을 JSON으로 읽어 수치 비교(허용 오차 1e-4)한 결과 **다른 값은 `BtnAchieve/Label`의 Text `업적` → `캐릭터` 하나뿐**. diff가 큰 것은 숫자 표기 형식 차이. 로비 스크린샷도 이전 배치 그대로(플레이·성장·캐릭터·스킬 뽑기 버튼, 좌상단 HUD) 정상.
6. **T19·T20·T22** — 위 표·3-1 참고. 모두 통과(T20은 고정 자세 확인).
7. **2-A 완료 조건 3** — 통과(`err_lobby promote:Hero`).

---

## 6. 자료 조사 에이전트에게 넘길 질문·결정

| 번호 | 질문 | 배경 |
|---|---|---|
| Q1 | 5차 이름 표기를 어떻게 할까? (a) 표 `Name5`는 `히어로 (5차)` 그대로, UI가 차수를 덧붙이지 않게 (b) `Name5`를 고유 이름으로 바꾸고 UI는 `(5차)`를 붙임 | B1. 목록·편성 띠·결과 문구(`… (5차)로 전직했습니다!`)까지 한 규칙으로 정해야 함 |
| Q2 | 잠긴 직업 미리보기 표현: 어두운 덮개 / 부위 색 / 실루엣 중 무엇으로? | B2. `AvatarGUIRendererComponent.Color`가 화면에 반영되지 않음 |
| Q3 | 바이퍼 5차 무기 `아케인셰이드 클로`(a5602ea4…)가 너클이 맞는지 재확인 | 관찰 3 |
| Q4 | 미리보기 아바타에 두손 무기 대기 자세(stand2)를 줄지 | 관찰 1 |
| Q5 | 6절 T4 기대 문구에 `(넘치는 경험치 10은 사라집니다)`를 추가해 문서 정정 | 5-1 |
| Q6 | 편성 띠 표기(차수 포함 여부, BestFit) | B3 |
| Q7 | 메인 바이퍼일 때 탑 HUD·전투가 나이트로드로 동작하는 문제를 4단계 요구사항에 넣을지 | 3-1 관찰 |
| Q8 | `Global/DefaultPlayer.model`이 작업 트리에서 3줄 바뀌어 있는데 2단계 허용 파일 목록(1절)에 없다. 어느 작업의 변경인지 확인 필요 (이번 테스트에서는 내용 확인 권한이 막혀 열어 보지 못함) | 완료 조건 "허용한 파일만 변경" |

---

## 7. 변경 파일 확인 (`git status` 기준)

- 2단계 문서 허용: `ui/LobbyUI.ui`(글자만), `Lobby/LobbyUILogic.mlua`(한 줄+주석), `Skill/HeroSkillUILogic.mlua`(한 줄), `Tower/TowerRunManager.mlua`(ApplyCostume·CopyCostumeToCompanion), 신규 `Character/`, `Data/Char*.csv/.userdataset`, `ui/CharacterUI.ui`.
- 다른 세션(유물 뽑기 연출) 파일: `Engrave/PlayerEngrave.mlua`, `Lobby/RelicUILogic.mlua`, `ui/RelicUI.ui`, `Docs/UI-Design-System.md` — 2단계와 무관한 변경(`ReceiveRelicDraw` 등)으로 확인.
- 확인 필요: `Global/DefaultPlayer.model` (Q8).

## 8. 마무리 상태

- 저장 데이터: `DebugReset()` + `DoSetParty("Hero","Bishop","NightLord")` 후 저장 완료(`RosterDirty=false`). 최종 덤프 `dia=3000;book=50,20,10;cc=2;jc=Hero:3,Bishop:1,...;ch=Hero:1:1:0:1|Bishop:1:1:0:1|NightLord:1:1:0:1|Bowmaster:0:...|Viper:0:...`, 편성 `Hero,Bishop,NightLord`.
- 메이커: 편집 모드(Lobby).
