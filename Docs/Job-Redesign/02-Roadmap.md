# 직업·캐릭터 개편 전체 로드맵

> 작성: 자료 조사 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(최우선), `01-Agent-Rules.md`, `Docs/UI-Design-System.md`
> 이 문서는 단계 순서·작업 단위·완료 조건을 정한다. 단계별 세부 수치는 `Phase-N-*.md`에서 확정한다.

---

## 1. 다른 게임 조사 요약

### 1-1. 사례

| 게임 | 레벨업 재화 | 승급(전직)·한계 돌파 | 엔드 콘텐츠 강화 | 화면 배치 |
|---|---|---|---|---|
| 붕괴: 스타레일 | 경험치 책 3단계 (1,000 / 5,000 / 20,000, 비율 1:5:20) | 20·30·40… 레벨마다 **승급 관문**. 관문에서 레벨이 멈추고 재료를 넣어야 한도가 오름 | 성흔(중복 캐릭터)·광추 | 목록(좌) + 큰 일러스트(중앙) + 오른쪽 세로 탭(상세/레벨업/스킬). 레벨업 창은 재료 3칸 + 자동 채우기 + 목표 레벨 미리보기 |
| 명일방주 | 작전 기록 4단계 (200 / 400 / 1,000 / 2,000) | 정예화 E1·E2: **최대 레벨 도달 후** 직업군 칩(전용 계열 재료) + 화폐로 승급. 승급하면 한도 상승·외형 변화 | 모듈·스킬 특화 | 승급 화면에서 "필요 레벨 / 필요 재료 / 보유량"을 한 줄씩 비교해 부족분을 붉게 표시 |
| 에픽세븐 | 펭귄(경험치 아이템), 자동 선택 버튼 | 진화: 최대 레벨 후 같은 등급 재료 소모. 레벨업 화면에서 자동 선택 버튼 제공 | 각성 6단계(속성 룬, 마지막 단계는 희귀 촉매) — 3단계마다 스킬 강화 | 영웅 목록 격자 → 상세(스탯·스킬·장비 탭) |
| 블루 아카이브 (일반 지식) | 활동 보고서 4단계 | 학생 **전용 신비 조각**으로 성급 상승, 부족하면 **공용 대체재(엘리그마)**로 교환 | 고유 무기 | 뽑기 1회당 모집 포인트 1점, 200점 = 원하는 픽업 학생 교환 (천장) |
| 승리의 여신: 니케 (일반 지식) | 신용·전투 데이터 칩 | 동기화 장치: 일정 레벨 이후 한계 돌파는 중복 캐릭터 | — | 10연 뽑기, 마일리지 상점 교환 |

### 1-2. 공통으로 잘 된 점

1. **경험치 아이템은 3~4단계, 대략 1:5:20 비율**. 자동 채우기 + 목표 레벨 미리보기가 사실상 필수.
2. **승급 관문**: 정해진 레벨에서 레벨이 멈추고, 재료를 넣으면 한도가 풀린다. 레벨과 승급이 서로를 막아 성장 속도를 조절한다.
3. **전용 재료 + 공용 대체재**: 전용 재료가 부족할 때 공용 재료로 1:1 메운다. 공용 재료는 희귀해서 "가장 좋은 결과"가 된다.
4. **뽑기 천장은 교환 포인트**: 1회 1점, 일정 점수로 원하는 대상과 교환. 확률 천장보다 이해하기 쉽다.
5. **엔드 강화는 확률 + 단계별 눈에 띄는 변화**(에픽세븐 각성: 3단계마다 스킬 강화). 우리 스펙의 "각성 단계마다 스킬 하나씩 각성"과 같은 구조.

### 1-3. 우리 프로젝트에 맞춘 결론

