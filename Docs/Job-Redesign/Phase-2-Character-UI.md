# 2단계 요구사항: 로비 캐릭터 화면 + 차수 외형

> 작성: 자료 조사 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(최우선) → `01-Agent-Rules.md` → `Docs/UI-Design-System.md` → 이 문서
> 선행: 1단계(`Phase-1-Character-Core.md`, `Character/PlayerRoster.mlua`) 완료 + 같은 문서 끝 "1단계 보완" 작업(이 문서의 2-A)
> 이 문서만 보고 구현할 수 있도록 엔티티·좌표·RUID·흐름을 모두 적는다. 문서에 없는 판단이 필요하면 마스터 스펙 의도대로 정하고 보고에 적는다.

---

## 0. 결정 사항 요약

| 번호 | 결정 | 근거 |
|---|---|---|
| E1 | **진입 위치 = 로비의 `BtnAchieve`(현재 "업적", 누르면 "준비 중" 안내만 뜸)를 "캐릭터" 버튼으로 바꾼다.** 엔티티·UUID·위치(0,-376)·크기(288×96)·아이콘은 그대로, 글자만 `캐릭터`로, 클릭 연결만 `_CharacterUILogic:Open()`으로 바꾼다 | 가장 덜 침습적: ① 업적은 기능이 없는 자리 ② `BtnGacha`(스킬 뽑기)는 6단계까지 살아 있고 3단계에서 "캐릭터 뽑기"로 바뀔 자리라 건드리지 않는다 ③ 성장창 탭으로 넣으면 `GrowthUILogic`의 탭 수·탭 폭(368×5)·`CloseContent` 분기를 모두 고쳐야 하고, 캐릭터 화면이 커서 탭 본문(성장창 내용 영역) 안에 넣기 어렵다 ④ 로비 메뉴 버튼 수·배치가 그대로라 메인화면 규칙(보조 메뉴 3개 같은 크기)을 지킨다 |
| E2 | 캐릭터 화면은 **새 파일 `ui/CharacterUI` + 새 스크립트 `RootDesk/MyDesk/Character/CharacterUILogic.mlua`(@Logic, ClientOnly)**. 골조(외곽·밝은 본문·제목·공통 X·파란 구분선)는 성장창 `ui/GrowthUI`와 같은 리소스·좌표 | 기존 화면 수정 최소화, UI-Design-System 준수 |
| E3 | 탭 없음. 한 화면에 **왼쪽 직업 목록 / 오른쪽 상세(미리보기·능력치·레벨업·전직)**, 위쪽 띠에 **편성 3칸 + 재화** | 마스터 스펙 7절 "최대한 단순하게". 기능이 4개뿐이라 탭 전환이 오히려 클릭 수를 늘린다 |
| E4 | 레벨업 버튼은 **`1레벨업` / `최대 레벨업(관문까지)` 2개만**. 둘 다 `PlanBooksFor`로 미리보기를 보여 주고 `RequestLevelTo`를 보낸다. 책 개수 직접 선택 UI는 만들지 않는다 (`DoUseBooks`는 UI에서 안 씀) | 1단계 보완 Q3. 스타레일·명일방주의 "자동 채우기 + 목표 미리보기"가 표준 |
| E5 | 전직 버튼 2개: **`전직`(전용 카드만) / `공용 카드로 전직`(부족분만 공용)**. 공용 버튼은 전용 카드가 부족하고 공용으로 메울 수 있을 때만 활성. 별도 확인 팝업 없이 "따로 누르는 버튼"이 확인 역할 | 로드맵 1-3 "대체 사용은 버튼을 따로 눌러 확인" |
| E6 | **편성은 캐릭터 화면이 정본**. 위쪽 띠의 `1번 / 2번 / 3번` 버튼 = "지금 선택한 직업을 그 자리에 넣기(이미 다른 자리에 있으면 서로 바꿈)" — 기존 스킬 탭 편성과 같은 조작. 요청은 `PlayerRoster:RequestSetParty` | 기존 편성 UX 유지, 해금 검사가 있는 새 함수 사용 |
| E7 | 기존 성장창 5번 "스킬" 탭의 편성 버튼(`HeroSkillUILogic.OnFormation`)은 **요청 한 줄만 `RequestSetParty`로 바꾼다**(잠긴 직업 편성 차단). 버튼·배치는 6단계 정리 때까지 그대로 | 두 화면이 서로 다른 규칙으로 편성을 바꾸지 않게. 최소 수정 |
| E8 | 차수 외형 표 **`CharOutfitTable.csv`** (열 `Job,Tier,Slot,RUID,#Name`). 25세트·143행, 모든 RUID는 리소스 API 응답에서 가져와 일괄 재검증(119개 고유 RUID 모두 `avataritem` 확인) | 마스터 스펙 4절 |
| E9 | 세트 레벨대: **1차 = 10제급 / 2차 = 30제급 / 3차 = 70제급(망토 시작) / 4차 = 150제 루타비스(하이네스·이글아이·트릭스터) / 5차 = 200제 아케인셰이드**. 낮은 차수는 망토·장갑이 없다 | 사용자 예시(10/30/70/120/200제)에 맞추되, MSW 검색에서 직업별로 확실히 구분되는 4차 세트가 150제 루타비스뿐이라 4차는 150제를 쓴다 (120제 타임리스는 직업 구분 근거 부족) |
| E10 | 외형 적용 함수는 **`PlayerRoster`에 추가**(`OutfitFor`, `ApplyOutfit`, 로비 자동 적용). 탑은 `TowerRunManager.ApplyCostume / CopyCostumeToCompanion` 본문을 이 함수 호출로 바꾼다. 장비 단계 외형·`JobWeapons`는 쓰지 않는다 | 마스터 스펙 4절 "장비 외형 사용 안 함" |
| E11 | 무기 동작 계열(`TowerPlayer.WeaponFamily`)은 **`CharJobTable`에 새 열 `WeaponFamily`** 로 둔다: 히어로 T, 비숍 O, 나이트로드 O, 보우마스터 T, 바이퍼 O | 현재 탑에서 실제로 동작 중인 값(`TowerRunManager.JobWeapons`: 나이트로드·바이퍼 O)을 유지. 2-C 테스트에서 무기가 사라지면 T로 바꾼다 (6절 T20) |
| E12 | 캐릭터 화면 미리보기 아바타 = **UI 엔티티에 `AvatarGUIRendererComponent` + `CostumeManagerComponent`**(MSW 공식 가이드 "Representing Avatars in the UI" 방식). 클라이언트에서 `SetEquip`으로 선택 직업·차수 세트를 입힌다. 표시가 안 되면 생략하고 그 칸에 차수 이름 큰 글자만 둔다 | 문서 검색으로 방식 확인. 이 프로젝트에서는 아직 사용 사례가 없어 메이커 실측 필요 |

---

## 1. 작업 단위

| 단위 | 내용 | 바꾸는 파일 | 완료 조건 |
|---|---|---|---|
| **2-A** | 1단계 보완: `DebugSetCurrency`, `err_lobby` 동작 이름, `DoLevelTo` 결과에 목표 레벨 | `Character/PlayerRoster.mlua` | `Phase-1-Character-Core.md` 끝 "1단계 보완"의 완료 조건 1~3 |
| **2-B1** | `ui/CharacterUI` 골조와 정적 엔티티 전부(2절 표), 버튼 상태 이미지 패치 | `ui/CharacterUI` (신규) | UI 린트 오류 0, 메이커 스크린샷으로 정렬·글자 영역 확인 |
| **2-B2** | `CharacterUILogic.mlua`: 열기/닫기, 목록·상세 갱신, 레벨업·전직·해금·편성 요청, 결과 문구 | `Character/CharacterUILogic.mlua` (신규) | 6절 T1~T15 |
| **2-B3** | 로비 버튼 교체 + 스킬 탭 편성 요청 교체 | `ui/LobbyUI`(Label 글자만), `Lobby/LobbyUILogic.mlua`(한 줄), `Skill/HeroSkillUILogic.mlua`(한 줄) | T1, T16 |
| **2-C1** | 외형 표 `CharOutfitTable.csv` + `.userdataset`, `CharJobTable`에 `WeaponFamily` 열, `PlayerRoster`에 `OutfitFor` / `ApplyOutfit` / 로비 자동 적용 | `Data/CharOutfitTable.*`(신규), `Data/CharJobTable.csv`, `Character/PlayerRoster.mlua` | T17, T18 |
| **2-C2** | 탑 외형 연결: `TowerRunManager.ApplyCostume`, `CopyCostumeToCompanion` 본문 교체 | `Tower/TowerRunManager.mlua` | T19, T20 |
| **2-C3** | 캐릭터 화면 미리보기 아바타 | `ui/CharacterUI`, `CharacterUILogic.mlua` | T21 (안 되면 생략하고 보고) |

순서: 2-A → 2-C1 → 2-B1 → 2-B2 → 2-B3 → 2-C2 → 2-C3. 각 단위 끝에 메이커 새로고침·빌드 로그 확인, 마지막에 6절 전체를 한 번의 플레이에서 확인한다.

**건드리지 않는 파일**(다른 세션 작업 중): `Engrave/PlayerEngrave.mlua`, `Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md`. 이 단계에서는 이 파일들을 고칠 필요가 없다. `ui/GrowthUI`, `Lobby/GrowthUILogic.mlua`도 고치지 않는다(골조 값을 **읽어서 복사**만 한다).

---

## 2. 캐릭터 화면 `ui/CharacterUI`

### 2-1. 공통 규칙

- 1920×1080 기준. 표의 `pos`는 **부모 중심 기준 anchoredPosition**(앵커·피벗 0.5,0.5), 따로 적은 것만 예외. 크기 단위 px.
- 빌더: `.claude/skills/msw-ui-system/scripts/msw_ui_builder.cjs`, 경로는 node 안에서 `'ui/'+'CharacterUI'+'.'+'ui'`로 조립(셸 가드). 만들기 전 `Docs/UI-Design-System.md`, `builder-protocol.md`(핵심) + `builder-protocol-ui.md` §3를 읽는다.
- 그룹: `new UIBuilder("CharacterUI", 10, true)` (성장창과 같은 GroupOrder 10). `Root`는 **enable false**로 만든다(처음엔 닫힘).
- 모든 버튼: 빌더로 만든 뒤 `ButtonComponent`를 `Transition: 2`(SpriteSwap) + `ImageRUIDs`(Highlighted/Pressed/Selected/Disabled)로 패치(`01-Agent-Rules.md`). 배경 렌더러 `PreserveSprite = 0`(None), 패널·버튼 `Type = 1`(Sliced, 경계 12px), close는 `Type = 0`(Simple). 버튼 글자는 버튼 자체 `TextGUIRendererComponent`에 넣고 가로 Center·세로 Middle, Padding 0.
- 런타임 상태색은 `_UIButtonLogic:UseTheme(entity, role)`로 연결한다(역할 `button / card / action / close / primary`). 비활성은 `_UIButtonLogic:SetDisabled`, 선택은 `_UIButtonLogic:SetSelected`.
- 글자색: 일반 `Color(0.14,0.17,0.21)`, 보조 `Color(0.36,0.40,0.46)`, 유리한 변화·성공 `Color(0.20,0.48,0.12)`, 부족·오류 `Color(0.78,0.20,0.20)`, 어두운 면 위 흰색.

### 2-2. 버튼 상태 이미지 (manifest `output/ui/growth_neutral/manifest.json`)

| 계열 | Normal | Hover | Pressed | Disabled | Selected | Selected Hover | Selected Pressed |
|---|---|---|---|---|---|---|---|
| card (직업 목록 카드) | `a88cfe62e8e64c39bf14d682a77ef4a0` | `e9f1678141ff4a57bf17d5cf52e0a27b` | `1ee26fe6cab54961b986f0c552838954` | `d262f8e8502e4b7b88f381a9b7a4e4fb` | `1e130afe90a842028aed70edadbfb9f1` | `9c050999cae047ffb4181315d7f321af` | `0049a1c1e17e4ac29de951a509838cd4` |
| button (1레벨업·공용 카드·편성) | `65d8340b880940b5b2d2d3342dad63aa` | `292e7807ec754ea19ff3c28ec4b10311` | `7e3db0c2c7944ba29d49c35222ad98b3` | `b5179a32f5c54fabbd757a0d8a2fbe18` | `8739b8a9b7e84759abd297cae38822dc` | `7680060fe5824db88ccbec16b68ff83c` | `dd5c7090249c4601b00410cead0c846b` |
| action (최대 레벨업·전직) | `ffd05f5bf0274527b1e5cb977f2c49fe` | `2a218c009982464f9c6f581e763ea6aa` | `3127fb2d53b14c6a94e8d0c7e309ac92` | `db304db9f8de48278c56d7b13071f0d5` | (Normal과 같음) | — | — |
| close (공통 X 64×64) | `63cc491b18bd41ab961880e152b57e4d` | `234a3cef58ae4954a5ff024688ca66fd` | `6b79d55bd6d44f18b5ab84ffbe16da72` | `0473791f50d642249eb0fb7f4738e791` | — | — | — |
| 패널 | panel_outer `ad1157dddb334c91b1b58c1073e060e1` / panel_light `951c44d6b73c4d46929c47a479cfb4aa` / panel_inner `bea05f1ee4ea484eb10471a620c69046` / outline_selected `f6850016d5a94cfba07d55858733d8f1` |||||||

