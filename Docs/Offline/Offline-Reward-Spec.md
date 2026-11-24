# 방치(오프라인) 보상 설계서

> 작성: 2026-11-24 · 상태: **설계 초안 (사용자 확정 전)** · 게임 코드 미수정
> 먼저 읽을 문서: `CLAUDE.md`, `AGENTS.md`, `Docs/Job-Redesign/01-Agent-Rules.md`, `Docs/Job-Redesign/00-Master-Spec.md`, `Docs/UI-Design-System.md`
> 구현 담당은 이 문서만 보고 작업할 수 있도록 썼다. §9의 "확정 필요" 항목은 구현 전에 메인 세션이 사용자에게 확인한다.

---

## 1. 한 줄 요약

마지막으로 받은 뒤 흐른 **서버 시각** 기준으로 최대 **12시간**까지 보상이 쌓이고, 시간당 양은 **클리어한 최고 층(`PlayerEngrave.ClearedFloor`)** 에 비례한다. 접속해 로비에 들어오면 **보상 팝업**이 뜨고, 로비 왼쪽 위 **상자 버튼**으로도 언제든 받는다. 화면 글자는 아이콘+수량+시간뿐이고 설명은 툴팁으로 돌린다.

---

## 2. 레퍼런스 조사와 차용

| 게임 | 쌓이는 최대 시간 | 양의 기준 | 받는 방식 | 가속 요소 | 출처 |
|---|---|---|---|---|---|
| AFK 아레나 | 12시간 (상자가 가득 차면 멈춤) | 현재 도달 캠페인 스테이지 | 캠페인 화면의 방치 상자를 눌러 수령 | **빠른 보상**: 하루 1회 무료, 이후 다이아 50·80·100 (하루 3회), 한 번에 방치 2시간 분량 즉시 지급 | [afk.guide 초보 가이드](https://afk.guide/afk-arena-beginner-guide/), [GamePress 일일 체크리스트](https://gamepress.gg/dailies-checklist-afk-arena?amp), [afk.guide 빠른 보상](https://afk.guide/?p=5151) |
| 승리의 여신: 니케 (전초기지 방어) | 24시간 | 캠페인 진행(5스테이지마다 방어 레벨 +1) | 전초기지 화면에서 수령 | **섬멸**: 하루 첫 회 무료, 이후 보석 소모, 2시간 분량 즉시 | [Prydwen 전초기지 가이드](https://www.prydwen.gg/nikke/guides/outpost), [West Games 계산기](https://west-games.com/nikke-outpost-defense-calculator/) |
| 버섯커 키우기 (Legend of Mushroom) | 커뮤니티 답변상 약 8시간 (비공식) | 진행 단계 | 접속 시 방치 보상 창 | 광고 1회 = 2시간 분량, 하루 3회 | [Gamertweak](https://gamertweak.com/how-to-get-lamps-in-legend-of-mushroom), [AppGamer Q&A](https://www.appgamer.com/legend-of-mushroom/is-there-a-max-time-amount-for-afk-rewards), [TheRiaGames](https://theriagames.com/?p=8040) |
| 메이플 키우기 (넥슨 방치형) | 공식 수치 확인 못 함 | (일반적으로) 최고 스테이지 | 접속 시 오프라인 보상 팝업 | 확인 못 함 | [앱스토어 소개](https://apps.apple.com/kr/app/%EB%A9%94%EC%9D%B4%ED%94%8C-%ED%82%A4%EC%9A%B0%EA%B8%B0/id6739616715) — "잘 때도 캐릭터는 성장 중" 콘셉트만 확인 |
| 리니지M류 MMORPG 휴식 보상 | 휴식 게이지(일정 상한) | 캐릭터 레벨 | 다음 사냥 시 경험치 추가 배율로 소모 | 없음 | 일반 지식(출처 링크 없음). **채택 안 함**: 판 안 경험치가 없어진 이 게임 구조(마스터 스펙 §3 판 안 레벨 삭제)와 맞지 않음 |

> 쿠키런 킹덤은 생산 건물 대기열 방식이라 "최고 층 비례 방치 보상"과는 결이 달라 비교에서 뺐다.
> 웹 자료는 공략·커뮤니티 기준이며 현재 게임 수치와 다를 수 있다.

### 채택한 것

| 차용 요소 | 출처 | 이 게임에 맞춘 형태 |
|---|---|---|
| 상한 12시간, 가득 차면 멈춤 | AFK 아레나 | `MaxHours = 12`. 넘친 시간은 버림 |
| 진행도 비례 시간당 보상, **수령 시점의 진행도** 적용 | AFK 아레나·니케 | 수령 순간의 `ClearedFloor`로 전체 시간을 계산 (층을 깬 뒤 받으면 이득 → 탑 등반 동기) |
| 접속 시 자동 팝업 + 상시 버튼 | 메이플 키우기·버섯커 키우기 / AFK 상자 | 로비 진입 시 1회 팝업, 로비 왼쪽 위 상자 버튼 |
| 하루 1회 무료 + 다이아 즉시 정산(2시간 분량) | AFK 빠른 보상·니케 섬멸 | **2단계(선택)**: "빠른 정산". MSW에는 광고가 없으므로 광고 대신 무료 1회 + 다이아 |
| 대기 시간 게이지와 가득 참 알림 | 공통 | 버튼 아래 작은 진행 막대 + 가득 차면 빨간 점 |

### 채택하지 않은 것
- 광고 시청 2배: MSW 월드에 광고 기능이 없다.
- 무작위 희귀 장비 드롭(AFK의 방치 장비): 이 게임 장비는 편성 칸 5부위 강화형이라 드롭 장비가 없다. 대신 2단계 후보로 "N시간마다 큐브 1개" 같은 확정 재료를 §4.4에 남긴다.

---

## 3. 현재 코드 조사 결과 (구현 근거)

| 항목 | 위치 | 내용 |
|---|---|---|
| 최고 층 | `RootDesk/MyDesk/Engrave/PlayerEngrave.mlua` `property integer ClearedFloor` (39행), `Loaded` | 클리어한 최고 층 수. 저장 키 `OrbisEngrave`. 다른 컴포넌트(`PlayerRoster`, `PlayerGear`, `PlayerPet`, `PlayerTalent`)가 **읽기만** 한다. 이 파일은 다른 세션 작업 중이라 **수정 금지**, 읽기만 |
| 층 규모 | `RootDesk/MyDesk/Data/FloorTable.csv` | 1~97층. `FloorsPerStage = 1` (층 하나가 한 판, 10스테이지) |
| 서버 시각 | `Environment/NativeScripts/Misc/DateTime.d.mlua` `DateTime.UtcNow`, `Elapsed`(밀리초) | 선례: `PlayerEngrave:EnergyNow()` = `math.floor(DateTime.UtcNow.Elapsed / 1000)` (ServerOnly). 에너지 미접속 충전이 이미 이 방식으로 동작한다. `_UtilLogic.ServerElapsedSeconds`는 월드 시작 기준 경과라 접속 간 비교에 쓰지 않는다 |
| 한국 날짜(하루 1회 초기화용) | `PlayerRoster:KstToUtcElapsed`, `CharConfig.csv` `KstOffsetHours = 9` | UTC 초 + 9시간 → 날짜 문자열로 일일 초기화 |
| 저장 방식 | `PlayerEngrave:LoadEnergy` / `ConsumeEntryEnergy` | `_DataStorageService:GetUserDataStorage(ProfileCode)` → `GetAndWait`/`SetAndWait`. 코드 `0` 성공, `1000002` 저장 없음. JSON은 `_HttpService:JSONEncode/JSONDecode` |
| 금색 메소 지급 | `RootDesk/MyDesk/Character/PlayerGear.mlua` `AddGoldMeso(n)` + `Flush()` | `AddGoldMeso`는 보유만 늘림, `Flush()`가 저장. `GearLoaded` 확인 필요 |
| 다이아 지급 | `RootDesk/MyDesk/Character/PlayerRoster.mlua` `GiveDiamonds(n)` | 내부에서 `SaveRoster()` |
| 경험의 서 지급 | `PlayerRoster:GiveBooks(s, m, l)` | 소 100 / 중 500 / 대 2000 경험치 (`CharConfig.csv` BookExpS/M/L). 1→30레벨 총 46,520 |
| 지급 선례 | `PlayerRoster:CheckLevelRewards` | 다른 컴포넌트를 `self.Entity:GetComponent("script.PlayerGear")` 로 얻어 지급하고 Seq/Data Sync 값으로 클라이언트 연출을 띄운다 |
| 플레이어 컴포넌트 부착 | `Global/DefaultPlayer.model` | `PlayerEngrave, PlayerRoster, PlayerGear, PlayerPet, PlayerTalent` 부착. 새 컴포넌트도 여기에 ModelBuilder로 추가 |
| 로비 UI | `ui/LobbyUI.ui` + `RootDesk/MyDesk/Lobby/LobbyUILogic.mlua` | `Root/AccountChip`(왼쪽 위 기준, (32,-28) 320×104), `Root/BtnSkillTest`(왼쪽 위 기준 (32,-146) 256×72, 메이커 플레이 전용), `Root/Wallet`(위 가운데 (124,-60) 480×64), `Root/EnergyHUD`(위 가운데 (-248,-60)). 재화 표시 `RefreshWallet()` |
| 화면 위 팝업 선례 | `RootDesk/MyDesk/Account/LevelUpUILogic.mlua` + `ui/LevelUpUI.ui` | 서버 `LevelUpSeq` 변화 감지 → 표시, 전체 화면 입력 차단 판. 그룹 순서: LevelUpUI 27, TooltipUI 28, ConfirmUI 17, GrowthUI 12, LobbyUI 2 |
| 툴팁 | `RootDesk/MyDesk/Tooltip/TooltipLogic.mlua` | `_TooltipLogic:ShowText(대상, 제목, 본문)`, `ShowItems`, `Register(대상, fn)` |
| 아이콘 | `ResultPopupLogic.mlua` | 금색 메소 `02a489cccff24a139a6c3582a5871f58`, 다이아 `81462aeb825b4660991cef05651271cf`. 경험의 서 소·중·대 아이콘은 `ui/CharacterUI.ui`의 `Book1~3/Icon`에 지정된 RUID를 UIBuilder로 읽어 재사용 |
| 상자 그림 | `output/animations/chest_open_glow/runtime_manifest.json` | 192×192, 10프레임. 닫힌 상자 = 1프레임 `35d6753b40ab48bf93f7c734b4634de3`, 열린 상자 = 10프레임 `09f299e5e7c54fd1adcd1925bde00a7a` |
| 공통 UI 리소스 | `output/ui/growth_neutral/manifest.json` | `panel_outer/light/inner`, `button_*`, `action_*`(초록 실행), `close_*`(64×64 X) 4상태 |
| 디버그 선례 | `PlayerRoster:DebugGrant`, `DebugSetBanner` | ServerOnly 메서드를 `maker_execute_script`(server_main)로 호출 |

---

## 4. 보상 공식

### 4.1 기호
- `B` = 수령 순간의 `PlayerEngrave.ClearedFloor` (0이면 보상 없음 — 1층 클리어 전에는 버튼 잠금)
- `T` = 쌓인 초 = `min(now - last, MaxHours × 3600)`, 음수면 0
- `H` = `T / 3600` (실수 시간)

### 4.2 시간당 지급량 (모든 수치는 `OfflineConfig.csv`에서 조정)

| 재화 | 시간당 공식 | 지급 계산 |
|---|---|---|
| 금색 메소 | `GoldBase + GoldPerFloor × B` = **600 + 60 × B** | `floor(시간당 × H)` |
| 경험의 서 | 경험치 `ExpBase + ExpPerFloor × B` = **300 + 30 × B** | `floor(시간당 × H)` 경험치를 큰 책부터 나눔 (§4.3) |
| 다이아 | `DiaBase + floor(B / DiaFloorStep)` = **2 + floor(B / 20)** | **꽉 찬 시간만**: `시간당 × floor(H)` |

**근거(금색 메소)**: 현재 판 지급은 스테이지당 `(20 + 3 × 층) × 배율(정예 2, 보스 5)`, 층당 10스테이지 → 한 판 약 `15 × (20 + 3B)`. 한 판을 약 12분으로 보면 직접 플레이 시간당 약 `75 × (20 + 3B)`. 방치 시간당은 그 **약 30%** 가 되도록 맞췄다 (니케·AFK도 방치 효율은 직접 진행보다 낮게 둔다).

### 4.3 경험의 서 나누기
`exp` 를 아래 순서로 나눈다. 나머지(100 미만)는 버린다.
1. `B ≥ BookLFloor(30)` 이면 대(2000)로 가능한 만큼
2. `B ≥ BookMFloor(10)` 이면 중(500)으로 가능한 만큼
3. 나머지를 소(100)로

### 4.4 예시 (12시간 가득)

| 최고 층 B | 금색 메소 | 경험치 → 경험의 서 | 다이아 |
|---|---|---|---|
| 1 | 7,920 | 3,960 → 소 39 | 24 |
| 10 | 14,400 | 7,200 → 중 14 · 소 2 | 24 |
| 30 | 28,800 | 14,400 → 대 7 · 중 0 · 소 4 | 36 |
| 60 | 50,400 | 25,200 → 대 12 · 중 2 · 소 2 | 60 |
| 97 | 77,040 | 38,520 → 대 19 · 중 1 · 소 0 | 72 |

(참고: 슬롯 강화 1회 비용 100 + 30 × 단계, 캐릭터 뽑기 1회 다이아 100)

### 4.5 2단계 후보 (확정 필요, 1단계에는 넣지 않음)
- **빠른 정산**: 버튼을 누르면 **2시간 분량**(위 공식, `T = 7200`)을 즉시 지급. 쌓인 시간(`last`)은 건드리지 않는다. 하루 1회 무료, 이후 다이아 `50 / 80 / 100` (하루 최대 3회 유료). 하루 기준은 한국 시간 0시.
- **확정 재료**: 쌓인 꽉 찬 `CubeEveryHours`(예 6)시간마다 큐브 1개(`PlayerGear:AddCubes`). 큐브 해금(Lv25) 전에는 지급 안 함.

---

## 5. 규칙과 서버 검증 (조작 방지)

1. **모든 계산·지급은 서버(ServerOnly)에서** 한다. 클라이언트는 "받기" 요청만 보낸다. 클라이언트 기기 시간은 쓰지 않는다.
2. 시각은 `math.floor(DateTime.UtcNow.Elapsed / 1000)` (에너지와 동일).
3. **시작 시각**: 저장값이 없는 계정(신규·기존 모두)은 처음 불러올 때 `last = now`로 저장하고 시작한다. **지난 기간 소급 없음** (계정 레벨 보상 규칙과 같은 원칙).
4. **쌓이는 기준 = 마지막 수령 시각** (AFK·니케와 같음). 접속 중에도 시간은 흐른다. 로그아웃 시각 저장(`OnEndPlay`)에 의존하지 않아서 비정상 종료에도 안전하다. → 사용자가 "접속 안 한 시간만"을 원하면 §9-1 대안 참고.
5. 서버 시각이 저장값보다 이전이면(시계 되감김) `T = 0`으로 보고 `last`를 바꾸지 않는다.
6. 상한 12시간을 넘긴 시간은 버린다(수령 시 `last = now`).
7. **수령 순서 (중복 지급 방지 우선)**: ① 상태 검사 → ② 보상 계산 → ③ `last = now`를 `SetAndWait`로 **먼저 저장** → ④ 저장 성공 시에만 재화 지급 → ⑤ 결과 Sync. 저장 실패면 아무것도 주지 않고 실패 알림. ③과 ④ 사이에서 서버가 죽으면 그 회차는 잃는다(중복 지급보다 낫다; 에너지 차감과 같은 원칙).
8. 수령 조건: 본인 요청(`senderUserId == UserId`), `OfflineLoaded`, `PlayerEngrave.Loaded`, `PlayerRoster.RosterLoaded`, `PlayerGear.GearLoaded`, 처리 중 아님(`_T.busy`), **로비 맵에서만**, `B ≥ 1`, `T ≥ MinClaimSec(60)`.
9. 연타 방지: `_T.busy` 동안 요청 무시. 클라이언트 버튼도 응답 올 때까지 비활성.
10. 디버그 판 여부와 무관(로비에서만 받으므로).

---

## 6. 저장 데이터

- 새 컴포넌트 전용 키: **`OrbisOfflineV1`** (UserDataStorage). 기존 키(`OrbisEngrave`, `OrbisRosterV1`, `OrbisGearV1` 등)는 건드리지 않는다.
- 값(JSON):

```json
{ "v": 1, "last": 1795000000, "fd": "2026-11-24", "fu": 0 }
```

| 필드 | 뜻 |
|---|---|
| `v` | 형식 버전 |
| `last` | 마지막 수령(또는 시작) 서버 UTC 초 |
| `fd` | (2단계) 빠른 정산 횟수를 센 한국 날짜 |
| `fu` | (2단계) 그 날짜에 쓴 빠른 정산 횟수 |

- 불러오기: `code == 1000002` 또는 빈 값 → 신규(`last = now` 저장). 그 외 실패/형식 오류 → `OfflineLoaded = false`, 10초 뒤 재시도, 그동안 수령 차단 (에너지와 같은 방식).

---

## 7. UI (텍스트 최소, 설명은 툴팁)

`Docs/UI-Design-System.md`를 따른다: 차콜 외곽 + 밝은 본문, 초록 = 실행, 공통 64×64 X, 버튼 4상태(SpriteSwap), 버튼 글자 정중앙, 오른쪽 위 금지 영역(x≥740, y≥410) 피하기.

### 7.1 로비 상자 버튼 (`ui/LobbyUI` 의 `Root/BtnOffline`)
- 위치: 왼쪽 위 기준(앵커·피벗 (0,1)), `(32, -236)`, 크기 **112×112**. `AccountChip`(위 28~132)·`BtnSkillTest`(146~218, 메이커 전용) 아래라 겹치지 않는다.
- 구성:
  - 배경: `button_*` 4상태 (Sliced, 경계 12, PreserveSprite None)
  - `Icon`: 닫힌 상자 프레임1 `35d6753b40ab48bf93f7c734b4634de3`, 80×80 중앙
  - `Gauge`: 아래쪽 안쪽 가는 막대(96×8). 채움 = `T / 상한`. 가득 차면 초록
  - `Time`: 막대 위 작은 글자 `h:mm` (예 `3:20`). 가득 차면 `12:00`
  - `Dot`: 오른쪽 위 빨간 점 20×20. `T ≥ 1시간` 또는 가득 차면 표시
- 잠금(`B = 0`): Disabled 이미지 + 입력 차단, 툴팁 "1층 클리어 시 열림".
- 누르면 보상 팝업(§7.2)을 연다.
- 툴팁(마우스 올림/길게 누름): 제목 `방치 보상`, 본문 `최고 {B}층 기준 · 최대 12시간` + 시간당 `금색 메소 {g} · 경험치 {e} · 다이아 {d}`.

### 7.2 보상 팝업 (새 `ui/OfflineUI`, 그룹 순서 **26** — 레벨업 연출(27)·툴팁(28) 아래)
- 화면 가운데 패널 **720×480** (`panel_outer` + 안쪽 `panel_light`). 뒤에 반투명 검정 전체 판(입력 차단).
- 위: 상자 그림 160×160 (닫힌 상자). 제목 글자는 넣지 않는다.
- 상자 아래: 진행 막대 520×16 + 오른쪽 `h:mm / 12:00` 글자 하나. 막대·글자에 툴팁 `최고 {B}층 기준, 최대 12시간까지 쌓입니다`.
- 가운데: 보상 칸 가로 정렬, 칸마다 **아이콘 64×64 + 아래 수량** (`card_normal` 바탕 96×120). 0인 재화는 칸을 숨긴다. 순서: 금색 메소 · 경험의 서(대·중·소) · 다이아. 아이콘을 누르면/올리면 툴팁으로 이름과 시간당 양.
- 아래: 초록 실행 버튼 `action_*` 4상태 **240×72**, 글자 `받기` (정중앙). `T < 60초` 또는 처리 중이면 Disabled + 입력 차단.
- (2단계) 그 왼쪽 `button_*` 240×72 `빠른 정산` + 비용 아이콘(무료면 `무료`, 아니면 다이아 아이콘+수량). 툴팁 `2시간 분량을 바로 받습니다 (하루 1회 무료)`.
- 패널 안 오른쪽 위(안쪽 여백 16) 공통 `close_*` X.
- 받기 성공: 상자를 10프레임 열림 애니메이션(1.2초, 기존 매니페스트 프레임 시간) → 보상 칸이 위쪽 `Wallet`으로 날아가는 연출은 선택 사항(없어도 됨) → 0.6초 뒤 팝업 자동 닫힘. 실패면 `_LobbyUILogic:ShowNotice(문구)`.
  - **(2026-11-24 구현 변경) 보상 순서 등장**: 받기 버튼 또는 상자를 누르면 받기(상자에 누르기 영역). 서버 성공 응답 뒤 상자 열림 연출과 함께, 실제 받은 칸만 0.35초부터 0.2초 간격으로 상자 위치(보상 묶음 기준 (0,170))에서 튀어나와 자기 자리로 0.45초 동안 이동(감속 + 위로 솟는 호), 크기 0.2→1.25→1.0, 투명도 0→1, 수량은 0부터 최종값까지 올라가고 안착 순간 짧은 효과음(UI 클릭음, 볼륨 0.5).
  - 모두 안착하고 상자 연출이 끝난 뒤 0.35초에 받기 버튼 자리 글자가 "확인"(같은 초록 4상태 버튼)으로 바뀌고 누르면 닫힘. 자동으로 닫지 않음. 연출 중 받기 버튼은 비활성(입력 차단), X를 누르면 연출만 건너뛰고 지급은 유지. 응답 대기 중에는 팝업 갱신을 멈춰 칸 깜빡임 방지.
  - 게이지: 채움은 원본 초록 게이지(`UICashShop.img/mvpView/gradeWnd/gaugeBar_next/gauge.png`, RUID `66a967155db7495284b99f037d9128bf`), 바탕은 같은 그림을 회색조·차콜로 바꾼 제작 리소스(RUID `6d89769154754047af7ca67da52a98d6`). 둘 다 좌우 9분할 6px, 가로 길이 조절(최소 12px). 진행 중 밝기 0.86, 가득 차면 원본 밝기 1.0.

### 7.3 자동 팝업 규칙
- 접속 후 **로비에 처음 들어왔을 때 1회만**, `B ≥ 1` 이고 `T ≥ AutoPopupSec(600, 10분)` 이면 연다.
- 레벨업 연출(`ui/LevelUpUI` 루트가 켜져 있음)이 떠 있으면 닫힐 때까지 기다렸다가 연다.
- 탑에서 로비로 돌아올 때는 자동으로 열지 않는다(버튼의 빨간 점으로만 알림).

---

## 8. 바꿀 파일·새로 만들 파일과 구현 순서

### 8.1 파일 목록

| 구분 | 경로 | 내용 |
|---|---|---|
| 새로 | `RootDesk/MyDesk/Offline/PlayerOffline.mlua` | `@Component`, 플레이어 엔티티 부착. 저장·계산·수령·디버그 (서버) + Sync 값 |
| 새로 | `RootDesk/MyDesk/Offline/OfflineUILogic.mlua` | `@Logic`, ClientOnly. 팝업·자동 팝업·툴팁 연결 |
| 새로 | `ui/OfflineUI.ui` | UIBuilder로 생성 (`.ui` 경로는 node에서 `'ui/'+'OfflineUI'+'.'+'ui'` 로 조립) |
| 새로 | `RootDesk/MyDesk/Data/OfflineConfig.csv` + `OfflineConfig.userdataset` | 수치 표 (UTF-8 BOM, `PetConfig.userdataset`을 복사해 새 uuid·name) |
| 수정 | `Global/DefaultPlayer.model` | ModelBuilder로 `script.PlayerOffline` 컴포넌트 추가 (기존 컴포넌트 유지) |
| 수정 | `ui/LobbyUI.ui` | UIBuilder로 `Root/BtnOffline`(+Icon/Gauge/Time/Dot) 추가, 버튼 4상태 패치 |
| 수정 | `RootDesk/MyDesk/Lobby/LobbyUILogic.mlua` | `btnOffline` 속성·바인딩·`RefreshOffline()`(OnUpdate에서 값이 바뀔 때만 갱신) 작은 추가 |
| 수정 안 함 | `PlayerEngrave.mlua`, `RelicUILogic.mlua`, `ui/RelicUI`, `.codeblock` | 읽기만 (다른 세션 작업 중 / 자동 생성) |

### 8.2 `OfflineConfig.csv` (열: `Key,Value,#Desc`)

```
Key,Value,#Desc
MaxHours,12,최대 누적 시간
MinClaimSec,60,받기 최소 누적 초
AutoPopupSec,600,접속 시 자동 팝업 최소 누적 초
GoldBase,600,금색 메소 시간당 기본
GoldPerFloor,60,금색 메소 시간당 층당 증가
ExpBase,300,경험치 시간당 기본
ExpPerFloor,30,경험치 시간당 층당 증가
BookMFloor,10,경험의 서 중 지급 시작 층
BookLFloor,30,경험의 서 대 지급 시작 층
DiaBase,2,다이아 시간당 기본 (꽉 찬 시간만)
DiaFloorStep,20,이 층마다 다이아 시간당 +1
FastHours,2,(2단계) 빠른 정산 시간
FastFree,1,(2단계) 하루 무료 횟수
FastCosts,50|80|100,(2단계) 유료 다이아 비용
```

### 8.3 `PlayerOffline.mlua` 설계

Sync 속성 (클라이언트 표시용, 서버가 1초마다 갱신):

| 속성 | 타입 | 뜻 |
|---|---|---|
| `OfflineLoaded` | boolean | 불러오기 완료 |
| `OfflineSec` | integer | 현재 쌓인 초 `T` (상한 적용) |
| `OfflineMaxSec` | integer | 상한 초 |
| `OfflineFloor` | integer | 계산에 쓰는 `B` |
| `OfflinePreview` | string | `"gold:s:m:l:dia"` 지금 받으면 얻는 양 |
| `OfflineRate` | string | `"gold:exp:dia"` 시간당 (툴팁용) |
| `OfflineSeq` | integer | 수령 응답 순번 |
| `OfflineMsg` | string | `ok` / `short` / `busy` / `notready` / `save_fail` / `locked` |
| `OfflineResult` | string | `"gold:s:m:l:dia:sec"` 실제 지급량 |

메서드 (모두 본문 첫 줄에 한국어 설명 주석, `[VRF]` 로그 포함):

| 메서드 | 공간 | 내용 |
|---|---|---|
| `OnBeginPlay` | ServerOnly | `LoadConfig()`, `LoadOffline()` |
| `OnUpdate(delta)` | ServerOnly | 미로드면 재시도 타이머. 1초마다 `RefreshPreview()` |
| `Now()` | ServerOnly | `math.floor(DateTime.UtcNow.Elapsed / 1000)` |
| `LoadConfig()` | ServerOnly | `_DataService:GetTable("OfflineConfig")` → `_T.cfg`. 없으면 기본값 |
| `Cfg(key, fallback)` | 공용 | 설정 읽기 (`PlayerRoster:Cfg` 방식) |
| `LoadOffline()` | ServerOnly | §6 규칙. 신규면 `last = Now()` 저장 후 로드 완료 |
| `Encode()` | ServerOnly | §6 JSON |
| `Floor()` | ServerOnly | `PlayerEngrave.ClearedFloor` 읽기 (미로드면 -1) |
| `Accum(now)` | ServerOnly | §4.1 `T` |
| `Calc(B, sec)` | ServerOnly | §4.2~4.3 → `{gold, s, m, l, dia}` |
| `RefreshPreview()` | ServerOnly | Sync 값 갱신 (값이 바뀔 때만 대입) |
| `RequestClaim()` | **Server** | 클라이언트 요청. `senderUserId` 확인 후 `DoClaim()` |
| `DoClaim()` | ServerOnly | §5-7 순서. `SetAndWait` 성공 후 `gear:AddGoldMeso(gold)`, `gear:Flush()`, `roster:GiveBooks(s,m,l)`, `roster:GiveDiamonds(dia)` |
| `Reply(code, result)` | ServerOnly | `OfflineSeq += 1`, Msg/Result 대입, 로그 |
| `DebugShift(sec)` | ServerOnly | **메이커 플레이에서만** (`Environment:IsMakerPlay()`). `last = last - sec` 저장 → 시간을 당긴 효과 |
| `DebugSetLast(agoSec)` | ServerOnly | 메이커 전용. `last = Now() - agoSec` 저장 |
| `DebugClaimCalc(B, sec)` | ServerOnly | 메이커 전용. 지급 없이 `Calc` 결과만 로그 (공식 확인) |

### 8.4 `OfflineUILogic.mlua` 설계 (ClientOnly)
- 내 플레이어의 `PlayerOffline` 컴포넌트를 찾아 Sync 값을 읽는다 (`_UserService.LocalPlayer`).
- `Open()` / `Close()` / `OnClaim()` (버튼 → `comp:RequestClaim()`; 버튼 Disabled 후 `OfflineSeq` 변화 대기).
- `OnUpdate`: 열려 있으면 막대·시간·보상 칸을 Sync 값이 바뀔 때만 갱신. `OfflineSeq`가 바뀌면 결과 처리(성공 → 상자 열림 연출 → 닫기, 실패 → 안내).
- 자동 팝업: §7.3. `_T.autoDone` 으로 접속당 1회. 로비 맵이고 `_LevelUpUILogic` 연출이 꺼졌을 때.
- 툴팁: 보상 아이콘·막대·로비 버튼에 `_TooltipLogic:Register` + `ShowText`.

### 8.5 구현 순서
1. **준비**: `Skill` 도구로 `msw-general`, `msw-ui-system`, `msw-scripting` 로드, 기초 참조 문서·`builder-protocol.md`·`builder-protocol-ui.md`·`builder-protocol-model.md` 읽기 (AGENTS.md 규칙).
2. `OfflineConfig.csv` + `.userdataset` 만들기.
3. `PlayerOffline.mlua` 작성 → ModelBuilder로 `Global/DefaultPlayer.model`에 부착 → 메이커 새로고침 → 빌드 로그 오류 0 → 플레이해서 `[VRF] offline loaded` 확인.
4. 디버그로 공식·수령 서버 동작 확인 (§10 1~5).
5. `ui/OfflineUI.ui` 생성(UIBuilder) + 버튼 4상태 패치(`Transition: 2` + `ImageRUIDs`) + `OfflineUILogic.mlua` 작성, `injectBindings`.
6. `ui/LobbyUI.ui` 에 `BtnOffline` 추가 + `LobbyUILogic.mlua` 바인딩·갱신.
7. 메이커 확인: 자동 팝업·받기·잠금·툴팁 (§10 6~9). UI 린트·스크린샷.
8. (사용자 확정 후) 2단계 빠른 정산.

---

## 9. 확정 필요 (구현 전 사용자에게 질문)

1. **쌓이는 기준**: (권장) 마지막 수령부터 계속 쌓임(AFK·니케 방식, 접속 중에도 쌓임) / (대안) 접속하지 않은 시간만 — 대안은 로그아웃 시각을 `OnEndPlay`에서 저장해야 하고 비정상 종료 시 누락될 수 있으며, 접속 중엔 쌓이지 않으므로 버튼 게이지가 의미 없어진다.
2. **보상 종류**: 금색 메소 + 경험의 서 + 다이아(소량)로 충분한지, 큐브·탑의 열쇠 등 추가할지.
3. **최대 시간**: 12시간(AFK) / 24시간(니케).
4. **수치**: §4.2 기본값(방치 시간당 ≈ 직접 플레이의 30%)으로 시작해도 되는지.
5. **빠른 정산(2단계)** 도입 여부와 비용(무료 1회 + 다이아 50·80·100).
6. **해금**: 1층 클리어로 열기(권장) / 계정 레벨 해금 표(`UnlockTable.csv`)에 넣기.

---

## 10. 테스트 방법 (기능만 짧게, 밸런스 측정 안 함)

메이커 요령은 `Docs/Job-Redesign/01-Agent-Rules.md` "메이커 사용 요령"을 따른다. 사용자가 시작한 플레이는 멈추지 않는다.

서버 스크립트 예 (`maker_execute_script`, context `server_main`):

```lua
local p
for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
local o = p:GetComponent("script.PlayerOffline")
o:DebugShift(3 * 3600)      -- 3시간 당기기
wait(1.5)
log("[VRF] offline sec=" .. tostring(o.OfflineSec) .. " preview=" .. o.OfflinePreview .. " floor=" .. tostring(o.OfflineFloor))
```

| # | 확인 | 방법 | 기대 |
|---|---|---|---|
| 1 | 신규 시작 | 저장 없는 상태로 로드 | `OfflineSec` ≈ 0, 소급 없음 |
| 2 | 공식 | `DebugClaimCalc(10, 43200)` | 금색 메소 14400 · 중 14 · 소 2 · 다이아 24 |
| 3 | 상한 | `DebugShift(20 * 3600)` | `OfflineSec == 43200` |
| 4 | 수령 | `DebugShift(3600)` 후 `o:DoClaim()` | `OfflineMsg == "ok"`, 금색 메소·다이아·경험의 서 증가, `OfflineSec` ≈ 0 |
| 5 | 연속 수령 차단 | 4 직후 다시 `DoClaim()` | `short`, 지급 없음 |
| 6 | 자동 팝업 | `DebugShift(3600)` → 재접속(플레이 재시작) | 로비 진입 시 팝업 1회 |
| 7 | 받기 버튼 | 클라이언트에서 받기 버튼에 `SendEvent(ButtonClickEvent())` | 상자 열림 → 닫힘, 로비 재화 갱신 |
| 8 | 잠금 | `ClearedFloor = 0` 계정 | 로비 버튼 비활성·입력 차단, 툴팁 |
| 9 | 저장 복원 | 4 후 재시작 | `last` 유지(다시 쌓이지 않음) |

> 메이커 플레이의 DataStorage는 출시 월드와 분리되어 있다. 디버그 메서드는 `Environment:IsMakerPlay()`가 아니면 아무것도 하지 않아야 한다.