- **경험의 서 소·중·대 = 100 / 500 / 2,000 경험치** (1:5:20). 레벨업 창에는 칸 3개, `+1레벨`, `최대(전직 관문까지)` 버튼, 결과 미리보기만 둔다.
- **전직 관문 = 각 차수의 최대 레벨(7·13·19·25)**. 관문에서 경험치가 더 쌓이지 않고 경험의 서도 소모되지 않는다. 전직하면 같은 레벨에서 다음 차수가 시작된다 (명일방주·스타레일 방식).
- **전직 카드**: 전용 카드 우선 소모, 부족분은 **공용 카드 1:1** 대체. 대체 사용은 버튼을 따로 눌러 확인한다 (희귀 재화 실수 방지).
- **뽑기 천장**: 블루 아카이브식 교환 포인트(1회 1점, 30점 = 원하는 직업 전용 카드 1장). 3단계에서 확정.
- **화면 배치(1920×1080, 모바일 겸용)**: 기존 성장창(GrowthUI) 골조를 그대로 쓴다.
  - 왼쪽: 캐릭터 목록 세로 카드 5장 (직업 아이콘·차수 이름·Lv, 미해금은 어두운 덮개 + 자물쇠).
  - 가운데: 선택 캐릭터 아바타 미리보기 + 차수 이름 + 레벨·경험치 바 + 능력치 3줄.
  - 오른쪽: 탭 2개(레벨업 / 전직), 아래 초록 실행 버튼. 스킬 칸 6개는 가운데 하단에 한 줄로.
  - 버튼은 최소 96px 높이로 터치 영역 확보. 공통 64×64 X 닫기.