기타 리소스:

| 용도 | RUID | 출처 |
|---|---|---|
| 파란 구분선(성장창 `TabJoin`) | `e64a2ce6121a4dbf9d377aa8204cf00c`, Color (0.15, 0.54, 0.81, 1), Simple | `ui/GrowthUI` Root/TabJoin |
| 자물쇠 아이콘 | `f190dca8fa0c4209b38eee869bd5e4ea` | `Lobby/RegionSelectLogic.mlua` `IconLock` |
| 직업 아이콘(4차 주력기 스킬 아이콘) | 히어로 `644fc39bcc684c209264eb9d9a4cd6f8`(레이징 블로우) / 비숍 `a1382da8fa7c417296cdff334f57acb1`(엔젤레이) / 나이트로드 `da7c1ce80f2c44079b2130022212bbbf`(쿼드러플 스로우) / 보우마스터 `54008fb6d99c400db04783b61de1c0bb`(폭풍의 시) / 바이퍼 `d8c5e12e2a6640bfa9aa6652ca79660b`(데몰리션) | `HeroSkillTable.csv` IconRUID. 코드에서는 표에서 읽는다: `CharSlotTable`의 `Main` 칸 `T4` Id → `HeroSkillTable.IconRUID` |
| 로비 "캐릭터" 버튼 아이콘 | 기존 업적 아이콘 `983bdc209e734848987d4e9f0b3c803c` 그대로(임시) | 전용 캐릭터 아이콘이 없음 → 보고 목록 |

경험의 서·전직 카드 전용 아이콘은 프로젝트에 없다. 이번 단계는 **글자로만** 표시하고 "빠진 리소스"로 보고한다.

### 2-3. 엔티티 표

`Root` 아래 (Root 중심 = 화면 중심)

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Root` | panel(빈) | 0,0 | 1920×1080 | — | enable false |
| `Root/Bg` | sprite | 0,0 | 1920×1080 | panel_outer, Sliced | 성장창 Bg와 같은 값 |
| `Root/Frame` | sprite | 0,0 | 1920×1080 | panel_light, Sliced | 성장창 Frame과 같은 값 |
| `Root/Title` | text | 앵커·피벗 (0,1), pos 28,-24 | 300×60 | `캐릭터`, 40, 흰색 | 성장창 Title 값 복사 |
| `Root/Close` | button | 앵커·피벗 (1,1), pos -20,-20 | 64×64 | close 4상태, Simple, 글자 없음 | |
| `Root/HeadLine` | sprite | 0,360.5 | 1904×3 | 파란 구분선 | |
| `Root/PartyText` | text | -260,400 | 600×44 | `편성  1 히어로 · 2 비숍 · 3 나이트로드`, 26, 흰색, 왼쪽 정렬 | 이름은 현재 차수 이름 |
| `Root/Party1` | button(button) | 140,400 | 112×76 | `1번`, 26 | 선택 직업을 1번 자리에 |
| `Root/Party2` | button(button) | 260,400 | 112×76 | `2번`, 26 | |
| `Root/Party3` | button(button) | 380,400 | 112×76 | `3번`, 26 | |
| `Root/Currency` | text | 660,400 | 360×48 | `다이아 3,000 · 공용 카드 2`, 26, 흰색, 오른쪽 정렬 | |
| `Root/List` | sprite | -690,-90 | 460×860 | panel_inner, Sliced | 직업 목록 영역 |
| `Root/Detail` | sprite | 230,-90 | 1340×860 | panel_inner, Sliced | 상세 영역 |

`Root/List` 아래 직업 카드 5장 `Job1`~`Job5` (CharJobTable `Order` 순서. 표에 직업이 5개를 넘으면 이번 단계는 앞 5개만 표시하고 보고)

| 경로 | 종류 | pos(List 기준) | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Job{n}` | button(card) | 0, 320 / 160 / 0 / -160 / -320 | 420×148 | card 7상태 | 선택 = Selected 유지 |
| `Job{n}/Icon` | sprite | -140,0 | 96×96 | 직업 아이콘, PreserveSprite None, Raycast false | 잠기면 Color 0.70 틴트 |
| `Job{n}/Name` | text | 40,36 | 240×40 | `파이터`, 30, 일반색, 왼쪽 | 현재 차수 이름 |
| `Job{n}/Sub` | text | 40,-4 | 240×32 | `전사 · Lv.9 · 2차`, 22, 보조색, 왼쪽 | 잠김: `전사 · 잠김` |
| `Job{n}/Cards` | text | 40,-40 | 240×30 | `전용 카드 2장`, 22, 보조색, 왼쪽 | |
| `Job{n}/PartyMark` | sprite | 172,46 | 48×48 | button_selected `8739b8a9b7e84759abd297cae38822dc`, Sliced, Raycast false | 편성된 직업만 enable |
| `Job{n}/PartyMark/Num` | text | 0,0 | 48×48 | `1`, 24, 흰색, 가운데 | |
| `Job{n}/Lock` | sprite | 0,0 | 420×148 | panel_outer, Sliced, Color (1,1,1,0.55), Raycast false | 잠긴 직업만 enable (어두운 덮개) |
| `Job{n}/Lock/Icon` | sprite | 150,0 | 56×56 | 자물쇠, Raycast false | |

- 잠긴 카드도 **클릭 가능**(상세에서 해금 버튼을 보려고). 덮개와 아이콘은 입력을 가로채지 않게 Raycast false.

`Root/Detail` 아래 (Detail 기준 좌표, 가로 -670~670, 세로 -430~430)

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `PreviewBox` | sprite | -450,170 | 380×460 | card_normal `a88cfe62e8e64c39bf14d682a77ef4a0`, Sliced | 미리보기 칸 |
| `Avatar` | avatar + CostumeManager | -450,150 | 300×400 | 3-1 참고 | 2-C3. 실패 시 enable false |
| `PreviewName` | text | -450,-35 | 360×36 | `파이터 (2차)`, 26, 일반색, 가운데 | 아바타 아래 이름 |
| `StatTitle` | text | -450,-130 | 340×36 | `능력치`, 26, 일반색, 왼쪽 | |
| `StatAtk` | text | -450,-185 | 340×40 | `공격력  24`, 28, 일반색, 왼쪽 | |
| `StatHp` | text | -450,-235 | 340×40 | `체력  211` | |
| `StatDef` | text | -450,-285 | 340×40 | `방어력  9` | |
| `StatNote` | text | -450,-345 | 360×56 | `차수 보너스 +10%\n전직하면 +20%`, 22, 보조색, 왼쪽, 2줄 | 5차면 1줄 |
| `Group` | text | 210,380 | 840×36 | `전사 · 히어로 계열`, 24, 보조색, 왼쪽 | `JobGroup · 4차 이름 + " 계열"` |
| `TierName` | text | 210,330 | 840×60 | `파이터 (2차)`, 44, 일반색, 굵게, 왼쪽 | |
| `LevelText` | text | 210,272 | 840×40 | `Lv.9  /  관문 Lv.13`, 30, 일반색, 왼쪽 | 5차 Lv30: `Lv.30 (최고)` |
| `ExpBg` | sprite | 210,222 | 840×28 | card_disabled `d262f8e8502e4b7b88f381a9b7a4e4fb`, Sliced | |
| `ExpFill` | sprite | 앵커 (0.5,0.5) 피벗 (0,0.5), pos -210,222 | 가변 ×28 | action_normal `ffd05f5bf0274527b1e5cb977f2c49fe`, Sliced | 폭 = `840 × exp / NeedExp(level)`. 폭 < 24면 enable false (슬라이스 경계 12+12 보호). 관문·최고 레벨이면 폭 840 |
| `ExpText` | text | 210,187 | 840×30 | `경험치 300 / 620`, 22, 보조색, 왼쪽 | 관문: `관문 도달 — 전직하면 계속 성장합니다` |
| `LvTitle` | text | 210,130 | 840×36 | `레벨업 — 경험의 서`, 26, 일반색, 왼쪽 | |
| `Book1` / `Book2` / `Book3` | sprite | -80,62 / 210,62 / 500,62 | 260×90 | card_normal, Sliced | 소 / 중 / 대 |
| `Book{n}/Name` | text | 0,18 | 240×32 | `경험의 서 (소)`, 22, 일반색, 가운데 | |
| `Book{n}/Count` | text | 0,-18 | 240×30 | `50권 · 권당 100`, 20, 보조색, 가운데 | 0권이면 부족색 |
| `PreviewOne` | text | 210,2 | 840×32 | `1레벨업  Lv.9 → 10  ·  소 2 · 중 1 · 대 0`, 22, 일반색, 왼쪽 | 4-3 |
| `PreviewMax` | text | 210,-34 | 840×32 | `최대  Lv.9 → 13  ·  소 1 · 중 3 · 대 1`, 22, 일반색, 왼쪽 | 넘침 있으면 뒤에 `(넘치는 경험치 80은 사라집니다)` |
| `BtnLv1` | button(button) | -10,-105 | 320×88 | `1레벨업`, 28 | |
| `BtnLvMax` | button(action) | 350,-105 | 360×88 | `최대 레벨업`, 28, 흰 글자 | |
| `PromoteTitle` | text | 210,-185 | 840×36 | `전직 — 다음 차수: 크루세이더 (3차)`, 26, 일반색, 왼쪽 | |
| `PromoteInfo` | text | 210,-230 | 840×32 | `필요 레벨 Lv.13 (현재 9) · 전용 카드 1 / 2장 · 공용 카드 2장`, 22, 왼쪽 | 조건 미달이면 부족색, 충족이면 유리색 |
| `PromoteNote` | text | 210,-266 | 840×30 | `전직하면 같은 레벨에서 다음 차수가 시작되고 능력치 보너스가 오릅니다`, 20, 보조색, 왼쪽 | |
| `BtnPromote` | button(action) | -10,-340 | 320×88 | `전직`, 28, 흰 글자 | |
| `BtnPromoteCommon` | button(button) | 350,-340 | 360×88 | `공용 카드로 전직`, 26 | 부족분 표시: `공용 카드로 전직 (1장)` |
| `BtnUnlock` | button(action) | 350,-340 | 360×88 | `공용 카드로 해금 (1장)`, 26, 흰 글자 | 잠긴 직업에서만 enable (`BtnPromote*`와 같은 자리, 동시에 켜지지 않음) |
| `Msg` | text | 210,-405 | 840×28 | 결과 문구, 22, 왼쪽 | 2.5초 뒤 숨김 |

글자가 한 줄을 넘는 텍스트(`PromoteInfo`, `PreviewMax`)는 `BestFit`(최소 16)을 켠다.

### 2-4. 화면 상태별 표시

| 선택 직업 상태 | 레벨업 영역 | 전직 영역 |
|---|---|---|
| 잠김 | `Book*`는 보유량만 표시, `PreviewOne/Max` = `잠긴 직업입니다`, `BtnLv1`·`BtnLvMax` 비활성 | `PromoteTitle` = `잠긴 직업 — 캐릭터 뽑기에서 전용 카드를 얻으면 해금됩니다`, `PromoteInfo` = `공용 카드 n장 보유`, `BtnPromote`·`BtnPromoteCommon` enable false, `BtnUnlock` enable true (공용 ≥ 1이면 활성) |
| 해금, 관문 전(level < cap) | 미리보기 2줄, 책이 있으면 활성 | 전직 정보 표시, `BtnPromote`·`BtnPromoteCommon` 비활성(레벨 미달) |
| 해금, 관문 도달(level == cap, tier < 5) | `PreviewOne/Max` = `관문 도달 — 전직이 필요합니다`, 두 버튼 비활성 | 전용 ≥ 필요: `BtnPromote` 활성, `BtnPromoteCommon` 비활성 / 전용 < 필요 ≤ 전용+공용: `BtnPromote` 비활성, `BtnPromoteCommon` 활성 / 그 외 둘 다 비활성 |
| 5차 Lv30 | `PreviewOne/Max` = `최고 레벨입니다`, 두 버튼 비활성 | `PromoteTitle` = `최고 차수 달성`, `PromoteInfo`·`PromoteNote` 빈칸, 두 버튼 enable false |
| 5차 Lv < 30 | 일반 | `PromoteTitle` = `최고 차수 (5차)`, 두 버튼 enable false |

- 요청을 보낸 뒤 결과(`RosterSeq` 증가)가 오거나 3초가 지날 때까지 **모든 실행 버튼(레벨업·전직·해금·편성)을 비활성**으로 둔다(연타 방지).
- 편성 버튼: 선택 직업이 잠기면 3개 모두 비활성. `party[k] == 선택 직업`이면 `k`번 버튼 Selected 표시(클릭해도 요청 안 보냄).

---

## 3. 미리보기 아바타 (2-C3)

### 3-1. UI 엔티티

- `b.avatar("Root/Detail/Avatar", { pos: [-450,150], rect_size: [300,400], raycast: false })` 후 같은 엔티티에 `b.addComponent(path, "MOD.Core.CostumeManagerComponent", {...})` — 값: `UseCustomEquipOnly: true`, `CustomBodyEquip: "3cf4cb29ae344577b9eed37e531688a4"`, `CustomHairEquip: "29246028db7642baba92689a1ef777bb"`, `CustomFaceEquip: "8355ad1710e04ea3b1fc800951caa977"` (DefaultPlayer·구성원 모델과 같은 몸·머리·얼굴, ModelBuilder로 확인한 값).
- 대기 동작 재생이 필요하면 MSW 가이드처럼 `StateComponent` + `AvatarStateAnimationComponent`도 붙인다. 없어도 정지 모습은 나온다(메이커에서 확인).
- 클라이언트 `CharacterUILogic.ShowPreview(job, tier)`: `local cm = avatarEntity.CostumeManagerComponent` → `roster:OutfitFor(job, tier)`의 부위를 4-2와 같은 규칙으로 `cm:SetEquip(category, ruid)`(없는 부위는 `""`). 잠긴 직업은 1차 세트 + `AvatarGUIRendererComponent.Color = Color(0.45,0.45,0.45,1)`.
- 확인: 직업·차수 바꿀 때 옷이 바뀌는지 스크린샷. **표시가 안 되거나 오류가 나면 `Avatar`를 enable false로 두고 `PreviewName`만 남긴 뒤 보고**한다(기능 필수 아님).

---

## 4. 차수 외형

### 4-1. 데이터

**`CharJobTable.csv`에 열 추가** — `ModelId` 다음에 `WeaponFamily`. 값: `Hero=T, Bishop=O, NightLord=O, Bowmaster=T, Viper=O`. `PlayerRoster.LoadTables`에서 `j.weaponFamily = jt:GetCell(i,"WeaponFamily") or "O"`.

**새 표 `CharOutfitTable.csv`** (UTF-8 BOM) + `CharOutfitTable.userdataset`(기존 `RelicTable.userdataset` 복사, 새 uuid, name·id = `CharOutfitTable`).

- 열: `Job,Tier,Slot,RUID,#Name` (`#Name`은 사람이 읽는 아이템 이름, 코드에서는 무시)
- `Slot` 값 = `MapleAvatarItemCategory` 이름 그대로: `Cap / Coat / Pants / Longcoat / Glove / Shoes / Cape / OneHandedWeapon / TwoHandedWeapon`. 무기 칸은 **아이템 자체의 분류**(리소스 API `category`가 `weapon` → OneHandedWeapon, `twohandweapon` → TwoHandedWeapon)를 따른다. 그래서 아대·너클도 TwoHandedWeapon 칸에 들어간다.
- 세트에 없는 부위는 행이 없다 = 입히지 않음(빈 칸).
- 상의+하의(`Coat`+`Pants`)와 한벌옷(`Longcoat`)은 한 세트 안에 섞지 않았다.

```csv
Job,Tier,Slot,RUID,#Name
Hero,1,Cap,05c4f79aa6894cd9846f154fb0e678ed,메탈 코이프
Hero,1,Coat,adaf82ca5e144526ad2d5491451b1a7d,브라운 로리카 아머
Hero,1,Pants,52d008d5f89e40f7a552ef32b3d7df8c,브라운 로리카 바지
Hero,1,Shoes,380d0a7f13b04d28a2b17ffdb32c27f2,스틸 그리브
Hero,1,TwoHandedWeapon,43ad7bfeb9184252b24e6d60f76c17e5,목검
Hero,2,Cap,06855f3baa3444a48a0c369d0c6cff64,스틸 샤프 헬름
Hero,2,Coat,8d0ca70ba5594bf5b9b8987b7cb92a67,레드 숄더메일
Hero,2,Pants,f34cb05fda4449e3b6f49f77a895dc9e,레드 숄더메일 바지
Hero,2,Glove,04e49af681b643c3845be3104c444360,베논
Hero,2,Shoes,2a67053e769041df8329001ecba64224,다크 워 부츠
Hero,2,TwoHandedWeapon,40f7a7d1e55c417f8c2ecbddcea536d2,그륜힐
Hero,3,Cap,819ebb46d5e74dac8fca8f871d2b46d5,브론즈 플레닛
Hero,3,Coat,cdef422d64e749178704497ad0a012b7,블러드 플라티나
Hero,3,Pants,5c45b7c5b73d497eb00351af348b1c41,블러드 플라티나 바지
Hero,3,Glove,60ceb9ee59c94a2aadbc8c8a1358c42a,다크 허스크
Hero,3,Shoes,f709b9af772846818b103c37487f7807,다크 카젠부츠
Hero,3,Cape,08879b8c46ea43fa8206b22dff8fce97,검은색 세라프의 망토
Hero,3,TwoHandedWeapon,79f564e8e5884324aa2e230e1d04fa92,막스 반 레온 스워드
Hero,4,Cap,de9c87efde854fac8c7e7be4662cab76,하이네스 워리어헬름
Hero,4,Coat,2c9ebd4a71654d069da14f9411ceaebd,이글아이 워리어아머
Hero,4,Pants,8c3772942b914ed7be40733439b42827,트릭스터 워리어팬츠
Hero,4,Glove,ad76497733c74828b4e6aaeff251c35d,앱솔랩스 나이트글러브
Hero,4,Shoes,9885f9d8ee00456e842d0b19b1caae1f,앱솔랩스 나이트슈즈
Hero,4,Cape,1cd3d12f6c184166a97ebbdd74bce6a1,앱솔랩스 나이트케이프
Hero,4,TwoHandedWeapon,b5a56a2da60b4b73afb3331742ec599d,앱솔랩스 브로드세이버
Hero,5,Cap,927bc02eee4d4448bf82d288c7e83cbb,아케인셰이드 나이트햇
Hero,5,Longcoat,f0d982e53a7b433ab08044df6e327da7,아케인셰이드 나이트슈트
Hero,5,Glove,dfd4da8447cd45219f57ba91d3ad642d,아케인셰이드 나이트글러브
Hero,5,Shoes,ce9c38b11f72410483fed1cbbe148480,아케인셰이드 나이트슈즈
Hero,5,Cape,54c1be1e29f94560859afde688d5bbae,아케인셰이드 나이트케이프
Hero,5,TwoHandedWeapon,e1ee66e2320b4973a2fd0fdde4e84f32,아케인셰이드 투핸드소드
Bishop,1,Cap,5749a4754ec541a3a40d69a35ebc02b8,파란색 사제의 모자
Bishop,1,Longcoat,f28a1a593e244e9387f3e6ddba503a82,파란색 사제의 로브
Bishop,1,Shoes,8d90bfb6fcb44494b7053bb313911f21,블루 매직슈즈
Bishop,1,OneHandedWeapon,061712eadf7d4357a6c306dfd5368860,우드 완드
Bishop,2,Cap,b290bc5571ca449ab32856b0dba5837c,블루 크로스햇
Bishop,2,Longcoat,8992a9e69da247068eb5051f736bfdda,블루 도로스 로브
Bishop,2,Glove,abf55a6607224f76a09214b18771a82b,블루 와이즈
Bishop,2,Shoes,04179e7c41de482d8c3092415e64ef22,블루 문슈즈
Bishop,2,OneHandedWeapon,3ae7d735d1d5420697f525f0b0696ce1,미스릴 완드
Bishop,3,Cap,63b8793aec784f5b8940cb10f05af685,블루 핀터
Bishop,3,Longcoat,b689ca348ef14925ac9ff4c3434fb911,블루 데빌즈 로브
Bishop,3,Glove,453691633a804142aa2cee8b1cb36ecd,블루 시아르
Bishop,3,Shoes,44884e29e2b743718c3ee21f603f58e6,블루 쥬얼리 슈즈
Bishop,3,Cape,5bf669b99da14b3aa6b53a6d93f2238a,흰색 모험가의 망토
Bishop,3,OneHandedWeapon,102c15b28a944d2f905872dd10f795ec,크리스탈 완드
Bishop,4,Cap,3f5d39f2af304dd8be093be4eb0e3a44,하이네스 던위치햇
Bishop,4,Coat,54a6dcc69fc84963b7ae468ce7d151dc,이글아이 던위치로브
Bishop,4,Pants,db98c2a542d0445580cefdf352567a59,트릭스터 던위치팬츠
Bishop,4,Glove,ad76497733c74828b4e6aaeff251c35d,앱솔랩스 나이트글러브
Bishop,4,Shoes,9885f9d8ee00456e842d0b19b1caae1f,앱솔랩스 나이트슈즈
Bishop,4,Cape,1cd3d12f6c184166a97ebbdd74bce6a1,앱솔랩스 나이트케이프
Bishop,4,OneHandedWeapon,6af36c66763d4076bf1129abf1f9c364,파프니르 마나테이커
Bishop,5,Cap,06feb050faf144289945a2519a4767d6,아케인셰이드 메이지햇
Bishop,5,Longcoat,a836af46f7d947f785dd5a42a1e56690,아케인셰이드 메이지슈트
Bishop,5,Glove,dfd4da8447cd45219f57ba91d3ad642d,아케인셰이드 나이트글러브
Bishop,5,Shoes,ce9c38b11f72410483fed1cbbe148480,아케인셰이드 나이트슈즈
Bishop,5,Cape,54c1be1e29f94560859afde688d5bbae,아케인셰이드 나이트케이프
Bishop,5,OneHandedWeapon,0de959f44ddb4a7c8ab39c9db557d37b,아케인셰이드 완드
NightLord,1,Cap,bed1990d5b6a4655a0adbce3ae204110,검정색 좀도둑 털모자
NightLord,1,Longcoat,688e0c08b07440dca0a036e8606109bb,블랙 가리나
NightLord,1,Shoes,a5309ee66252440d981b6463c54c31c5,블랙 가리나 슈즈
NightLord,1,TwoHandedWeapon,985e44451afc4730a46eb206716e7ecf,가니어
NightLord,2,Cap,a2354df0a94b4e708ab28d17ea4fc8bb,다크 하이드후드
NightLord,2,Longcoat,94946182be6142ddae01fd0dc3893b98,다크 후르츠
NightLord,2,Glove,b9aa861b8e1042c585cd88826fdf47c9,다크 클리브
NightLord,2,Shoes,5209e9d5b7a84a9da273d3ac9d9c0f4f,다크 모스부츠
NightLord,2,TwoHandedWeapon,de0cfdaae45345dd92e7875d33003abe,아다만티움 이고르
NightLord,3,Cap,ca657a96142547e1955292d89a8a7583,블랙 루즈캡
NightLord,3,Longcoat,28625b9456224bf69f66768a15e2d649,어둠의 그림자복
NightLord,3,Glove,7dce36acb4e642e885426563fef34055,다크 미스트라
NightLord,3,Shoes,0a92aafa10554e7bb2d50cc60fc5adf9,퍼플 쉐도우부츠
NightLord,3,Cape,04458a90640c424bbbe3921952dda263,검은색 모험가의 망토
NightLord,3,TwoHandedWeapon,5b1995dfa3d94971bbaa37d970426069,레드 크리븐
NightLord,4,Cap,545689223bb6479181771e7f416c344c,하이네스 어새신보닛
NightLord,4,Coat,368dfa5b12844367939a92fed25a2f1e,이글아이 어새신셔츠
NightLord,4,Pants,af1cb630e9904ebd80fa6af94f11f41a,트릭스터 어새신팬츠
NightLord,4,Glove,ad76497733c74828b4e6aaeff251c35d,앱솔랩스 나이트글러브
NightLord,4,Shoes,9885f9d8ee00456e842d0b19b1caae1f,앱솔랩스 나이트슈즈
NightLord,4,Cape,1cd3d12f6c184166a97ebbdd74bce6a1,앱솔랩스 나이트케이프
NightLord,4,TwoHandedWeapon,0e36c778dcff42438c76abd44d8b485d,파프니르 리스크홀더
NightLord,5,Cap,8ac5385cd4fd47f4ad875626cdd74174,아케인셰이드 시프햇
NightLord,5,Longcoat,dab97520b06e4ff684775c86388abe99,아케인셰이드 시프슈트
NightLord,5,Glove,dfd4da8447cd45219f57ba91d3ad642d,아케인셰이드 나이트글러브
NightLord,5,Shoes,ce9c38b11f72410483fed1cbbe148480,아케인셰이드 나이트슈즈
NightLord,5,Cape,54c1be1e29f94560859afde688d5bbae,아케인셰이드 나이트케이프
NightLord,5,TwoHandedWeapon,b696eee8e7ba473ab78d59cd20c8a23c,아케인셰이드 가즈
Bowmaster,1,Cap,c0c01ebb890f41dfb73bd80888212dcc,초록색 고급 가죽 모자
Bowmaster,1,Longcoat,868f3b09b9a045038ed57e4d60480cf2,그린 아르미스
Bowmaster,1,Shoes,d17c62c7a81b43abaad097f53da9091d,그린 헌터부츠
Bowmaster,1,TwoHandedWeapon,3ffe515570ad435caa397ca0f2e2f83b,워 보우
Bowmaster,2,Cap,3eac0ca56b6c4432aa7e9f90cc2e0ed5,그린 크라페
Bowmaster,2,Longcoat,3456ad15a9d94eb19350d8a75285a1c1,그린 아데스
Bowmaster,2,Glove,8c78d894ca1944ae9847409efc8761ef,그린 아데브
Bowmaster,2,Shoes,f13d4fbd48464203a37be2b1bfc4661c,그린 아데스슈즈
Bowmaster,2,TwoHandedWeapon,edde225a13754c149f7191fd8a2dac35,메이플 보우
Bowmaster,3,Cap,1cf81d3439ba40fdab6bd6f62d4b765c,그린 폴레패더햇
Bowmaster,3,Longcoat,8c5c5bdff4504328b906b812f0aa282f,그린 리넥스
Bowmaster,3,Glove,671eac92ecce48cd8211630e9632f252,그린 코든
Bowmaster,3,Shoes,d1619f368e3449e69156b139e1cda91e,그린 리넥스슈즈
Bowmaster,3,Cape,a7651f5bc6034bcda63101867490e8d3,초록색 모험가의 망토
Bowmaster,3,TwoHandedWeapon,1703296c5f194f47ad2fab061685225d,메이플 칸데오
Bowmaster,4,Cap,3688ac5098ab4266b87af7e943844f7a,하이네스 레인져베레
Bowmaster,4,Coat,81af6401f7af49b1a930be2ab477223c,이글아이 레인져후드
Bowmaster,4,Pants,1e6aec7f5f9d4ef1983892b1c0025e80,트릭스터 레인져팬츠
Bowmaster,4,Glove,ad76497733c74828b4e6aaeff251c35d,앱솔랩스 나이트글러브
Bowmaster,4,Shoes,9885f9d8ee00456e842d0b19b1caae1f,앱솔랩스 나이트슈즈
Bowmaster,4,Cape,1cd3d12f6c184166a97ebbdd74bce6a1,앱솔랩스 나이트케이프
Bowmaster,4,TwoHandedWeapon,18d80518f407457aa489da97c5fead0d,파프니르 윈드체이서
Bowmaster,5,Cap,b72cc1ef4d2d4f64866677050e0a9fb9,아케인셰이드 아처햇
Bowmaster,5,Longcoat,e93e89b82c274f0694102271585934f7,아케인셰이드 아처슈트
Bowmaster,5,Glove,dfd4da8447cd45219f57ba91d3ad642d,아케인셰이드 나이트글러브
Bowmaster,5,Shoes,ce9c38b11f72410483fed1cbbe148480,아케인셰이드 나이트슈즈
Bowmaster,5,Cape,54c1be1e29f94560859afde688d5bbae,아케인셰이드 나이트케이프
Bowmaster,5,TwoHandedWeapon,f0c8f18069be499f92736cbf5acb7eaa,아케인셰이드 보우
Viper,1,Cap,324bc734a43c423191a399f3925062a1,브라운 피레타햇
Viper,1,Longcoat,3d26c09ea6284d6c8e1fe625a9e48721,브라운 툴크갤리
Viper,1,Shoes,0cc27439d15a4e69be1d041e0803ccb8,브라운 레더크라그
Viper,1,TwoHandedWeapon,9c03a78d2caf49a982a1d4b1d924cee4,스틸 너클
Viper,2,Cap,bb9dc197b65648d1b40af3388234a84c,다크 피레타햇
Viper,2,Longcoat,e80bfac34d494e42a87d568db43d5fba,레드 비스카
Viper,2,Glove,b1e8c6a49eb8452d98f95009739b73fd,브라운 레더 아머글로브
Viper,2,Shoes,16bd622735784efb944bfe4a6ada1487,블랙 바실즈 부츠
Viper,2,TwoHandedWeapon,1c703981fc754042871cc8abb589d0ff,메이플 스톰 핑거
Viper,3,Cap,65e4777b6e50484ba228fed7aa9096aa,블랙 파이렛츠 반다나
Viper,3,Longcoat,1c73a794497942cabb9848a951aacfbe,블랙 로얄바롱
Viper,3,Glove,a61dd22c78e94536bd37201e7e31baf4,블랙 스켈듀크
Viper,3,Shoes,11877303850542e18d33bb4fbcc374bb,블랙 듀크바킨 슈즈
Viper,3,Cape,724a9e818ddd41ba9825b6fcce789521,빨간색 모험가의 망토
Viper,3,TwoHandedWeapon,b11507a9c7864a3e822d88c5386049b8,레드 너클
Viper,4,Cap,57569b6988fe4acead1277299cb6af84,하이네스 원더러햇
Viper,4,Coat,7a925a2880e747228144c93ee0a3d92d,이글아이 원더러코트
Viper,4,Pants,983e931f4ce54ad88a99d8664eb82b12,트릭스터 원더러팬츠
Viper,4,Glove,ad76497733c74828b4e6aaeff251c35d,앱솔랩스 나이트글러브
Viper,4,Shoes,9885f9d8ee00456e842d0b19b1caae1f,앱솔랩스 나이트슈즈
Viper,4,Cape,1cd3d12f6c184166a97ebbdd74bce6a1,앱솔랩스 나이트케이프
Viper,4,TwoHandedWeapon,3fbc7c62d4d5453bb5a8dca5d8671e0e,파프니르 펜리르탈론
Viper,5,Cap,db33b98e092046af81d94dfdefde203a,아케인셰이드 파이렛햇
Viper,5,Longcoat,a0289186949a4e8389b540612d05399a,아케인셰이드 파이렛슈트
Viper,5,Glove,dfd4da8447cd45219f57ba91d3ad642d,아케인셰이드 나이트글러브
Viper,5,Shoes,ce9c38b11f72410483fed1cbbe148480,아케인셰이드 나이트슈즈
Viper,5,Cape,54c1be1e29f94560859afde688d5bbae,아케인셰이드 나이트케이프
Viper,5,TwoHandedWeapon,a5602ea49007472a96e4887cb0dfda9e,아케인셰이드 클로
```