출처: [Game8 스타레일 경험치 재료](https://game8.co/games/Honkai-Star-Rail/archives/409162), [Push Square 스타레일 경험치 재료](https://pushsquare.com/guides/honkai-star-rail-how-to-get-character-exp-materials), [명일방주 위키 EXP](https://arknights.wiki.gg/wiki/EXP_(Operator)), [명일방주 위키 Promotion](https://arknights.wiki.gg/wiki/Promotion), [명일방주 위키 Chip](https://arknights.wiki.gg/wiki/Chip), [BlueStacks 에픽세븐 캐릭터 가이드](https://www.bluestacks.com/blog/game-guides/epic-seven/es-character-guide-en.html), [Game8 에픽세븐](https://game8.co/games/Epic-Seven/archives/273694). 블루 아카이브·니케는 일반적으로 알려진 구조를 참고했다.

---

## 2. 우리 프로젝트 현황과 개편 영향 지점

### 2-1. 계정 데이터 — `RootDesk/MyDesk/Engrave/PlayerEngrave.mlua` (다른 세션 수정 중, 손대지 않음)

| 현재 기능 | 저장 키 | 개편 영향 | 처리 단계 |
|---|---|---|---|
| 편성 `MainJob / CompJob / Comp2Job` (저장 `mj·cj·c2`, `OrbisSkill` 안) | OrbisSkill | 해금된 직업만 편성 가능해야 함. **1~5단계 동안은 편성의 원본으로 계속 사용**. 새 `PlayerRoster`가 서버에서 직접 값을 바꾸고 `SaveSkills()`를 호출 (PlayerEngrave 코드 변경 0) | 1 |
| 편성 요청 `RequestSetFormation` (해금 검사 없음) | — | 새 UI는 `PlayerRoster.RequestSetParty`를 부른다. 옛 함수는 6단계에서 제거 | 2·6 |
| 스킬 뽑기·레벨·합성·장착 (`SkillOwnLv`, `SkillCnt`, `SkillEquip`, `SkillBooks`, `RequestGacha` 등) | OrbisSkill | **삭제 대상**. 4단계에서 전투가 칸 진화로 바뀐 뒤 6단계에서 UI·저장 정리 | 4·6 |
| 고유 패시브 레벨 `ComboAttackLv / HolyWaterLv / JobPassiveLv` | OrbisSkill | 차수 기반 패시브로 대체 | 4·6 |
| `SkillRunDataFor(job)` → 전투 컴포넌트 `BeginRun`이 읽음 | — | 4단계에서 `PlayerRoster:CharRunData(job)`로 교체 | 4 |
| 슬롯 강화 `SlotLv / SlotBonus / Scrolls` (공용 6부위, 신발 포함) | OrbisSlot | 5단계에서 직업별 5칸으로 이전 | 5 |
| 별 `RecordStars / RequestClaimSection` (구간 3단계 상자: 스킬북·깃털) | OrbisStar | 3단계: **마지막 상자에 다이아 추가**, 스킬북 보상은 6단계에서 경험의 서로 교체 | 3·6 |
| `ReportCleared(floor)` (최고 클리어 층) | OrbisEngrave | 3단계: 첫 클리어 다이아 지급 훅 (반환값 true일 때 = 첫 클리어) | 3 |
| 유물(탑의 열쇠) | OrbisRelicV2 | 영향 없음. 재화는 분리 유지 | — |

### 2-2. 판 진행 — `RootDesk/MyDesk/Tower/TowerRunManager.mlua`

| 지점 | 현재 | 개편 |
|---|---|---|
| `Level / Exp / ExpToNext`, `StageLevelUp`, `BeginWave`의 `ExpToNext = size`, `DoLevelUp`, `HasSkillToLearn`, `PendingPicks` | 스테이지마다 판 안 레벨업 + 스킬 카드 | 4단계에서 삭제 (판 안 레벨 고정, 스킬 카드 없음). 돌파 카드 흐름(`RollBreakChoices`, `RollPartyBreaks`)은 유지 |
| `RecalcStats`(2738행 부근) `BasePlayerAtk + LevelAtkBonus*(Level-1)` 등 | 판 안 레벨 스탯, 동료는 메인과 같은 체력 | 4단계: 구성원별 **캐릭터 레벨 스탯**(PlayerRoster)으로 교체. 동료도 자기 직업 스탯 |
| `StartRun` → `RollJobChoices` (직업 선택 카드), `startLevel`, `PendingPicks = startLevel` | 판 시작 시 직업 카드·스킬 카드 | 4단계: 편성 그대로 시작, 스킬은 차수 칸에서 바로 장착 |
| `LoadFormation` (eng.MainJob 등 읽음) | 편성 | 1단계에서 PlayerRoster가 PlayerEngrave 편성을 해금 직업으로 보정하므로 **변경 불필요** |
| `SpawnCompanion` (`CompanionModels` 직업→모델) | 직업별 모델 | 2단계: 차수 외형을 구성원에게 적용 |
| `ApplyCostume / CopyCostumeTo` | 장비 단계 외형 + 직업 무기 | 2단계: **차수 외형 세트**로 교체 (장비 외형 사용 안 함) |
| `StageClear` (스킬북·깃털 지급, 별 판정) | 임시 보상 | 3단계: 첫 클리어 다이아. 6단계: 스킬북→경험의 서 |
| `Bonus("StartLevel")`, `Bonus("ExtraPick")` (각인 효과) | 판 안 레벨 관련 각인 | 4단계: 효과 무력화 또는 다른 효과로 교체 (EngraveTable 확인) |
| `EquipTier / EquipEnh / SlotLevels` (판 안 장비) | 6부위 | 5단계 |

### 2-3. 직업 전투 — `HeroCombat / BishopCombat / JobCombat`

- `BeginRun(mgr)`이 `eng:SkillRunDataFor(job)`으로 장착 5칸·레벨을 받는다. `Equipped(id)`로 스킬 사용 여부 판정, `RunSkills(mgr)`로 목록 제공.
- 4단계에서 장착 목록을 **칸 6개의 현재 차수 스킬 Id**로 바꾸면 `Equipped` 기반 코드 대부분을 그대로 쓸 수 있다. 스킬 레벨은 차수·각성으로 대체.
- 하이퍼(긴 쿨타임 자동)·궁극기(게이지) 시전 흐름은 새로 만든다.

### 2-4. 데이터

| 파일 | 내용 | 개편 |
|---|---|---|
| `HeroSkillTable.csv` | 5직업 × 15스킬 (Order, SkillId, Name, Grade, IconRUID …, JobId) | 스킬 정의로 계속 사용. Grade(뽑기 등급)는 6단계에서 무의미 |
| `JobSkillTable.csv` | 전투 수치 (Trigger, Cooldown, Type1/V1 …) | 4단계에서 칸 진화에 맞게 수치 조정 |
| `RunConfig.csv` | `LevelAtkBonus` 등, `GachaCost`, `StageLevelUp`, `StageBook*` | 4·6단계에서 해당 키 정리 |
| `EquipSlotTable / EquipTable / Slot*Table` | 6부위 장비·주문서 | 5단계 |
| `SkillGradeTable / SkillStarOptionTable / SkillBreakTable` | 뽑기 등급·돌파 | 돌파는 칸 연결로 유지(4단계), 등급표는 6단계 정리 |
| 신규 | `CharJobTable`, `CharLevelTable`, `CharTierTable`, `CharSlotTable`, `CharConfig` | 1단계 |

### 2-5. 로비 UI

- `ui/LobbyUI` + `Lobby/LobbyUILogic.mlua`: 메뉴 버튼 `btnPlay / btnGrowth / btnAchieve / btnGacha(스킬 뽑기) / btnSkillTest`. 보조 메뉴는 288×96 규격.
- `ui/GrowthUI` + `Lobby/GrowthUILogic.mlua`: 탭 5개(슬롯 강화/각인/심볼(현 유물 대체)/은총/스킬). 6단계에서 스킬 탭 제거.
- 2단계: 로비에 **캐릭터** 버튼 추가(새 `ui/CharacterUI`). 3단계: `btnGacha`를 **캐릭터 뽑기**로 전환.

---

## 3. 단계별 작업 단위

각 작업 단위는 구현 에이전트 1명이 한 번에 끝낼 크기다. 단위마다 메이커 빌드 오류 0 + `[VRF]` 로그 확인이 기본 완료 조건이다.

### 1단계: 캐릭터 핵심 (데이터 + 계정 저장 + 서버 로직, UI 없음)
상세: `Phase-1-Character-Core.md`

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 1-A | 데이터 표 5개(csv + userdataset) 추가 | 메이커 새로고침 후 `_DataService:GetTable`로 5개 모두 행 수 확인 |
| 1-B | `Character/PlayerRoster.mlua` 저장·불러오기·첫 지급, DefaultPlayer 모델에 부착 | 새 계정 첫 접속 시 3직업 Lv1 + 시작 재화, 재접속 후 유지 |
| 1-C | 경험의 서 사용·레벨업·자동 계산 | 테스트 T1~T5 통과 |
| 1-D | 전직·해금·카드 지급·편성 보정·판 전달 함수 | 테스트 T6~T14 통과 |

### 2단계: 로비 캐릭터 화면 + 차수 외형

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 2-A | 차수 외형 세트 표 `CharLookTable.csv` (직업 5 × 차수 5, 부위별 아바타 RUID). `msw-search`/`msw-avatar`로 레벨대 비슷한 세트 선정 | 25세트 RUID가 모두 실제 아이템, 선정 목록 보고 |
| 2-B | 외형 적용: 로비 플레이어(메인 직업)·탑 구성원(`ApplyCostume`/`CopyCostumeTo` 교체) | 전직 직후 로비 외형 변경, 탑에서 3명 모두 자기 차수 외형 |
| 2-C | `ui/CharacterUI` 골조(목록 5장·미리보기·능력치·스킬 칸 6개·탭·X) + 로비 캐릭터 버튼 | 스크린샷으로 골조·정렬 확인, 버튼 상태 이미지 적용 |
| 2-D | 레벨업 탭(경험의 서 3칸, +1레벨, 최대, 미리보기) 연결 | 클릭 이벤트로 레벨 상승 확인, 재화 부족 시 비활성 |
| 2-E | 전직 탭(필요 레벨·전용 카드·공용 카드 대체 버튼) + 편성 3칸 변경 | 전직 성공 시 차수 이름·외형·스킬 칸 갱신, 미해금 편성 불가 |

### 3단계: 다이아 캐릭터 뽑기 + 다이아 지급

> 확정본: `Phase-3-Gacha.md`. 작업 단위 이름이 아래 표와 다르다(3-A 2단계 보완 / 3-B 뽑기 서버 / 3-C 뽑기 UI / 3-D 다이아 보상). 아래 표는 처음 제안으로 남겨 둔다.

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 3-A | 뽑기 표 `CharGachaTable.csv`(상시/픽업 풀, 확률), `CharBannerTable.csv`(픽업 기간·대상) | 확률 합계 100% 검증 로그 |
| 3-B | 서버: 1회/10연(100/1000 다이아), 10연 직업 카드 1장 보장, 교환 포인트(1회 1점, 30점 = 원하는 직업 카드), 결과는 PlayerRoster `GiveBooks/GiveJobCard/GiveCommonCard` 사용 | 1000회 모의 뽑기 분포 로그, 보장 위반 0 |
| 3-C | 다이아 지급: 층 첫 클리어(`ReportCleared` true 시점, TowerRunManager에서 호출), 별 구간 마지막 상자 | 첫 클리어 1회만 지급, 재클리어 0 |
| 3-D | 뽑기 UI(로비 `btnGacha` 전환, 결과 연출은 유물 뽑기 연출 재사용 검토) | 결과 카드 표시, 신규 해금 표시 |

제안 수치(3단계 문서에서 확정): 층 첫 클리어 다이아 = `20 + 2 × 층`(1층 22 ~ 97층 214, 합계 약 11,400), 별 구간 마지막 상자 300. 확률 제안: 경험의 서 소 45% / 중 30% / 대 12% / 직업 전용 카드 12%(픽업은 그중 절반을 픽업 직업에) / 공용 카드 1%.

### 4단계: 전투 개편

> 확정본: `Phase-4-Combat.md`. 작업 단위가 아래 표와 다르다(4-A 판 시작 정리 / 4-B 판 레벨 삭제·캐릭터 스탯 / 4-C 칸 스킬 / 4-D 하이퍼·궁극기 / 4-E 패시브 / 4-F 돌파 칸 연결 / 4-G 밸런스). 아래 표와 보류 항목은 처음 제안으로 남겨 둔다(보류 항목 2개는 확정본 4-A에 반영).

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 4-A | 판 안 레벨·스킬 카드·직업 선택 카드 삭제 (`DoLevelUp`, `StageLevelUp`, `PendingPicks`, `RollJobChoices` 흐름) | 판 시작 즉시 전투, 레벨업 연출 없음 |
| 4-B | `RecalcStats`를 구성원별 캐릭터 스탯(`PlayerRoster:CharRunData`)으로 교체 | 레벨 30 캐릭터가 레벨 1보다 표 값만큼 강함 |
| 4-C | 전투 3종 `BeginRun`의 장착 목록 = 칸 6개 현재 차수 스킬 | 차수별 사용 스킬이 진화표와 일치 |
| 4-D | 하이퍼(긴 쿨타임 자동 시전) + 궁극기 게이지(기본 공격·스킬 적중으로 충전, 가득 차면 자동) + HUD 게이지 | 4차 이상만 하이퍼, 5차만 궁극기 |
| 4-E | 고유 패시브 차수별 효과(기존 위기·확률 발동 흡수) | 차수별 효과 로그 |
| 4-F | 돌파를 칸(SlotKey)에 연결: 진화해도 돌파 유지 | 판 중 진화 없음이지만 저장 키가 칸 기준인지 확인 |

보류 항목 (이전 단계에서 넘어온 것 — 4단계 요구사항 문서에 반드시 포함):

| 출처 | 내용 | 기대 |
|---|---|---|
| 1단계 보완 Q8 | 탑에서는 `FixParty`가 돌지 않아 `PartyRunData()`가 잠긴·중복 직업을 넘길 수 있음 | 판 시작 때 `FixParty`와 같은 규칙으로 바꾸고 `[VRF] run party fixed a,b,c` |
| 2단계 테스트 Q7 | **메인(1번)을 바이퍼로 두고 판을 시작하면 HUD 1번 칸이 `1 나이트로드 Lv.1`, 자원 `표식 0/100`, 첫 스킬 칸 아이콘 비어 있음.** 플레이어 엔티티의 `JobCombat`이 시작부터 `job=NightLord`로 뜸(`[VRF] JobCombat begin job=NightLord`). 외형은 바이퍼로 정상 | 4-C(`BeginRun` 교체) 때 메인 직업이 무엇이든(나이트로드·보우마스터·바이퍼) 플레이어 `JobCombat`·HUD가 편성 1번 직업으로 시작. 메인 바이퍼·보우마스터로 각각 판 시작해 HUD 이름·자원·스킬 아이콘 확인 |

### 5단계: 편성 칸(1~3)별 장비 5부위

> 확정본: `Phase-5-Equipment.md`. 작업 단위가 아래 표와 다르다(5-A 데이터·저장·이전 / 5-B 주문서 직업별 + 장비 탭 / 5-C 판 반영 1 / 5-D 큐브 서버 / 5-E 잠재능력 UI / 5-F 판 반영 2 / 5-G 회귀). 아래 표는 처음 제안으로 남겨 둔다.

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 5-A | 직업별 5칸(무기·투구·갑옷·장갑·망토) 저장, 기존 `OrbisSlot` 6부위 → 직업별 이전(신발 단계는 보상으로 환산) | 이전 1회만, 기존 단계 보존 |
| 5-B | 주문서 강화를 직업별 칸으로 연결 | 기존 강화 확률·비용 유지 |
| 5-C | 큐브 잠재능력(등급 4개, 칸별 옵션 풀, 등급 상승 확률) | 옵션 풀 표·확률 로그 |
| 5-D | 상위 큐브 비포/애프터 선택 | 선택 전 재접속 시 이전 옵션 유지 |
| 5-E | 장비 UI(캐릭터 화면 안 장비 탭) | 스크린샷 확인 |

### 6단계: 각성 + 옛 시스템 정리 + 밸런스

| 단위 | 내용 | 완료 조건 |
|---|---|---|
| 6-A | 각성 Lv30~40: 각성의 서로 각성 경험치, 각성의 돌 + 다이아로 확률 단계 상승, 단계마다 스킬 하나 각성 | 확률·실패 처리 로그 |
| 6-B | 스킬 뽑기·스킬 레벨·합성·장착·패시브 레벨 UI·저장 정리, GrowthUI 스킬 탭 제거, 스킬북 보상 → 경험의 서 | 옛 기능 진입점 0, 기존 저장이 깨지지 않음 |
| 6-C | 각인 중 판 안 레벨 관련 효과 교체 | EngraveTable 갱신 |
| 6-D | 밸런스: 층별 몬스터 스탯 vs 캐릭터 레벨 곡선, 다이아 수급 | 1~30층 시뮬레이션 기록 |

---

## 4. 단계 간 의존

- 2단계는 1단계 서버 함수만 쓴다. 4단계는 1단계 `CharRunData`/`PartyRunData` 인터페이스를 쓴다.
- 3단계 뽑기는 1단계 `GiveBooks / GiveJobCard / GiveCommonCard`를 쓴다.
- 6단계 각성은 1단계 저장 형식의 `awkLv / awkExp`, `awkBook / awkStone` 칸을 쓴다.
- PlayerEngrave는 다른 세션이 작업 중이므로 6단계 전까지 **읽기 + 기존 서버 함수 호출만** 한다.