### 4-2. 선정 표 (리소스 API `searchAvatarItems` 이름 정확 일치로 찾고 `getResourcesBatch`로 재검증)

| 직업 | 차수 | 부위 (Slot) | 아이템 이름 | RUID |
|---|---|---|---|---|
| Hero | 1 | 모자 (Cap) | 메탈 코이프 | `05c4f79aa6894cd9846f154fb0e678ed` |
| Hero | 1 | 상의 (Coat) | 브라운 로리카 아머 | `adaf82ca5e144526ad2d5491451b1a7d` |
| Hero | 1 | 하의 (Pants) | 브라운 로리카 바지 | `52d008d5f89e40f7a552ef32b3d7df8c` |
| Hero | 1 | 신발 (Shoes) | 스틸 그리브 | `380d0a7f13b04d28a2b17ffdb32c27f2` |
| Hero | 1 | 두손 무기 (TwoHandedWeapon) | 목검 | `43ad7bfeb9184252b24e6d60f76c17e5` |
| Hero | 2 | 모자 (Cap) | 스틸 샤프 헬름 | `06855f3baa3444a48a0c369d0c6cff64` |
| Hero | 2 | 상의 (Coat) | 레드 숄더메일 | `8d0ca70ba5594bf5b9b8987b7cb92a67` |
| Hero | 2 | 하의 (Pants) | 레드 숄더메일 바지 | `f34cb05fda4449e3b6f49f77a895dc9e` |
| Hero | 2 | 장갑 (Glove) | 베논 | `04e49af681b643c3845be3104c444360` |
| Hero | 2 | 신발 (Shoes) | 다크 워 부츠 | `2a67053e769041df8329001ecba64224` |
| Hero | 2 | 두손 무기 (TwoHandedWeapon) | 그륜힐 | `40f7a7d1e55c417f8c2ecbddcea536d2` |
| Hero | 3 | 모자 (Cap) | 브론즈 플레닛 | `819ebb46d5e74dac8fca8f871d2b46d5` |
| Hero | 3 | 상의 (Coat) | 블러드 플라티나 | `cdef422d64e749178704497ad0a012b7` |
| Hero | 3 | 하의 (Pants) | 블러드 플라티나 바지 | `5c45b7c5b73d497eb00351af348b1c41` |
| Hero | 3 | 장갑 (Glove) | 다크 허스크 | `60ceb9ee59c94a2aadbc8c8a1358c42a` |
| Hero | 3 | 신발 (Shoes) | 다크 카젠부츠 | `f709b9af772846818b103c37487f7807` |
| Hero | 3 | 망토 (Cape) | 검은색 세라프의 망토 | `08879b8c46ea43fa8206b22dff8fce97` |
| Hero | 3 | 두손 무기 (TwoHandedWeapon) | 막스 반 레온 스워드 | `79f564e8e5884324aa2e230e1d04fa92` |
| Hero | 4 | 모자 (Cap) | 하이네스 워리어헬름 | `de9c87efde854fac8c7e7be4662cab76` |
| Hero | 4 | 상의 (Coat) | 이글아이 워리어아머 | `2c9ebd4a71654d069da14f9411ceaebd` |
| Hero | 4 | 하의 (Pants) | 트릭스터 워리어팬츠 | `8c3772942b914ed7be40733439b42827` |
| Hero | 4 | 장갑 (Glove) | 앱솔랩스 나이트글러브 | `ad76497733c74828b4e6aaeff251c35d` |
| Hero | 4 | 신발 (Shoes) | 앱솔랩스 나이트슈즈 | `9885f9d8ee00456e842d0b19b1caae1f` |
| Hero | 4 | 망토 (Cape) | 앱솔랩스 나이트케이프 | `1cd3d12f6c184166a97ebbdd74bce6a1` |
| Hero | 4 | 두손 무기 (TwoHandedWeapon) | 앱솔랩스 브로드세이버 | `b5a56a2da60b4b73afb3331742ec599d` |
| Hero | 5 | 모자 (Cap) | 아케인셰이드 나이트햇 | `927bc02eee4d4448bf82d288c7e83cbb` |
| Hero | 5 | 한벌옷 (Longcoat) | 아케인셰이드 나이트슈트 | `f0d982e53a7b433ab08044df6e327da7` |
| Hero | 5 | 장갑 (Glove) | 아케인셰이드 나이트글러브 | `dfd4da8447cd45219f57ba91d3ad642d` |
| Hero | 5 | 신발 (Shoes) | 아케인셰이드 나이트슈즈 | `ce9c38b11f72410483fed1cbbe148480` |
| Hero | 5 | 망토 (Cape) | 아케인셰이드 나이트케이프 | `54c1be1e29f94560859afde688d5bbae` |
| Hero | 5 | 두손 무기 (TwoHandedWeapon) | 아케인셰이드 투핸드소드 | `e1ee66e2320b4973a2fd0fdde4e84f32` |
| Bishop | 1 | 모자 (Cap) | 파란색 사제의 모자 | `5749a4754ec541a3a40d69a35ebc02b8` |
| Bishop | 1 | 한벌옷 (Longcoat) | 파란색 사제의 로브 | `f28a1a593e244e9387f3e6ddba503a82` |
| Bishop | 1 | 신발 (Shoes) | 블루 매직슈즈 | `8d90bfb6fcb44494b7053bb313911f21` |
| Bishop | 1 | 한손 무기 (OneHandedWeapon) | 우드 완드 | `061712eadf7d4357a6c306dfd5368860` |
| Bishop | 2 | 모자 (Cap) | 블루 크로스햇 | `b290bc5571ca449ab32856b0dba5837c` |
| Bishop | 2 | 한벌옷 (Longcoat) | 블루 도로스 로브 | `8992a9e69da247068eb5051f736bfdda` |
| Bishop | 2 | 장갑 (Glove) | 블루 와이즈 | `abf55a6607224f76a09214b18771a82b` |
| Bishop | 2 | 신발 (Shoes) | 블루 문슈즈 | `04179e7c41de482d8c3092415e64ef22` |
| Bishop | 2 | 한손 무기 (OneHandedWeapon) | 미스릴 완드 | `3ae7d735d1d5420697f525f0b0696ce1` |
| Bishop | 3 | 모자 (Cap) | 블루 핀터 | `63b8793aec784f5b8940cb10f05af685` |
| Bishop | 3 | 한벌옷 (Longcoat) | 블루 데빌즈 로브 | `b689ca348ef14925ac9ff4c3434fb911` |
| Bishop | 3 | 장갑 (Glove) | 블루 시아르 | `453691633a804142aa2cee8b1cb36ecd` |
| Bishop | 3 | 신발 (Shoes) | 블루 쥬얼리 슈즈 | `44884e29e2b743718c3ee21f603f58e6` |
| Bishop | 3 | 망토 (Cape) | 흰색 모험가의 망토 | `5bf669b99da14b3aa6b53a6d93f2238a` |
| Bishop | 3 | 한손 무기 (OneHandedWeapon) | 크리스탈 완드 | `102c15b28a944d2f905872dd10f795ec` |
| Bishop | 4 | 모자 (Cap) | 하이네스 던위치햇 | `3f5d39f2af304dd8be093be4eb0e3a44` |
| Bishop | 4 | 상의 (Coat) | 이글아이 던위치로브 | `54a6dcc69fc84963b7ae468ce7d151dc` |
| Bishop | 4 | 하의 (Pants) | 트릭스터 던위치팬츠 | `db98c2a542d0445580cefdf352567a59` |
| Bishop | 4 | 장갑 (Glove) | 앱솔랩스 나이트글러브 | `ad76497733c74828b4e6aaeff251c35d` |
| Bishop | 4 | 신발 (Shoes) | 앱솔랩스 나이트슈즈 | `9885f9d8ee00456e842d0b19b1caae1f` |
| Bishop | 4 | 망토 (Cape) | 앱솔랩스 나이트케이프 | `1cd3d12f6c184166a97ebbdd74bce6a1` |
| Bishop | 4 | 한손 무기 (OneHandedWeapon) | 파프니르 마나테이커 | `6af36c66763d4076bf1129abf1f9c364` |
| Bishop | 5 | 모자 (Cap) | 아케인셰이드 메이지햇 | `06feb050faf144289945a2519a4767d6` |
| Bishop | 5 | 한벌옷 (Longcoat) | 아케인셰이드 메이지슈트 | `a836af46f7d947f785dd5a42a1e56690` |
| Bishop | 5 | 장갑 (Glove) | 아케인셰이드 나이트글러브 | `dfd4da8447cd45219f57ba91d3ad642d` |
| Bishop | 5 | 신발 (Shoes) | 아케인셰이드 나이트슈즈 | `ce9c38b11f72410483fed1cbbe148480` |
| Bishop | 5 | 망토 (Cape) | 아케인셰이드 나이트케이프 | `54c1be1e29f94560859afde688d5bbae` |
| Bishop | 5 | 한손 무기 (OneHandedWeapon) | 아케인셰이드 완드 | `0de959f44ddb4a7c8ab39c9db557d37b` |
| NightLord | 1 | 모자 (Cap) | 검정색 좀도둑 털모자 | `bed1990d5b6a4655a0adbce3ae204110` |
| NightLord | 1 | 한벌옷 (Longcoat) | 블랙 가리나 | `688e0c08b07440dca0a036e8606109bb` |
| NightLord | 1 | 신발 (Shoes) | 블랙 가리나 슈즈 | `a5309ee66252440d981b6463c54c31c5` |
| NightLord | 1 | 두손 무기 (TwoHandedWeapon) | 가니어 | `985e44451afc4730a46eb206716e7ecf` |
| NightLord | 2 | 모자 (Cap) | 다크 하이드후드 | `a2354df0a94b4e708ab28d17ea4fc8bb` |
| NightLord | 2 | 한벌옷 (Longcoat) | 다크 후르츠 | `94946182be6142ddae01fd0dc3893b98` |
| NightLord | 2 | 장갑 (Glove) | 다크 클리브 | `b9aa861b8e1042c585cd88826fdf47c9` |
| NightLord | 2 | 신발 (Shoes) | 다크 모스부츠 | `5209e9d5b7a84a9da273d3ac9d9c0f4f` |
| NightLord | 2 | 두손 무기 (TwoHandedWeapon) | 아다만티움 이고르 | `de0cfdaae45345dd92e7875d33003abe` |
| NightLord | 3 | 모자 (Cap) | 블랙 루즈캡 | `ca657a96142547e1955292d89a8a7583` |
| NightLord | 3 | 한벌옷 (Longcoat) | 어둠의 그림자복 | `28625b9456224bf69f66768a15e2d649` |
| NightLord | 3 | 장갑 (Glove) | 다크 미스트라 | `7dce36acb4e642e885426563fef34055` |
| NightLord | 3 | 신발 (Shoes) | 퍼플 쉐도우부츠 | `0a92aafa10554e7bb2d50cc60fc5adf9` |
| NightLord | 3 | 망토 (Cape) | 검은색 모험가의 망토 | `04458a90640c424bbbe3921952dda263` |
| NightLord | 3 | 두손 무기 (TwoHandedWeapon) | 레드 크리븐 | `5b1995dfa3d94971bbaa37d970426069` |
| NightLord | 4 | 모자 (Cap) | 하이네스 어새신보닛 | `545689223bb6479181771e7f416c344c` |
| NightLord | 4 | 상의 (Coat) | 이글아이 어새신셔츠 | `368dfa5b12844367939a92fed25a2f1e` |
| NightLord | 4 | 하의 (Pants) | 트릭스터 어새신팬츠 | `af1cb630e9904ebd80fa6af94f11f41a` |
| NightLord | 4 | 장갑 (Glove) | 앱솔랩스 나이트글러브 | `ad76497733c74828b4e6aaeff251c35d` |
| NightLord | 4 | 신발 (Shoes) | 앱솔랩스 나이트슈즈 | `9885f9d8ee00456e842d0b19b1caae1f` |
| NightLord | 4 | 망토 (Cape) | 앱솔랩스 나이트케이프 | `1cd3d12f6c184166a97ebbdd74bce6a1` |
| NightLord | 4 | 두손 무기 (TwoHandedWeapon) | 파프니르 리스크홀더 | `0e36c778dcff42438c76abd44d8b485d` |
| NightLord | 5 | 모자 (Cap) | 아케인셰이드 시프햇 | `8ac5385cd4fd47f4ad875626cdd74174` |
| NightLord | 5 | 한벌옷 (Longcoat) | 아케인셰이드 시프슈트 | `dab97520b06e4ff684775c86388abe99` |
| NightLord | 5 | 장갑 (Glove) | 아케인셰이드 나이트글러브 | `dfd4da8447cd45219f57ba91d3ad642d` |
| NightLord | 5 | 신발 (Shoes) | 아케인셰이드 나이트슈즈 | `ce9c38b11f72410483fed1cbbe148480` |
| NightLord | 5 | 망토 (Cape) | 아케인셰이드 나이트케이프 | `54c1be1e29f94560859afde688d5bbae` |
| NightLord | 5 | 두손 무기 (TwoHandedWeapon) | 아케인셰이드 가즈 | `b696eee8e7ba473ab78d59cd20c8a23c` |
| Bowmaster | 1 | 모자 (Cap) | 초록색 고급 가죽 모자 | `c0c01ebb890f41dfb73bd80888212dcc` |
| Bowmaster | 1 | 한벌옷 (Longcoat) | 그린 아르미스 | `868f3b09b9a045038ed57e4d60480cf2` |
| Bowmaster | 1 | 신발 (Shoes) | 그린 헌터부츠 | `d17c62c7a81b43abaad097f53da9091d` |
| Bowmaster | 1 | 두손 무기 (TwoHandedWeapon) | 워 보우 | `3ffe515570ad435caa397ca0f2e2f83b` |
| Bowmaster | 2 | 모자 (Cap) | 그린 크라페 | `3eac0ca56b6c4432aa7e9f90cc2e0ed5` |
| Bowmaster | 2 | 한벌옷 (Longcoat) | 그린 아데스 | `3456ad15a9d94eb19350d8a75285a1c1` |
| Bowmaster | 2 | 장갑 (Glove) | 그린 아데브 | `8c78d894ca1944ae9847409efc8761ef` |
| Bowmaster | 2 | 신발 (Shoes) | 그린 아데스슈즈 | `f13d4fbd48464203a37be2b1bfc4661c` |
| Bowmaster | 2 | 두손 무기 (TwoHandedWeapon) | 메이플 보우 | `edde225a13754c149f7191fd8a2dac35` |
| Bowmaster | 3 | 모자 (Cap) | 그린 폴레패더햇 | `1cf81d3439ba40fdab6bd6f62d4b765c` |
| Bowmaster | 3 | 한벌옷 (Longcoat) | 그린 리넥스 | `8c5c5bdff4504328b906b812f0aa282f` |
| Bowmaster | 3 | 장갑 (Glove) | 그린 코든 | `671eac92ecce48cd8211630e9632f252` |
| Bowmaster | 3 | 신발 (Shoes) | 그린 리넥스슈즈 | `d1619f368e3449e69156b139e1cda91e` |
| Bowmaster | 3 | 망토 (Cape) | 초록색 모험가의 망토 | `a7651f5bc6034bcda63101867490e8d3` |
| Bowmaster | 3 | 두손 무기 (TwoHandedWeapon) | 메이플 칸데오 | `1703296c5f194f47ad2fab061685225d` |
| Bowmaster | 4 | 모자 (Cap) | 하이네스 레인져베레 | `3688ac5098ab4266b87af7e943844f7a` |
| Bowmaster | 4 | 상의 (Coat) | 이글아이 레인져후드 | `81af6401f7af49b1a930be2ab477223c` |
| Bowmaster | 4 | 하의 (Pants) | 트릭스터 레인져팬츠 | `1e6aec7f5f9d4ef1983892b1c0025e80` |
| Bowmaster | 4 | 장갑 (Glove) | 앱솔랩스 나이트글러브 | `ad76497733c74828b4e6aaeff251c35d` |
| Bowmaster | 4 | 신발 (Shoes) | 앱솔랩스 나이트슈즈 | `9885f9d8ee00456e842d0b19b1caae1f` |
| Bowmaster | 4 | 망토 (Cape) | 앱솔랩스 나이트케이프 | `1cd3d12f6c184166a97ebbdd74bce6a1` |
| Bowmaster | 4 | 두손 무기 (TwoHandedWeapon) | 파프니르 윈드체이서 | `18d80518f407457aa489da97c5fead0d` |
| Bowmaster | 5 | 모자 (Cap) | 아케인셰이드 아처햇 | `b72cc1ef4d2d4f64866677050e0a9fb9` |
| Bowmaster | 5 | 한벌옷 (Longcoat) | 아케인셰이드 아처슈트 | `e93e89b82c274f0694102271585934f7` |
| Bowmaster | 5 | 장갑 (Glove) | 아케인셰이드 나이트글러브 | `dfd4da8447cd45219f57ba91d3ad642d` |
| Bowmaster | 5 | 신발 (Shoes) | 아케인셰이드 나이트슈즈 | `ce9c38b11f72410483fed1cbbe148480` |
| Bowmaster | 5 | 망토 (Cape) | 아케인셰이드 나이트케이프 | `54c1be1e29f94560859afde688d5bbae` |
| Bowmaster | 5 | 두손 무기 (TwoHandedWeapon) | 아케인셰이드 보우 | `f0c8f18069be499f92736cbf5acb7eaa` |
| Viper | 1 | 모자 (Cap) | 브라운 피레타햇 | `324bc734a43c423191a399f3925062a1` |
| Viper | 1 | 한벌옷 (Longcoat) | 브라운 툴크갤리 | `3d26c09ea6284d6c8e1fe625a9e48721` |
| Viper | 1 | 신발 (Shoes) | 브라운 레더크라그 | `0cc27439d15a4e69be1d041e0803ccb8` |
| Viper | 1 | 두손 무기 (TwoHandedWeapon) | 스틸 너클 | `9c03a78d2caf49a982a1d4b1d924cee4` |
| Viper | 2 | 모자 (Cap) | 다크 피레타햇 | `bb9dc197b65648d1b40af3388234a84c` |
| Viper | 2 | 한벌옷 (Longcoat) | 레드 비스카 | `e80bfac34d494e42a87d568db43d5fba` |
| Viper | 2 | 장갑 (Glove) | 브라운 레더 아머글로브 | `b1e8c6a49eb8452d98f95009739b73fd` |
| Viper | 2 | 신발 (Shoes) | 블랙 바실즈 부츠 | `16bd622735784efb944bfe4a6ada1487` |
| Viper | 2 | 두손 무기 (TwoHandedWeapon) | 메이플 스톰 핑거 | `1c703981fc754042871cc8abb589d0ff` |
| Viper | 3 | 모자 (Cap) | 블랙 파이렛츠 반다나 | `65e4777b6e50484ba228fed7aa9096aa` |
| Viper | 3 | 한벌옷 (Longcoat) | 블랙 로얄바롱 | `1c73a794497942cabb9848a951aacfbe` |
| Viper | 3 | 장갑 (Glove) | 블랙 스켈듀크 | `a61dd22c78e94536bd37201e7e31baf4` |
| Viper | 3 | 신발 (Shoes) | 블랙 듀크바킨 슈즈 | `11877303850542e18d33bb4fbcc374bb` |
| Viper | 3 | 망토 (Cape) | 빨간색 모험가의 망토 | `724a9e818ddd41ba9825b6fcce789521` |
| Viper | 3 | 두손 무기 (TwoHandedWeapon) | 레드 너클 | `b11507a9c7864a3e822d88c5386049b8` |
| Viper | 4 | 모자 (Cap) | 하이네스 원더러햇 | `57569b6988fe4acead1277299cb6af84` |
| Viper | 4 | 상의 (Coat) | 이글아이 원더러코트 | `7a925a2880e747228144c93ee0a3d92d` |
| Viper | 4 | 하의 (Pants) | 트릭스터 원더러팬츠 | `983e931f4ce54ad88a99d8664eb82b12` |
| Viper | 4 | 장갑 (Glove) | 앱솔랩스 나이트글러브 | `ad76497733c74828b4e6aaeff251c35d` |
| Viper | 4 | 신발 (Shoes) | 앱솔랩스 나이트슈즈 | `9885f9d8ee00456e842d0b19b1caae1f` |
| Viper | 4 | 망토 (Cape) | 앱솔랩스 나이트케이프 | `1cd3d12f6c184166a97ebbdd74bce6a1` |
| Viper | 4 | 두손 무기 (TwoHandedWeapon) | 파프니르 펜리르탈론 | `3fbc7c62d4d5453bb5a8dca5d8671e0e` |
| Viper | 5 | 모자 (Cap) | 아케인셰이드 파이렛햇 | `db33b98e092046af81d94dfdefde203a` |
| Viper | 5 | 한벌옷 (Longcoat) | 아케인셰이드 파이렛슈트 | `a0289186949a4e8389b540612d05399a` |
| Viper | 5 | 장갑 (Glove) | 아케인셰이드 나이트글러브 | `dfd4da8447cd45219f57ba91d3ad642d` |
| Viper | 5 | 신발 (Shoes) | 아케인셰이드 나이트슈즈 | `ce9c38b11f72410483fed1cbbe148480` |
| Viper | 5 | 망토 (Cape) | 아케인셰이드 나이트케이프 | `54c1be1e29f94560859afde688d5bbae` |
| Viper | 5 | 두손 무기 (TwoHandedWeapon) | 아케인셰이드 클로 | `a5602ea49007472a96e4887cb0dfda9e` |

선정 메모:
- 레벨대는 메이플 원작 기준 추정이다. 1~3차 일반 장비는 MSW 리소스에 착용 레벨 정보가 없어 원작 기억과 이름으로 골랐다(예: 메탈 코이프 10제, 그륜힐 30제, 블러드 플라티나·다크 허스크·다크 카젠부츠 70제 — 앞의 셋은 프로젝트 `EquipTable.csv`의 `ReqLevel`과도 일치).
- 4차 장갑·신발·망토, 5차 장갑·신발·망토는 MSW에 **나이트(전사)용만 있어 모든 직업이 같은 것**을 쓴다(아케인셰이드 메이지·시프·아처·파이렛 신발/장갑/망토, 앱솔랩스 직업별 장갑/신발/망토는 검색 결과 없음). 사용자 확인 후 바꿀 후보.
- 4차 무기 중 히어로만 150제 파프니르 두손검이 MSW에 없어 **160제 앱솔랩스 브로드세이버**로 대체했다.
- 같은 이름 RUID가 2개인 아이템(예: 하이네스 워리어헬름 `de9c87ef…` / `acee231e…`)은 검색 첫 결과를 골랐다. 색이 다르면 다른 쪽으로 바꿔도 된다.
- 비숍 1차·나이트로드 3차 망토 색 등은 직업색(비숍 파랑·흰색, 나이트로드 검정, 보우마스터 초록, 바이퍼 갈색→검정·빨강)에 맞췄다.

### 4-3. `PlayerRoster`에 추가할 함수 (2-C1)

```lua
-- LoadTables 안에 추가: CharOutfitTable → self._T.outfits[job][tier] = { Cap = "ruid", ... }
method table OutfitFor(string jobId, integer tier)
	-- 그 직업·차수의 외형 세트 { Cap = ruid, ..., family = "T"/"O" }. 실행 공간 지정 없음(클라이언트 미리보기도 사용).

@ExecSpace("ServerOnly")
method void ApplyOutfit(Entity target, string jobId, integer tier)
	-- 대상 아바타에 차수 세트를 입힌다. 아래 9개 부위를 모두 설정(세트에 없으면 "")하고 몸·머리·얼굴은 건드리지 않는다.
	-- 부위: Cap, Coat, Pants, Longcoat, Glove, Shoes, Cape, OneHandedWeapon, TwoHandedWeapon (+ SubWeapon은 항상 "")
	-- cm:SetEquip(MapleAvatarItemCategory.<부위>, ruid) 사용 (TowerRunManager.ApplyCostume과 같은 서버 호출 방식)
	-- target:GetComponent("script.TowerPlayer")가 있으면 WeaponFamily = OutfitFor(...).family
	-- 로그: [VRF] outfit job=Hero tier=2 target=<이름> n=<입힌 부위 수>

@ExecSpace("ServerOnly")
method void ApplyOutfitForJob(Entity target, string jobId)
	-- 저장된 현재 차수로 ApplyOutfit (잠긴 직업은 1차).
```

**로비 자동 적용**: `OnUpdate`의 1초 타이머(`FixParty` 호출 바로 뒤)에서 `RefreshLobbyOutfit()`을 부른다.

```
RefreshLobbyOutfit():
  로비가 아니면 self._T.lobbyOutfitKey = "" 로 지우고 끝 (다시 로비에 오면 새로 입힘)
  eng = PlayerEngrave, eng.SkillLoaded 아니면 끝
  key = eng.MainJob .. ":" .. 차수
  key ~= self._T.lobbyOutfitKey 이면 ApplyOutfitForJob(self.Entity, eng.MainJob), self._T.lobbyOutfitKey = key
```

→ 전직(차수 변경)·편성 1번 변경·탑에서 로비 복귀 때 1초 안에 로비 외형이 바뀐다.

### 4-4. 탑 연결 (2-C2, `Tower/TowerRunManager.mlua`)

- `ApplyCostume()` 본문 맨 앞에 추가:
  ```lua
  local roster = isvalid(player) and player:GetComponent("script.PlayerRoster") or nil
  if roster ~= nil and roster.RosterLoaded then
  	roster:ApplyOutfitForJob(player, self._T.mainJob or "Hero")
  	self:CopyCostumeToCompanion()
  	return
  end
  ```
  (아래 기존 장비 외형 코드는 로스터가 없을 때의 예비로 남긴다. `JobWeapons`·`EquipRow` 외형은 이 경로에서 쓰이지 않는다.)
- `CopyCostumeToCompanion()` 본문을 다음으로 바꾼다:
  ```lua
  local roster = ... (위와 같이 메인 플레이어의 PlayerRoster)
  for i = 1, 2 do
  	local comp = self:Mate(i)
  	if isvalid(comp) then
  		if roster ~= nil and roster.RosterLoaded then roster:ApplyOutfitForJob(comp, self._T.mateJobs[i] or "")
  		else self:CopyCostumeTo(comp) end
  	end
  end
  ```
- 구성원 모델(`Models/Party/Tower*.model`)은 이미 `UseCustomEquipOnly = true`이고 몸·머리·얼굴이 같아서 모델 수정은 필요 없다. 스폰 직후 `SpawnCompanion`이 `CopyCostumeToCompanion`을 부르므로 새로 생긴 구성원도 바로 입는다.
- `ApplyCostume`은 장비 강화·해금 때도 불리지만 결과가 같으므로 문제없다.

---

## 5. 서버 요청·결과 처리 (`CharacterUILogic`)

### 5-1. 흐름

1. 클라이언트: `local roster = _UserService.LocalPlayer:GetComponent("script.PlayerRoster")`.
2. 버튼 → 요청 함수 호출(서버 RPC): `roster:RequestLevelTo(job, target)`, `roster:RequestPromote(job, useCommon)`, `roster:RequestUnlockWithCommon(job)`, `roster:RequestSetParty(j1, j2, j3)`. 보낸 직후 `self._T.pendingSeq = roster.RosterSeq`, `self._T.pendingAt = 경과시간`, 실행 버튼 비활성.
3. `OnUpdate`(열려 있을 때만): `roster.RosterSeq ~= self._T.lastSeq`이면 결과 처리(5-2) 후 `Refresh()`. 또 `CharData / JobCards / BookS / BookM / BookL / CommonCards / Diamonds`와 PlayerEngrave `MainJob/CompJob/Comp2Job`을 이어 붙인 문자열이 이전과 다르면 `Refresh()`(다른 경로로 바뀐 값 반영).
4. `Refresh()`: `roster:ParseCharData(roster.CharData)`, `roster:ParseJobCards(roster.JobCards)`로 목록·상세를 다시 그린다. 능력치는 `roster:CharStat(job, level, tier)`, 미리보기는 `roster:PlanBooksFor(level, exp, tier, BookS, BookM, BookL, target)`, 차수 이름은 `roster:TierName`, 관문은 `roster:CapLevel(tier)`, 필요 카드는 `roster:PromoteCards(tier+1)` — 모두 실행 공간 지정 없는 함수라 클라이언트에서 그대로 쓴다.
5. 로비를 벗어나면 닫는다(`GrowthUILogic.OnUpdate`와 같은 방식). 닫기 X = `Close()`.

버튼 연결은 `GrowthUILogic.Bind`처럼 클릭 핸들러 안에서 `entity.EnabledInHierarchy`와 `ButtonComponent.Enable`을 검사한다(비활성 버튼에 `SendEvent(ButtonClickEvent())`를 보내도 아무 일도 없어야 함). 클릭음 `_UISoundLogic:PlayClick()`.

### 5-2. 결과 문구 (`RosterMsg` → `Msg`)

`RosterResult`를 `:`로 나눈 값을 `f[1]`(동작), `f[2]`(직업) … 로 쓴다. 성공은 유리색, 오류는 부족색.

| 코드 | 문구 | 비고 |
|---|---|---|
| `ok_level` | `Lv.{f3} → Lv.{f4} 달성!` | `f5`가 있고 `f4 < f5`면 `경험의 서가 부족해 Lv.{f4}까지 올렸습니다` |
| `ok_promote` | `{새 차수 이름}(으)로 전직했습니다!` | 새 차수 = `f3`. 공용 사용(`f5` > 0)이면 뒤에 ` (공용 카드 {f5}장 사용)` |
| `ok_unlock` | `{4차 이름} 해금!` | |
| `ok_party` | `편성을 저장했습니다` | |
| `err_notready` | `캐릭터 정보를 불러오는 중입니다` | |
| `err_lobby` | `로비에서만 할 수 있습니다` | |
| `err_job` | `알 수 없는 직업입니다` | |
| `err_locked` | `잠긴 직업입니다` | |
| `err_unlocked` | `이미 해금된 직업입니다` | |
| `err_cap` | `관문 레벨입니다. 전직하면 계속 성장합니다` | |
| `err_max` | `최고 레벨·최고 차수입니다` | |
| `err_books` | `경험의 서가 부족합니다` | |
| `err_level` | 동작 `promote`: `전직 레벨(Lv.{관문})에 도달해야 합니다` / 동작 `level`: `이미 목표 레벨입니다` | |
| `err_cards` | 동작 `promote`: `전직 카드가 부족합니다` / 동작 `unlock`: `공용 전직 카드가 부족합니다` | |
| `err_party` | `편성할 수 없습니다 (중복 또는 잠긴 직업)` | |

- 결과가 지금 선택한 직업과 다른 직업이면(`f2 ~= 선택 직업`) 문구만 띄우고 선택은 바꾸지 않는다.
- 결과 처리 후 `[VRF] charui result <코드> <RosterResult>` 로그.

필수 `[VRF]` 로그 (테스트가 검색함): `OnBeginPlay` → `[VRF] CharacterUILogic begin`, `Open()` → `[VRF] charui open`, 직업 선택 → `[VRF] charui select <job>`, 요청 전송 → `[VRF] charui request <동작> <인자>`, 결과 → `[VRF] charui result …`, `Close()` → `[VRF] charui close`. `PlayerRoster.LoadTables`의 기존 로그는 `[VRF] roster tables jobs=5 outfits=143`(외형 행 수 추가)으로 바꾼다.

### 5-3. 버튼 동작

| 버튼 | 동작 |
|---|---|
| `Job{n}` | 선택 직업 변경 → `Refresh()` |
| `BtnLv1` | `RequestLevelTo(job, level + 1)` |
| `BtnLvMax` | `RequestLevelTo(job, CapLevel(tier))` |
| `BtnPromote` | `RequestPromote(job, false)` |
| `BtnPromoteCommon` | `RequestPromote(job, true)` |
| `BtnUnlock` | `RequestUnlockWithCommon(job)` |
| `Party{k}` | `party = {MainJob, CompJob, Comp2Job}`; `party[k] == job`이면 무시; 다른 칸에 `job`이 있으면 그 칸 = `party[k]`(서로 바꿈); `party[k] = job`; `RequestSetParty(party[1], party[2], party[3])` |
| `Close` | 닫기 |

활성 조건(2-4 표와 같음):
- `BtnLv1`: 해금 and `level < cap` and `PlanBooksFor(..., level+1)`의 `s+m+l > 0`.
- `BtnLvMax`: 해금 and `level < cap` and `PlanBooksFor(..., cap)`의 `s+m+l > 0`. 글자 `최대 레벨업 (Lv.{도달 레벨})`.
- `BtnPromote`: 해금 and `tier < 5` and `level == cap` and 전용 ≥ 필요.
- `BtnPromoteCommon`: 해금 and `tier < 5` and `level == cap` and 전용 < 필요 and 전용 + 공용 ≥ 필요.
- `BtnUnlock`: 잠김 and 공용 ≥ 1.

미리보기 문구(`PreviewOne`/`PreviewMax`): `plan = PlanBooksFor(...)`, `넘침 = plan.s×100 + plan.m×500 + plan.l×2000 - plan.gain` (책 경험치는 `CharConfig` 값으로 계산). 책이 모자라 `plan.newLevel < 목표`면 `… → Lv.{newLevel} (책 부족)`, `s+m+l == 0`이면 `경험의 서가 없습니다`.

### 5-4. 바꾸는 기존 파일 (2-B3)

- `ui/LobbyUI`: `Root/BtnAchieve/Label`의 `Text`를 `캐릭터`로만 바꾼다(UIBuilder `patchComponent`). 엔티티 이름·UUID·아이콘·크기 그대로.
- `Lobby/LobbyUILogic.mlua`: `self:Bind(self.btnAchieve, function() self:ShowNotice("업적은 준비 중입니다") end)` → `self:Bind(self.btnAchieve, function() _CharacterUILogic:Open() end)`. 속성 위 주석에 "업적 자리 → 캐릭터 화면(2단계)" 한 줄.
- `Skill/HeroSkillUILogic.mlua` `OnFormation`: 마지막 줄 `eng:RequestSetFormation(party[1], party[2], party[3])`를
  ```lua
  local roster = _UserService.LocalPlayer:GetComponent("script.PlayerRoster")
  if roster ~= nil then roster:RequestSetParty(party[1], party[2], party[3]) else eng:RequestSetFormation(party[1], party[2], party[3]) end
  ```
  로 바꾼다.

---

## 6. 테스트 케이스

준비: 플레이 후 로비. 서버(`server_main`)에서
```lua
local p = nil
for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
local r = p:GetComponent("script.PlayerRoster")
r:DebugReset()
```
클라이언트(`client`) 버튼 클릭: `_EntityService:GetEntityByPath("/ui/CharacterUI/Root/Detail/BtnLvMax"):SendEvent(ButtonClickEvent())`. 결과는 `[VRF]` 로그와 Sync 값(`r:DebugDump()`)으로 확인. 클라이언트 첫 스크립트는 UI 준비 전일 수 있으니 한 번 더 실행.

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T1 진입 | 클라이언트: `/ui/LobbyUI/Root/BtnAchieve`에 클릭 이벤트 | `CharacterUI/Root` enable, 로그 `[VRF] charui open`, 로비 버튼 글자 `캐릭터` |
| T2 목록 | 열린 직후 | Job1~3 히어로·비숍·나이트로드 `검사 / 매지션 / 로그`, `Lv.1 · 1차`, 카드 `3장 / 1장 / 0장`; Job4·5 덮개+자물쇠, `잠김`; PartyMark 1·2·3이 편성 직업에 표시 |
| T3 상세 | `Job1` 클릭 | `TierName` `검사 (1차)`, `LevelText` `Lv.1 / 관문 Lv.7`, `StatAtk 10 / StatHp 120 / StatDef 1`, `Book1~3` `50 / 20 / 10권` |
| T4 미리보기 | T3 상태 | `PreviewOne` `Lv.1 → 2 · 소 2 · 중 0 · 대 0`(R=120 → 소 1=100, 남은 20 → 소 1 추가), `PreviewMax` `Lv.1 → 7 · 소 4 · 중 2 · 대 0` (1390 = 중 2 + 소 3 + 남은 90 → 소 1) |
| T5 1레벨업 | `BtnLv1` 클릭 | `[VRF] roster reply ok_level level:Hero:1:2:2`, `Msg` `Lv.1 → Lv.2 달성!`, 소 48 |
| T6 최대 | `BtnLvMax` 클릭 | `ok_level level:Hero:2:7:7`, Lv7, 레벨업 버튼 2개 비활성, `PreviewOne` `관문 도달 — 전직이 필요합니다` |
| T7 비활성 차단 | T6 직후 `BtnLv1`에 클릭 이벤트 | 새 `roster reply` 로그 없음 (`RosterSeq` 그대로) |
| T8 전직 | `BtnPromote` 활성 확인 후 클릭 | `ok_promote promote:Hero:2:1:0`, `TierName` `파이터 (2차)`, 카드 2장, `StatAtk 24 / StatHp 211 / StatDef 9`, 1초 안에 `[VRF] outfit job=Hero tier=2` (로비 외형) |
| T9 공용 대체 | 서버 `r:DebugSetChar("Bishop",1,13,2)`, `r:DebugSetCards("Bishop",1)` → 클라이언트 `Job2` 선택 | `BtnPromote` 비활성, `BtnPromoteCommon` 활성 `공용 카드로 전직 (1장)` |
| T10 | `BtnPromoteCommon` 클릭 | `ok_promote promote:Bishop:3:1:1`, `Msg`에 `(공용 카드 1장 사용)`, 공용 2→1, `Currency` 갱신 |
| T11 둘 다 부족 | 서버 `r:DebugSetChar("NightLord",1,7,1)`, `r:DebugSetCurrency(3000,50,20,10,0)` → `Job3` 선택 | `BtnPromote`·`BtnPromoteCommon` 모두 비활성, `PromoteInfo` 부족색 |
| T12 해금 | 서버 `r:DebugSetCurrency(3000,50,20,10,1)` → `Job5`(바이퍼) 선택 → `BtnUnlock` 클릭 | `ok_unlock unlock:Viper`, 덮개 사라짐, `해적 / Lv.1 · 1차`, 공용 0, `BtnUnlock` 숨김 |
| T13 잠긴 직업 편성 차단 | `Job4`(보우마스터, 잠김) 선택 | `Party1~3` 모두 비활성, 클릭 이벤트 보내도 요청 로그 없음 |
| T14 편성 | 편성이 히어로·비숍·나이트로드인 상태에서 `Job5`(바이퍼) 선택 → `Party1` 클릭 | `ok_party party:Viper,Bishop,NightLord`, `PlayerEngrave.MainJob == "Viper"`, Job5에 PartyMark `1`, 히어로의 PartyMark 사라짐, `PartyText` 갱신, 1초 안에 `[VRF] outfit job=Viper tier=1` (바이퍼가 이미 2·3번에 있었다면 원래 1번 직업과 자리를 서로 바꾼다) |
| T15 책 없음 | 서버 `r:DebugSetCurrency(3000,0,0,0,0)` → `Job1` 선택(관문 전 상태로 `r:DebugSetChar("Hero",1,8,2)`) | 레벨업 버튼 2개 비활성, `PreviewOne` `경험의 서가 없습니다`, `Book*/Count` 부족색 |
| T16 스킬 탭 편성 | 성장창 5번 탭에서 잠긴 직업(보우마스터)을 편성 버튼으로 넣기 시도 | `[VRF] roster reply err_party`, `MainJob` 변화 없음 |
| T17 외형 표 | 서버 `r:OutfitFor("Bowmaster", 5)` 내용 로그 | `Cap=b72cc1ef…`, `Longcoat=e93e89b8…`, `TwoHandedWeapon=f0c8f180…`, `family=T`, 표 전체 로드 로그 `[VRF] roster tables … outfits=143` |
| T18 로비 외형 | 서버 `r:DebugSetChar("Hero",1,25,5)` 후 히어로를 1번으로 편성 | 1초 안에 `[VRF] outfit job=Hero tier=5`, 스크린샷에 아케인셰이드 나이트 세트 + 두손검. 서버 로그의 입힌 부위 수 `n=6` 확인 |
| T19 탑 구성원 | 편성 히어로(5차)·비숍(3차)·나이트로드(1차)로 탑 입장 후 판 시작 | `[VRF] outfit` 로그 3개(job·tier 각각), 스크린샷에서 세 명이 서로 다른 세트 |
| T20 스킬 동작 무기 | 탑에서 나이트로드·바이퍼·보우마스터를 메인으로 각각 판 시작, 기본 공격·스킬 시전 | 스킬 프레임 동안 무기가 사라지지 않음. 사라지면 그 직업의 `CharJobTable.WeaponFamily`를 `T`(또는 `O`)로 바꿔 재확인하고 보고 |
| T21 미리보기 | 캐릭터 화면에서 Job1→Job2→Job5 선택 | `Avatar`가 각 직업 현재 차수 세트로 바뀜(스크린샷). 안 되면 `Avatar` 끄고 보고 |
| T22 재접속 | stop → play → 캐릭터 화면 | T8~T14 결과 유지, 로비 외형 = 1번 직업 차수 세트 |

완료 조건: 메이커 빌드 오류 0, LSP 진단 오류 0, 콘솔 `LEA-` 없음, T1~T22 통과(T21은 생략 가능, 사유 보고), `git diff --stat`으로 이 문서가 허용한 파일만 바뀌었는지 확인.

---

## 7. 이후 단계로 넘기는 것

- 3단계: `BtnGacha`를 캐릭터 뽑기로 전환, 다이아 지급. 캐릭터 화면 `Currency`는 이미 다이아를 보여 준다.
- 4단계: 칸 6개 스킬 표시(캐릭터 화면 상세에 스킬 칸 한 줄 추가 자리 = `StatNote` 아래 또는 `PreviewBox` 아래), `PartyRunData` 잠긴 직업 방어(1단계 보완 Q8).
- 5단계: 장비 탭(캐릭터 화면에 장비 영역 추가). 장비 외형은 계속 쓰지 않는다.
- 빠진 리소스: 캐릭터 메뉴 아이콘, 경험의 서 소·중·대 아이콘, 전용·공용 전직 카드 아이콘. 사용자 확인 후 `msw-search` 또는 `msw-painter`로 준비.

---

## 2단계 보완 (3단계 조사에서 결정)

> 작성: 자료 조사 에이전트(3단계), 2026-11-22
> 근거: `Phase-2-Test-Report.md`의 버그 B1~B3, 관찰 1~6, 질문 Q1~Q8. 이 절의 코드 작업은 `Phase-3-Gacha.md`의 작업 단위 **3-A**다.
> 이 절이 정본이다. 위 본문(2~6절)은 그대로 두고, 다른 부분만 여기서 바꾼다.

### 결정 요약

| 번호 | 내용 | 결정 | 작업 |
|---|---|---|---|
| B1 / Q1 | 5차 이름 `히어로 (5차) (5차)` | **표 `Name5`(`히어로 (5차)`)는 그대로 둔다.** UI가 차수를 덧붙이는 두 곳(`RefreshDetail`의 `tierLabel`, `PromoteTitle`)에서 5차일 때만 덧붙이지 않는다. 목록 카드·결과 문구(`히어로 (5차)로 전직했습니다!`)는 이미 `TierName`만 써서 정상 | a |
| B2 / Q2 | 잠긴 직업 미리보기가 어두워지지 않음 | `AvatarGUIRendererComponent.Color`는 화면에 반영되지 않으므로 쓰지 않는다. **미리보기 칸 위에 반투명 어두운 덮개 엔티티**를 두고 잠긴 직업일 때만 켠다(유물 화면 `Cell_n/Dim`과 같은 값). `Color 0.45` 코드는 지운다 | b |
| B3 / Q6 | 편성 띠 두 줄 넘침 | 편성 띠는 **차수 표기 없이 직업 이름(4차 이름, `CharJobTable.Name4`)만** 쓴다: `편성  1 히어로 · 2 비숍 · 3 나이트로드`. 안전장치로 `BestFit`(최소 18)도 켠다 | c |
| 관찰 5 | 위쪽 띠 뒤로 로비 HUD가 비침 | 띠 영역에만 `panel_outer`를 한 겹 더 깐다(PNG 알파 약 96% → 두 겹이면 약 99.8%). 엔티티 Color는 이미 알파 1이라 값으로는 더 올릴 수 없다 | d |
| Q3 | 바이퍼 5차 무기 `a5602ea49007472a96e4887cb0dfda9e` | 리소스 API `getResource` 결과 이름이 **`아케인셰이드 클로`(도적 아대)** 로 너클이 아니다. MSW 리소스에 `아케인셰이드 너클`은 없다(한국어 `아케인셰이드 너클`·영어 `Arcane Umbra Knuckle` 검색 모두 없음). **160제 너클 `앱솔랩스 블로우너클` `573d7ad7d5c04ad681be5014e13d1f34`**(`getResource`로 `avataritem`·`twohandweapon` 확인)로 바꾼다. 4차 `파프니르 펜리르탈론`(150제 너클)보다 상위라 차수 순서도 맞다 | e |
| Q4 | 미리보기 두손 무기 대기 자세 | `AvatarGUIRendererComponent`에는 동작 지정 API가 없다(.d.mlua 확인). 대신 엔티티에 보내는 이벤트 `BodyActionTypeChangeEvent(needResetAction, actionType)`(동작 이름 문자열)와 `UIAvatarActionControllEvent(actionSpeed, bodyActionName, selectFrame)`가 있다. **한 번만 시도**하고 안 되면 보류한다(아래 f) | f (시도) |
| Q5 | T4 기대 문구 정정 + 조사 `2은` 어색함 | 넘침 꼬리 문구를 **숫자 뒤에 조사가 오지 않는 형태** `  (사라지는 경험치: N)`으로 바꾼다. 6절 T4의 `PreviewMax` 기대값은 `최대  Lv.1 → 7  ·  소 4 · 중 2 · 대 0  (사라지는 경험치: 10)`으로 정정한다 | g |
| 관찰 6 | 스킬 탭 편성 실패 시 안내 없음 | **6단계로 넘긴다.** 스킬 탭 자체가 6단계에서 사라지고 캐릭터 화면이 편성 정본이다. 지금 고치려면 성장창 UI에 안내 줄을 새로 만들어야 해 작은 개선이 아니다 | 없음 |
| Q7 | 메인 바이퍼로 판 시작 시 HUD·전투가 나이트로드 | **4단계 요구사항으로 기록**(`02-Roadmap.md` 4단계 "보류 항목"에 추가함) | 4단계 |
| Q8 | `Global/DefaultPlayer` 모델 변경 | 1단계 1-B에서 **`script.PlayerRoster` 컴포넌트를 붙인 정상 변경**이다. ModelBuilder로 읽어 컴포넌트 7개(`PlayerHit, TowerPlayer, PlayerEngrave, HeroCombat, BishopCombat, JobCombat, PlayerRoster`) 확인. 2단계 허용 파일 확인 때 1단계 변경으로 본다 | 없음 |
| 관찰 4 | 잠긴 직업 상세에 `Lv.1 / 관문 Lv.7`, `경험치 0 / 120` 표시 | 그대로 둔다(덮개와 `PromoteTitle` 안내로 잠김이 충분히 보임) | 없음 |

### 구현 작업 (3-A)

**(a) B1 — `Character/CharacterUILogic.mlua` `RefreshDetail` 두 곳**

```lua
-- 391행 부근: tierLabel
local tierLabel = tierName .. " (" .. tostring(tier) .. "차)"
if tier >= maxTier then tierLabel = tierName end   -- Name5에 이미 "(5차)"가 있음

-- 487행 부근: PromoteTitle
local nextName = roster:TierName(job, nextTier)
local nextLabel = nextName .. " (" .. tostring(nextTier) .. "차)"
if nextTier >= maxTier then nextLabel = nextName end
self:SetText("Detail/PromoteTitle", "전직 — 다음 차수: " .. nextLabel, nil)
```

**(b) B2 — 미리보기 덮개** (`ui/CharacterUI`는 UIBuilder로만, 경로는 node 안에서 `'ui/'+'CharacterUI'+'.'+'ui'`)

| 경로 | 종류 | pos (부모 기준) | 크기 | 값 |
|---|---|---|---|---|
| `Root/Detail/Avatar/Dim` | sprite | 0,-35 | 380×460 | card_normal `a88cfe62e8e64c39bf14d682a77ef4a0`, Sliced(Type 1), Color `(0.078, 0.102, 0.133, 0.55)`(유물 `Cell_n/Dim`과 같음), PreserveSprite 0, RaycastTarget false, **enable false** |
| `Root/Detail/Avatar/Dim/Icon` | sprite | 0,40 | 64×64 | 자물쇠 `f190dca8fa0c4209b38eee869bd5e4ea`, Simple, RaycastTarget false |

- `Avatar`(Detail 기준 -450,205, 230×300)의 **자식**으로 두는 이유: 빌더는 새 엔티티를 형제 맨 뒤에 붙인다. `Avatar`의 자식이면 아바타 바로 위·`PreviewName`(뒤 형제) 아래에 그려진다. 덮개(Detail 기준 -450,170, 380×460)는 `PreviewBox`와 정확히 겹친다.
- `ShowPreview`: `ar.Color` 대입 줄을 지우고 `self:SetShown("Detail/Avatar/Dim", not unlocked)`를 넣는다. 캐시 키(`previewKey`)가 같아 함수가 일찍 끝나도 덮개가 맞도록 **캐시 검사 앞**에 둔다.
- `Avatar`가 enable false(미리보기 실패)면 덮개도 함께 안 보이며 문제없다.
- 그리는 순서가 예상과 다르면(아바타가 덮개 위) 덮개를 `Root/Detail`의 마지막 자식 `Root/Detail/PreviewDim`(pos -450,170)으로 옮기고 `PreviewName`이 덮여도 되는지 스크린샷으로 판단한다.

**(c) B3 — 편성 띠** (`Refresh` 337행 부근)

```lua
for k = 1, 3 do
	local pj = self:JobById(party[k])
	names[k] = tostring(k) .. " " .. ((pj ~= nil and pj.name4) or party[k])
end
```
- `ui/CharacterUI` `Root/PartyText`의 `TextGUIRendererComponent`에 BestFit 켜기, 최소 크기 18 (필드 이름은 `msw-ui-system` `component-api.md`로 확인 후 `patchComponent`).

**(d) 관찰 5 — 위쪽 띠 배경 한 겹 추가**

| 경로 | 종류 | pos (Bg 기준) | 크기 | 값 |
|---|---|---|---|---|
| `Root/Bg/HeadBg` | sprite | 0,451 | 1920×178 | panel_outer `ad1157dddb334c91b1b58c1073e060e1`, Sliced, Color 흰색 알파 1, PreserveSprite 0, RaycastTarget false |

- 띠 높이 근거: `Root/Frame`은 렌더러 LocalPosition y -85.5, LocalScale y 0.8287이라 밝은 본문 윗변이 y≈362다. 띠는 362~540(높이 178, 중심 451).
- `Bg`의 자식이라 `Frame`·`Title`·편성 버튼보다 아래에 그려진다. 스크린샷에서 `감자`·`레벨 2`가 더 이상 비치지 않는지 확인한다.
- 같은 비침이 성장창(`ui/GrowthUI`)에도 있을 수 있으나 이번 범위가 아니다(보고만).

**(e) Q3 — `RootDesk/MyDesk/Data/CharOutfitTable.csv` 마지막 행 한 줄**

```
바꾸기 전: Viper,5,TwoHandedWeapon,a5602ea49007472a96e4887cb0dfda9e,아케인셰이드 클로
바꾼 뒤:   Viper,5,TwoHandedWeapon,573d7ad7d5c04ad681be5014e13d1f34,앱솔랩스 블로우너클
```
- 행 수(143)는 그대로. UTF-8 BOM 유지. 위 4-2 선정 표의 해당 줄도 이 값으로 읽는다.
- 확인: 서버 `r:OutfitFor("Viper",5).TwoHandedWeapon == "573d7ad7d5c04ad681be5014e13d1f34"`, 바이퍼 5차 미리보기 스크린샷.

**(f) Q4 — 두손 무기 대기 자세 (한 번만 시도, 실패 시 보류)**

- `CharacterUILogic`에 `property string PreviewStand2Jobs = "Hero"`(쉼표 구분)를 두고, `ShowPreview`의 `SetEquip` 직후 그 직업이면 `av:SendEvent(BodyActionTypeChangeEvent(true, "stand2"))`, 아니면 `"stand1"`을 보낸다(두손검만 `stand2`. 활·아대·너클은 원작에서도 `stand1`).
- 화면 변화가 없으면 `av:SendEvent(UIAvatarActionControllEvent(1, "stand2", -1))`로 한 번 더 시도한다.
- 둘 다 안 되거나 `LEA-` 오류가 나면 코드를 지우고 **보류**로 보고한다(지금 모습도 크게 어색하지 않다는 테스트 관찰).

**(g) Q5 — 넘침 꼬리 문구** (`PlanLine`)

```lua
if over > 0 then line = line .. "  (사라지는 경험치: " .. self:Comma(over) .. ")" end
```

### 3-A 완료 조건

- 메이커 빌드 오류 0, 콘솔 `LEA-` 0.
- 서버 `r:DebugSetChar("Hero",1,25,5)` → 캐릭터 화면 히어로: `TierName`·`PreviewName` = `히어로 (5차)`(한 번). `r:DebugSetChar("NightLord",1,25,4)` → `PromoteTitle` = `전직 — 다음 차수: 나이트로드 (5차)`.
- 잠긴 직업(보우마스터) 선택 → `Detail/Avatar/Dim` enable true, 해금 직업 선택 → false. 스크린샷으로 어두워짐 확인.
- 편성을 5차 직업 3개로 둔 뒤 `PartyText` = `편성  1 나이트로드 · 2 보우마스터 · 3 바이퍼` 한 줄.
- T4 다시 확인: `PreviewMax` 끝에 `(사라지는 경험치: 10)`.
- 스크린샷: 위쪽 띠 뒤 로비 HUD 비침 없음, 바이퍼 5차 무기가 너클.
- (f)는 결과(적용/보류)만 보고.
