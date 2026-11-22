# 5단계 요구사항: 편성 칸 장비 (주문서 강화 + 큐브 잠재능력)

> **2026-11-22 편성 칸 방식으로 개정** — 직업별 25칸(5직업×5부위) 안을 폐기하고, 편성 칸 1~3마다 5부위(15칸)로 바꿨다. 같은 날 큐브를 4종(무료·레드·블랙·플래티넘)으로 확정했다.
> 작성: 자료 조사 에이전트(초안) → 5단계 구현 에이전트(개정), 2026-11-22
> 기준: `00-Master-Spec.md`(최우선, 6절 장비) → `01-Agent-Rules.md` → `02-Roadmap.md` 5단계 → 이 문서
> 선행: 1~4단계 완료(4-G 보완 반영). `PlayerRoster`(1단계), `RunPartyData`/`RunDataFor`(4단계)를 그대로 쓴다.
> 이 문서만 보고 구현·테스트할 수 있도록 바꿀 파일·함수·값·완료 조건·확인 항목을 적는다. 문서에 없는 판단이 필요하면 마스터 스펙 의도대로 정하고 보고에 적는다.
> **건드리지 않는 파일**: `Engrave/PlayerEngrave.mlua`(읽기 + 기존 공개 값 읽기만), `Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md`(읽기만). 저장 키 `OrbisSlot`은 **읽기만** 하고 지우거나 덮어쓰지 않는다.

---

## 0. 결정 사항 요약

| 번호 | 결정 | 근거 |
|---|---|---|
| E1 | 장비 데이터는 새 컴포넌트 **`Character/PlayerGear.mlua`** + 새 저장 키 **`OrbisGearV1`** | `PlayerRoster` 저장 형식은 1~4단계에서 굳었다. 장비는 15칸×(단계·잠재·대기 결과)라 따로 불러오기·저장·재시도하는 편이 안전하다 |
| E2 | **편성 칸(1~3)마다 5부위 무기·투구·갑옷·장갑·망토 = 15칸**. 내부 Id는 `Weapon/Cap/Coat/Glove/Cape`(`Coat` 표시 이름만 `갑옷`). **신발(`Shoes`) 제거** | 마스터 스펙 6절(사용자 확정). Id를 유지하면 판 안 장비표·주문서표·아이콘을 그대로 쓴다 |
| E3 | 한 칸 = **주문서 단계(0~30, 기존 규칙) + 잠재능력(등급 + 3줄)**. 모두 **편성 칸 소유**. 칸에 들어온 캐릭터가 그 칸 장비 능력치를 받고, 캐릭터를 바꿔도 강화는 칸에 남는다. 잠재능력 계승 없음 | 마스터 스펙 6절 |
| E4 | 기존 계정 공용 슬롯(`OrbisSlot`) → **편성 칸 3개 모두에 같은 단계를 복사**. 신발 단계는 큐브로, 신발 주문서는 다른 5부위 주문서로 환산. **처음 1회만** | 쌓은 공용 단계를 어느 칸도 잃지 않는다. 4-1 |
| E5 | 주문서 보유량은 **계정 공용**(부위·단계별). 어느 칸의 같은 부위에도 쓸 수 있다. 보관 위치를 `PlayerEngrave`에서 `PlayerGear`로 옮긴다 | 드랍은 부위만 정해짐. PlayerEngrave 수정 금지 |
| E6 | 주문서 강화 확률·효율·실패 보너스·최대 30단계는 **그대로**. 수요가 3칸으로 늘어 **드랍량만 올린다**: 일반 몬스터 3%→4%, 층 보스 1~2장→2~3장 | 마스터 스펙 "기존 유지, 밸런스 문제면 조정 가능" |
| E7 | 잠재능력 등급 4개: **레어(1)·에픽(2)·유니크(3)·레전더리(4)**. 모든 칸은 처음부터 **레어 + 3줄**. 색은 기존 `GradeTable` 색 | "미개방" 상태를 없애 화면·저장이 단순 |
| E8 | 큐브 **4종**(사용자 확정): **무료**(흔한 아이템 큐브. 레드보다 등급 상승 확률 낮음·옵션 확률은 레드와 같음·즉시 적용) / **레드**(즉시 적용, 등급 상승) / **블랙**(비포/애프터 선택) / **플래티넘**(첫 줄 잠금, 2·3번째 줄만 새로, 비포/애프터 선택, 등급 상승 없음). 종류별 규칙은 새 표 `GearCubeTable`로 두고 굴림 함수는 **일반형 하나**(등급 상승표·천장 점수·잠글 줄 수·결과 후보 수·대기 저장 여부를 받음) | 수치가 바뀌어도 표만 고친다 |
| E9 | 등급 상승 확률 — 무료: 5% / 1.5% / 0.5%(레드의 절반). 레드: 레어→에픽 10% / 에픽→유니크 3% / 유니크→레전더리 1%. 블랙: 20% / 6% / 2%. 플래티넘: 상승 없음. **천장은 점수제**: 현재 등급에서 쓴 큐브의 천장 점수(무료 1, 레드·블랙 2, 플래티넘 0)가 30 / 100 / 300점에 닿으면 그 큐브로 확정 상승 → 레드·블랙만 쓰면 15 / 50 / 150번째, 무료만 쓰면 2배인 30 / 100 / 300번째. 섞어 써도 점수가 합쳐진다. 등급 하락 없음 | 무료 천장 2배(메인 세션 결정)를 한 횟수 값으로 처리 |
| E10 | 줄 등급: 1번째 줄 = 장비 등급. 2·3번째 줄은 각자 레드 20% / 블랙 30% 확률로 장비 등급, 아니면 한 등급 아래(레어 장비는 모두 레어) | 메이플식 "윗잠 줄" |
| E11 | 대기 저장 큐브(블랙, 플래티넘) 결과는 **대기 결과로 저장**(재접속해도 남음). 대기 중엔 그 칸에 큐브를 또 쓸 수 없고 선택해야 한다. 판에는 항상 **적용된 옵션**만 반영 | 로드맵 5-D |
| E12 | **망토 = 공용 스킬 칸**. 옵션 풀은 칸 스킬 피해(주력기·보조기·기술·하이퍼·궁극기)와 궁극기 게이지 충전만. **직업 고유 옵션 없음** | 사용자 확정. 칸 장비라 어느 직업이 들어와도 통해야 한다 |
| E13 | 큐브 획득 — 무료(가장 흔함): **스테이지 클리어 50% 확률 1개**, **층 보스 처치 3개**, 첫 지급 30. 레드: **층 보스 처치 2개**, 대기열 정예 처치 25% 확률 1개, 첫 지급 20. 블랙: **층 첫 클리어 10층마다 1개**, **별 구간 마지막 상자 3개**(소급), 첫 지급 3. 플래티넘(드묾): **별 구간 마지막 상자 1개**(소급), 첫 지급 1. **다이아 구매** 레드 30 / 블랙 100 / 플래티넘 200(무료는 구매 없음). 하루 제한 없음 | 기존 보상 경로 재사용. 첫 지급 플래티넘 1은 사용자가 기능을 바로 써 보게 하기 위함 |
| E14 | 판 반영: **편성 칸 번호 = 장비 번호**(1칸 = 메인 플레이어, 2칸 = 동료 1, 3칸 = 동료 2). 장비 수치 배율(1 + 0.1 × 주문서 단계)과 잠재능력은 **구성원마다 자기 칸 값**. 파티 공용 효과(정비소 강화 보너스·명중·단계 고유 효과)는 **1칸 기준** | 단순: 3칸 비교·평균 없이 한 칸만 본다. 1칸은 메인 캐릭터라 화면의 판 장비 표시(메인 기준)와도 맞는다 |
| E15 | 신발이 주던 **치명타 데미지**는 판 안 **장갑의 두 번째 능력치**로. 신발 단계 보너스 회피 3%는 **망토 30단계**(죽은 `ExpPct` 자리)로, `EvadeCounter`는 없앤다 | 신발 삭제로 전투력이 빠지지 않게 |
| E16 | UI는 **성장창 1번 탭("슬롯 강화" → "장비") 재사용**: 아래쪽 버튼 **"1칸/2칸/3칸" 3개**(각 버튼에 지금 그 칸 캐릭터 이름), 오른쪽 위 `주문서 강화 / 잠재능력` 전환, 잠재능력 모드는 묶음 `Pot`. 새 `.ui` 파일 없음 | 사용자 확정 |
| E17 | 칸은 항상 3개 모두 사용 가능(잠금 없음). 칸 버튼은 요청 처리 중에만 Disabled | 칸은 직업 해금과 무관 |
| E18 | 단위 7개(5-A~5-G). 앞 단위만 끝나도 게임이 돈다 | 1절 |

---

## 1. 작업 단위와 순서

| 단위 | 내용 | 끝나면 게임 상태 | 확인 |
|---|---|---|---|
| **5-A** | 데이터 표 4개 + `PlayerGear` 골격: 저장·불러오기·기존 `OrbisSlot` 이전·첫 지급·일반형 잠재 굴림(서버 내부)·디버그. UI 없음 | 장비 데이터가 새 키에 생김. 게임은 아직 옛 슬롯으로 동작 | F1~F6 |
| **5-B** | 주문서 강화 칸별 전환: `PlayerGear.RequestScroll`, 탑 주문서 드랍·저장을 `PlayerGear`로, 드랍 수치 조정. 성장창 1번 탭 = "장비", 칸 버튼 3개, 신발 칸 숨김 | 로비에서 칸별 주문서 강화 | F7~F11 |
| **5-C** | 판 반영 1: `GearRunData`로 칸별 단계, 구성원별 장비 배율, 신발 제거, 장갑 치명타 데미지, 파티 공용 효과 = 1칸 | 판이 칸별 단계로 계산 | F12~F15 |
| **5-D** | 큐브 서버: 4종 사용·대기 결과·선택, 등급 상승·천장, 다이아 구매, 큐브 획득, 분포 모의 | 서버에서 잠재능력 완성 | F16~F22 |
| **5-E** | 잠재능력 UI: 모드 전환, `Pot` 묶음, 비포/애프터·후보 선택, 칸 버튼 등급 표시 | 로비에서 큐브 사용 | F23~F27 |
| **5-F** | 판 반영 2: 잠재능력 합계를 구성원 보너스 경로에, 칸 스킬 피해·궁극기 게이지 훅, 결과 팝업 큐브 표시 | 잠재능력이 전투에 반영 | F28~F32 |
| **5-G** | 회귀: 저장 복원·이전 1회·옛 진입점 0·1층 판 1회 | 5단계 완료 | F33~F36 |

- 단위마다 `maker_run_command stop` → `maker_refresh_workspace` → `maker_build_logs` 오류 0 → `maker_clear_logs` → 플레이 → 그 단위 확인 항목. **다른 작업이 메이커를 쓰고 있으면 기다리거나 정적 검사만 하고 보고**한다.
- 확인은 **기능이 동작하는지만 짧게**. 확률은 서버 모의(`DebugCubeSim`) 로그 한 번으로 본다. 밸런스 실측 없음.

---

## 2. 현재 구조 조사 결과

### 2-1. 장비 관련 두 층

| 층 | 위치 | 저장 | 내용 |
|---|---|---|---|
| **판 안 장비 (정비소·메인 패널 장비 탭)** | `Tower/TowerRunManager.mlua` `EquipTier / EquipEnh / EquipCost / EquipUnlocked`(Sync), `RequestEnhance`, `EquipStat`, `CalcEquipCost`, `LoadEquipTable`. 표 `EquipTable.csv`, `EquipSlotTable.csv` | 판마다 초기화 | 메소로 0~10강. **파티 공용 6부위** |
| **계정 슬롯 강화 (주문서)** | `Engrave/PlayerEngrave.mlua` 1166~1335행: `SlotLv / SlotBonus / Scrolls`, `LoadSlots`, `EncodeSlots`, `AddScrolls`, `FlushSlots`, `SlotRunData`, `RequestSlotUpgrade`. 표 `SlotScrollTable`, `SlotRateTable`, `SlotMilestoneTable` | `OrbisSlot` JSON `{"v":1,"lv":{부위:단계},"b":{부위:추가확률},"s":{"부위_단계":장수}}` | **계정 공용 6부위** 0~30단계 |

- 주문서 강화 공식: 같은 주문서 1~3장, 확률 = `SlotRateTable(단계) × 효율 × 장수 + 누적 추가 확률`(최대 100). 실패 시 추가 확률 += 시도 확률 × 0.1(소수 둘째 자리), 성공 시 0. 로비에서만.
- 판 반영(`StartRun`): `engrave:SlotRunData()` → `_T.slotInfo`(부위별 `lv, enhRate, costDown, startEnh, doubleEnh`) + `_T.slotBonus`(단계 고유 효과). `SlotLevels[slot]`이 `EquipStat`의 배율 `1 + SlotStatStep(0.1) × 단계`.
- 단계 보너스(`SlotMilestoneTable`): 5 정비소 강화 확률 +5%, 10 비용 −10%, 15 부위 고유, 20 판 시작 +2강, 25 이중 강화 10%, 30 부위 고유.
- 주문서 드랍(`DropScrolls`): 층 보스·정예 1~2장, 일반 몬스터 `3% + …`. `eng:AddScrolls` → 판 끝 `eng:FlushSlots()`. 디버그 판은 지급 안 함.
- 판 능력치(`RecalcStats`): 구성원마다 `CharRunData.atk/hp/def × 계정% + 무기/투구/갑옷 EquipStat`(공용). 장갑 → `ActorCrit`, 신발 → `CritDmg`, 망토 → `Accuracy`.
- 편성 칸 순서: `TowerRunManager` 630행 `_T.mainJob = jobs[1]`, `_T.mateJobs = { jobs[2], jobs[3] }` (`RunPartyData` 순서). `MateIndex(actor)` = 1·2. → **칸 번호 = 플레이어 1, 동료 `MateIndex + 1`**.
- 구성원별 보너스 경로: `GoddessBonus(actor, eff)` → `AccessoryBonus(actor, eff)`. 이미 구성원별로 연결된 효과: `AtkPct, DefPct, HpPct, Crit, CritDmg, Speed, SkillDmg, CdReduce, EliteDmg, DmgTaken(=DmgReduce), LifestealPct, Undying`.

### 2-2. 신발(`Shoes`)이 남아 있는 곳 (5-C에서 정리)

| 파일 | 위치 | 처리 |
|---|---|---|
| `Tower/TowerRunManager.mlua` | `_T.slots`, `cats`, `SlotKoName`, `CritDmg` | 목록에서 빼고 `CritDmg`는 장갑 두 번째 능력치로 |
| `Tower/TowerRunManager.mlua` | `CopyCostumeTo`의 `CustomShoesEquip` | **그대로**(외형) |
| `Tower/MainPanelLogic.mlua` | 장비 줄 목록·이름 | 목록에서 빼고 `Shoes` 줄 숨김 |
| `Slot/SlotUILogic.mlua` | `statName` | 5-B에서 다시 씀 |
| `Character/PlayerRoster.mlua` | 외형 부위 | **그대로**(외형) |
| `Data/EquipSlotTable.csv` | Shoes 줄 | 삭제(5-B) |
| `Data/EquipTable.csv`, `SlotScrollTable.csv`, `SlotMilestoneTable.csv` | Shoes 줄 | **남긴다**(PlayerEngrave가 계속 읽음. 새 코드는 걸러 낸다) |

### 2-3. 성장창 1번 탭 현재 배치 (`ui/GrowthUI` `Root/Slot`, 1920×1080, 중심 기준)

| 엔티티 | pos | 크기 | 비고 |
|---|---|---|---|
| `Avatar` | -584,-47 | 868.8×1158.4 | 실루엣 배경 |
| `Slot_Cap` / `Slot_Cape` / `Slot_Coat` / `Slot_Glove` / `Slot_Weapon` / `Slot_Shoes` | -586,264 / -803,-42 / -586,-222 / -803,-302 / -364,-302 / -586,-452 | 138×138 | 부위 버튼, 자식 `Icon`·`Selected` |
| `Panel` | 315,-95 | 1050×810 | 오른쪽 본문 |
| `Preview` (+`Icon`) / `Fx1`·`Fx2` | -32,134 | 190×190 / 300×200 | |
| `Info` | 469,135 | 640×250 | |
| `SetHead` / `Div1` | 1,-50 / 315.5,-72 | 300×30 / 929×2 | |
| `CardLabel`, `Card1~3` | -19,-94 / -70·104·278,-193 | 260×28 / 158×151 | |
| `QtyLabel`, `Qty1~3` | 518,-94 / 450·584·718,-155 | 260×28 / 124×74 | |
| `Div2` / `RateTitle` / `Rate` / `RateDetail` / `RateBonus` | 315.5,-285 / -57,-313 / 145,-310 / 133,-372 / 133,-404 | | |
| `Have` / `Upgrade` / `Msg` | 585,-222 / 620,-403 / 530,-340 | 390×30 / 320×70 / 500×28 | |
| `Root/Tab1~5` | y 400 | 368×76 | 1번 탭 글자 "슬롯 강화" |

---

## 3. 데이터 구조 (5-A)

### 3-1. 저장 키 `OrbisGearV1` (문자열, `PlayerRoster`와 같은 `키=값;` 형식)

```
v=1;mig=1;gift=1;cube=F:0,R:20,B:3,P:0;sc=Weapon_1:3,Cap_2:1;bf=20;bs=1,2;g=<칸목록>;p=<대기목록>;<모르는 키 보존>
```

| 키 | 값 | 설명 |
|---|---|---|
| `v` | `1` | 형식 버전 |
| `mig` | `1` | `OrbisSlot` 이전 완료. 있으면 다시 읽지 않는다 |
| `gift` | `1` | 첫 지급 완료 |
| `cube` | `종류:개수,...` | 큐브 보유(`GearCubeTable` 순서). 모르는 종류는 원문 보존 |
| `sc` | `부위_단계:장수,...` | 주문서 보유(0은 생략). 5부위만 |
| `bf` | 정수 | 블랙 큐브 첫 클리어 보상을 계산한 최고 층 |
| `bs` | `구간,...` | 블랙 큐브를 받은 별 구간 |
| `g` | `칸/부위:단계:추가확률:등급:천장점수:줄1,줄2,줄3` 를 `\|`로 | 15칸(칸 1~3 × 5부위). `추가확률`은 소수(예 `2.5`). `천장점수` = 현재 등급에서 쌓은 천장 점수. 줄 = `옵션Id@줄등급`(예 `AtkPct@2`) |
| `p` | `칸/부위:종류:등급:천장점수:후보1~후보2...` 를 `\|`로 | 대기 결과(없으면 생략). 후보는 `줄1,줄2,줄3`, 후보 사이는 `~`(지금은 블랙·플래티넘 모두 후보 1개. 표 `Choices`가 늘면 쓰는 자리) |

- 표에 없는 칸 번호·부위의 칸, 모르는 큐브 종류는 원문 그대로 보관해 다시 저장한다(`PlayerRoster`의 `unknownChars` 방식). 모르는 옵션 Id 줄은 **그대로 보관하되 합계에서 0**.
- 저장에 없는 칸은 **단계 0·추가확률 0·레어 + 3줄 새로 굴림**으로 만들고 저장한다.

### 3-2. `PlayerGear.mlua` 속성 (서버가 쓰고 클라이언트가 읽음)

| 속성 | 형식 | 내용 |
|---|---|---|
| `@Sync GearLoaded` | boolean | 불러오기 완료 |
| `@Sync CubeData` | string | `cube`와 같은 형식(표의 4종 전부, 0 포함) |
| `@Sync ScrollData` | string | `sc`와 같은 형식(0 포함 전체 15개: 5부위×3단계) |
| `@Sync GearData` | string | `g`와 같은 형식(칸·부위 순서, 모르는 칸 제외) |
| `@Sync PendData` | string | `p`와 같은 형식 |
| `@Sync GearSeq` / `GearMsg` / `GearResult` | integer / string / string | 요청 처리마다 +1, 결과 코드, 상세(`scroll:2:Weapon:1:13` / `cube:R:2:Weapon:2:1` = 종류·칸·부위·등급·상승여부 / `pend:...` / `choose:...` / `buy:R:1`) |
| `GearKey` | `"OrbisGearV1"` | |
| `OldSlotKey` | `"OrbisSlot"` | 이전용 읽기 전용 |

- 양쪽 공용 파싱: `ParseGear(str)` → `{ [칸] = { [부위] = { lv, bonus, grade, cnt, lines = {{id, g}, ...} } } }`, `ParseScrolls(str)`, `ParseCubes(str)`, `LineValue(part, id, g)`, `LineLabel(part, id, g)`, `PotSum(slotNo)`(효과별 합계).
- 칸 수는 `GearConfig.PartySlots`(3). 내부 상태는 `self._T`.

### 3-3. 새 표 4개 (`RootDesk/MyDesk/Data/`, UTF-8 BOM csv + 같은 이름 `.userdataset`)

**`GearGradeTable.csv`**

```
Grade,Name,ColorHex,FreeUp,RedUp,BlackUp,Ceiling,RedPrime,BlackPrime,#Desc
1,레어,5BC0EB,5,10,20,30,0,0,다음 등급(에픽) 상승 확률·천장 점수
2,에픽,9B59D0,1.5,3,6,100,20,30,
3,유니크,F5A623,0.5,1,2,300,20,30,
4,레전더리,7ED321,0,0,0,0,20,30,최고 등급
```
- `FreeUp/RedUp/BlackUp` = 이 등급에서 다음 등급 상승 확률(%) 세트. `Ceiling` = 천장 점수(이 점수에 닿는 큐브는 확정 상승, 0 = 없음). `RedPrime/BlackPrime` = 2·3번째 줄이 장비 등급일 확률(%) 세트.

**`GearCubeTable.csv`** — 큐브 종류별 규칙. 굴림 함수는 이 행만 보고 동작한다.

```
Kind,Name,IconRUID,UpSet,PrimeSet,CeilPt,LockLines,Choices,Pending,DiaPrice,#Desc
F,무료 큐브,358fb90997e84243992bdba43cbf333d,Free,Red,1,0,1,0,0,흔한 큐브·즉시 적용·상승 확률 레드의 절반·천장 2배
R,레드 큐브,009913f229524a09828296535165b2df,Red,Red,2,0,1,0,30,즉시 적용·등급 상승
B,블랙 큐브,1edec01f822c422293f62729e7462a22,Black,Black,2,0,1,1,100,비포/애프터 선택·등급 상승 높음
P,플래티넘 큐브,98ad6a59b9d345cbab8a60d64e3cda5c,None,Black,0,1,1,1,200,첫 줄 잠금·2·3번째 줄만·비포/애프터 선택·등급 상승 없음
```
- `UpSet` = 등급 상승표(`Free`/`Red`/`Black` → `GearGradeTable`의 `FreeUp`/`RedUp`/`BlackUp`, `None` = 상승 없음). `PrimeSet` = 2·3번째 줄 표(`Red`/`Black`). `CeilPt` = 1회 사용 시 쌓이는 천장 점수(0 = 천장과 무관). `LockLines` = 앞에서부터 바꾸지 않는 줄 수. `Choices` = 결과 후보 수(지금 모두 1). `Pending` = 1이면 대기 결과로 저장하고 이전 유지/새 옵션 중 선택. `DiaPrice` = 1개 다이아 가격(0 = 구매 불가).
- 잠긴 줄이 있으면 등급도 바뀌지 않는다(1번째 줄 = 장비 등급). 플래티넘 2·3번째 줄 등급은 현재 장비 등급 기준으로 `BlackPrime`(30%)을 쓴다(상위 큐브라 블랙과 같게, 메인 세션 위임 판단).

**`GearPotTable.csv`** — 부위별 옵션 풀. 값 열 `R/E/U/L` = 줄 등급 1~4일 때 수치. `Weight` = 같은 부위 안 가중치.

```
Slot,OptId,Weight,R,E,U,L,Label
Weapon,AtkPct,10,3,6,9,12,공격력 +{v}%
Weapon,SkillDmg,10,4,8,12,16,스킬 피해 +{v}%
Weapon,EliteDmg,8,5,10,15,20,정예·보스 피해 +{v}%
Weapon,CritDmg,8,4,8,12,16,크리티컬 데미지 +{v}%
Cap,HpPct,10,4,8,12,16,최대 체력 +{v}%
Cap,DefPct,8,4,8,12,16,방어력 +{v}%
Cap,CdReduce,6,2,4,6,8,스킬 재사용 대기시간 -{v}%
Cap,DmgReduce,6,2,4,6,8,받는 피해 -{v}%
Coat,HpPct,10,4,8,12,16,최대 체력 +{v}%
Coat,DefPct,10,5,10,15,20,방어력 +{v}%
Coat,DmgReduce,8,2,4,6,8,받는 피해 -{v}%
Coat,LifestealPct,5,0.5,1,1.5,2,준 피해의 {v}% 회복
Glove,Crit,10,2,4,6,8,크리티컬 확률 +{v}%
Glove,CritDmg,10,4,8,12,16,크리티컬 데미지 +{v}%
Glove,Speed,8,3,6,9,12,공격 속도 +{v}%
Glove,AtkPct,6,2,4,6,8,공격력 +{v}%
Cape,MainDmg,10,5,10,15,20,주력기 피해 +{v}%
Cape,SubDmg,8,5,10,15,20,보조기 피해 +{v}%
Cape,TechDmg,8,5,10,15,20,기술 피해 +{v}%
Cape,HyperDmg,6,8,16,24,32,하이퍼 스킬 피해 +{v}%
Cape,UltDmg,6,8,16,24,32,궁극기 피해 +{v}%
Cape,UltGainPct,6,5,10,15,20,궁극기 게이지 충전 +{v}%
```
- 22행. 직업 열 없음(망토도 공용 옵션만). 같은 옵션이 여러 줄에 나와도 합산. 효과 Id는 판에서 그대로 효과 이름(5-F).
- 근거: 무기 레전더리 3줄 평균 ≈ 공격력 30% 남짓 — 장신구 한두 개 크기. 사용자 플레이 뒤 표만 고친다.

**`GearConfig.csv`** (`Key,Value,#Desc`, 19행)

| Key | Value | 설명 |
|---|---|---|
| `PartySlots` | 3 | 편성 칸 수(장비 세트 수) |
| `PotLines` | 3 | 잠재능력 줄 수 |
| `ScrollMaxLevel` | 30 | 주문서 최대 단계 |
| `ScrollMaxCount` | 3 | 한 번에 쓰는 주문서 최대 장수 |
| `ScrollFailRatio` | 0.1 | 실패 시 쌓이는 비율 |
| `StartCubeRed` / `StartCubeBlack` | 20 / 3 | 첫 지급(새 계정·기존 계정 모두 1회) |
| `ShoesLvToRed` | 2 | 이전: 신발 단계 1당 레드 큐브 |
| `ShoesLvPerBlack` | 5 | 이전: 신발 단계 5당 블랙 큐브 1 |
| `FirstClearBlackEvery` | 10 | 이 층 배수를 처음 깰 때마다 블랙 1 |
| `SectionBlack` | 3 | 별 구간 마지막 상자마다 블랙 |
| `SectionLastTier` | 3 | 마지막 상자 단계(PlayerRoster와 같은 값) |
| `BossCubeRed` | 2 | 층 보스 처치 레드(5-D에서 사용) |
| `EliteCubeRedChance` | 25 | 대기열 정예 처치 레드 확률 %(5-D에서 사용) |
| `StartCubeFree` / `StartCubePlatinum` | 30 / 1 | 첫 지급 무료·플래티넘 |
| `StageFreeChance` | 50 | 스테이지 클리어 시 무료 큐브 1개 확률 %(5-D) |
| `BossCubeFree` | 3 | 층 보스 처치 무료 큐브(5-D) |
| `SectionPlatinum` | 1 | 별 구간 마지막 상자마다 플래티넘(5-D, 소급) |

(다이아 가격은 `GearCubeTable.DiaPrice`로 옮겼다.)

### 3-4. 기존 표 수정

| 표 | 수정 | 단위 |
|---|---|---|
| `EquipSlotTable.csv` | `Shoes` 줄 삭제, `Coat`의 `SlotName` `옷` → `갑옷` | 5-B |
| `EquipTable.csv` | 열 `Stat2,Stat2Value` 추가. **공용 `Glove` 10줄만** `CritDmg` + 같은 Tier 신발 값(3.2 … 32). Shoes 줄은 남김 | 5-C |
| `SlotMilestoneTable.csv` | `Cape,30,ExpPct,10,...` → `Cape,30,Evade,3,회피 +3%` | 5-C |
| `RunConfig.csv` | `ScrollChance` 3→4, `ScrollBossMin` 1→2, `ScrollBossMax` 2→3 | 5-B |

---

## 4. 규칙 상세

### 4-1. 기존 슬롯 이전 (5-A, 처음 1회)

`PlayerGear.LoadGear()`:
1. `OrbisGearV1`을 읽는다. 값이 있으면 `Decode` → `mig=1`이면 끝. (실패 코드면 `LoadRetryDelay` 뒤 재시도, 그동안 **저장 금지**)
2. 값이 없으면(1000002 또는 빈 값) `OrbisSlot`을 **직접 읽는다**(`ds:GetAndWait(self.OldSlotKey)`, `JSONDecode`). 읽기 실패면 재시도(저장 금지), 없으면 빈 표.
3. 칸 1~3 × 5부위: `lv = 옛 lv[부위]`(0~30), `bonus = 옛 b[부위]`, 레어, 횟수 0, 3줄 새로 굴림. **세 칸 모두 같은 단계**.
4. 주문서: 5부위 `옛 s["부위_단계"]` 그대로. **신발 주문서** `Shoes_t` n장 → 같은 단계 주문서를 `Weapon, Cap, Coat, Glove, Cape` 순서로 1장씩 돌아가며(n=7이면 2,2,1,1,1).
5. **신발 단계** L → 레드 `L × ShoesLvToRed`, 블랙 `floor(L / ShoesLvPerBlack)`. 신발 추가 확률은 버림.
6. 첫 지급(`gift`가 없으면) 무료 `StartCubeFree`, 레드 `StartCubeRed`, 블랙 `StartCubeBlack`, 플래티넘 `StartCubePlatinum`, `gift=1`.
7. `mig=1`로 **즉시 저장**(`SetAsync`, 실패 시 재시도). 로그 `[VRF] gear migrated slots=3 lv=Weapon:3,Cap:0,... shoesLv=L cube=F:30,R:..,B:..,P:1 scrolls=..`.
- `OrbisSlot`은 지우지 않는다. 5-B 뒤엔 아무도 `AddScrolls/RequestSlotUpgrade/FlushSlots`를 부르지 않아 다시 저장되지 않는다.

### 4-2. 주문서 강화 (5-B)

`RequestScroll(integer slotNo, string part, integer tier, integer count)` (`@ExecSpace("Server")`):
- 조건: 본인·불러오기 완료·**로비**·칸 1~3·5부위·단계 1~3·장수 1~`ScrollMaxCount`·단계 < 30·보유 ≥ 장수. 안 맞으면 `Reply("err_*")`(`err_lobby / err_max / err_scroll / err_arg`).
- 확률 = `min(100, SlotRateTable(단계) × SlotScrollTable 효율 × 장수 + 추가확률)`. 성공: 단계 +1, 추가확률 0. 실패: 추가확률 += `floor(rate × ScrollFailRatio × 100 + 0.5) / 100`.
- 저장, `Reply("ok", "scroll:2:Weapon:1:4")`, 로그 `[VRF] gear scroll s2 Weapon tier=1 x2 rate=50 ok=true lv=4 bonus=0`.
- 판 중 드랍: `AddScrolls(part, tier, n)` = 보유만 늘리고 `dirty`(저장은 `Flush()`).

### 4-3. 잠재능력 굴림 (5-A 일반형, 5-D에서 큐브 연결)

`RollPot(part, grade, cnt, curLines, rule)` → `{ grade, cnt, up, ceil, cands = { 줄목록, ... } }`. `rule` = `{ up = "Free"|"Red"|"Black"|"None", prime = "Red"|"Black", ceilPt = n, lock = n, choices = k, pending = bool }` (`CubeRule(kind)`이 `GearCubeTable` 행으로 만든다. 처음 생성은 `NewRule()` = `{ up = "None", prime = "Red", ceilPt = 0, lock = 0, choices = 1 }`).
1. 등급 상승(`up ~= None`, `lock == 0`, grade < 4): `cnt + ceilPt`. `rand×100 < Up[grade]` 또는 `Ceiling[grade] > 0 and cnt >= Ceiling[grade]`(천장)이면 `grade + 1`, `cnt = 0`. `up == None`이면 `cnt`도 그대로.
2. 후보 k개: 앞 `lock`줄은 `curLines` 그대로, 나머지 줄은 — 줄 등급: 1번째 = 새 등급, 2·3번째 = 새 등급이 1이면 1, 아니면 `rand×100 < Prime[새 등급]`이면 새 등급, 아니면 −1. 옵션: `GearPotTable`에서 `Slot == part`인 행을 `Weight` 가중 무작위.
- `Pending == 0`: 후보 1을 바로 적용(등급·횟수·줄). `Pending == 1`: `pend[칸/부위] = { kind, grade, cnt, cands }`, 현재 값은 그대로.
- 선택 `RequestCubeChoose(slotNo, part, pick)`: `pick` 0 = 이전 유지(등급·줄 유지, 천장 점수는 쌓인 채로: 대기 결과가 상승하지 않았으면 대기 결과의 점수, 상승한 결과를 버리면 이전 점수 + `CeilPt`), 1~k = 그 후보 적용. 대기 삭제·저장.
- 천장 표기: "천장 n / Ceiling점". 고른 큐브로 다음이 확정이면 "다음 큐브 등급 상승 확정".

### 4-4. 큐브 사용·구매 (5-D)

- `RequestCube(slotNo, part, kind)`: 조건 = 본인·로비·칸 1~3·5부위·표에 있는 kind·보유 ≥ 1·**그 칸에 대기 결과 없음**(`err_pending`). 1개 소모 → `RollPot(…, CubeRule(kind))` → 즉시 적용 / 대기 → 저장 → `Reply("ok", "cube:R:2:Weapon:2:1")` / `"pend:B:2:Weapon:2:1"`. 로그 `[VRF] gear cube R s2 Weapon 1->2 up=true cnt=0 lines=AtkPct@2,SkillDmg@1,AtkPct@2`.
- `RequestBuyCube(kind, n)`: n 1 또는 10, 로비, `DiaPrice > 0`(무료 큐브는 0이라 구매 불가). `roster:SpendDiamonds(가격×n)`가 true일 때만 지급·저장. 결과 `buy:R:1`. 부족 `err_dia`.
- 무료 큐브도 보유 수량을 소모한다. 하루 제한 없음.
- 다른 칸에는 대기 결과와 상관없이 큐브를 쓸 수 있다. 판 시작·탑 이동은 막지 않는다.

### 4-5. 큐브 획득 (5-D)

| 경로 | 위치 | 규칙 |
|---|---|---|
| 층 보스 처치 | `TowerRunManager` 보스 처치 분기(`DropScrolls(bossKilled)` 옆) | `DropCubes("boss")`: 레드 `BossCubeRed`(2) + 무료 `BossCubeFree`(3) → `gear:AddCubes`(저장은 판 끝 Flush), `RunCubes`/`RunFreeCubes` 증가 |
| 스테이지 클리어 | 스테이지 클리어 처리 지점(5-D에서 찾는다) | `StageFreeChance`% 확률 무료 1 |
| 대기열 정예 처치 | `DropScrolls(qElite)` 옆 | `EliteCubeRedChance`% 확률 레드 1 |
| 층 첫 클리어 | `PlayerGear.OnUpdate` 1초마다(PlayerEngrave **읽기만**) | `eng.ClearedFloor > bf`면 `bf+1 ~ ClearedFloor` 중 `FirstClearBlackEvery` 배수 층 수만큼 블랙, `bf = ClearedFloor`, 저장. 소급 |
| 별 구간 마지막 상자 | 같은 1초 확인 | `eng.SectionData` 마지막 상자 비트가 켜진 구간 중 `bs`에 없는 구간마다 블랙 `SectionBlack`(3) + 플래티넘 `SectionPlatinum`(1). 소급 |
| 다이아 구매 | 장비 탭 버튼 | 4-4 |
| 디버그 판 | — | `_T.debugRun`이면 지급하지 않음 |

- 판 끝 저장: 기존 `eng:FlushSlots()` 두 곳을 `gear:Flush()`로 바꾼다.
- 결과 팝업: `TowerRunManager` `@Sync RunCubes` → `ResultPopupLogic` 주문서 옆에 `{ icon = 레드 큐브 아이콘, name = "큐브", n = RunCubes }`.

### 4-6. 판 반영 (5-C 주문서 / 5-F 잠재능력)

`PlayerGear:GearRunData()` (`ServerOnly`):
```
{
  slots = { [1] = { lv = { Weapon = n, ... }, pot = { AtkPct = 합, ... } }, [2] = ..., [3] = ... },
  party = { slots = { [부위] = { lv = 1칸 단계, enhRate, costDown, startEnh, doubleEnh } },
            bonus = { [효과] = 합 } }   -- SlotMilestoneTable을 1칸 단계 기준으로 (PlayerEngrave.SlotRunData와 같은 분기)
}
```

`TowerRunManager` 수정(5-C):
1. `StartRun`의 `engrave:SlotRunData()` → `gear:GearRunData()`. `_T.gearRun = 결과`, `_T.slotInfo = party.slots`, `_T.slotBonus = party.bonus`. 없거나 `not GearLoaded`면 빈 값 + `log_warning("[VRF] gear run data missing")`.
2. `SlotLevels[slot]`(Sync, 패널 표시) = **1칸** 단계.
3. 새 `ActorSlotNo(actor)`: 플레이어 → 1, 동료 → `MateIndex(actor) + 1`, 그 밖 → 1.
4. `EquipStat(slot)` → `EquipStatFor(slot, no)`: `row.value × (1 + 0.1 × 강) × (1 + SlotStatStep × gearRun.slots[no].lv[slot])`. 새 `EquipStat2For(slot, no)`: 같은 배율로 `row.value2`(장갑 `CritDmg`).
5. `RecalcStats`: 무기·투구·갑옷을 구성원마다 `EquipStatFor(.., ActorSlotNo)`. `CritDmg`(Sync) = `BaseCritDmg + EquipStat2For("Glove", 1) + Bonus("CritDmg")`. `Accuracy` = `EquipStatFor("Cape", 1) + Bonus("Acc")`.
6. `ActorCrit(actor)` → `EquipStatFor("Glove", ActorSlotNo(actor))`. `ActorCritDmg(actor)` → `BaseCritDmg + EquipStat2For("Glove", no) + Bonus("CritDmg") + AccessoryBonus(actor, "CritDmg")`.
7. `LoadEquipTable`·`_T.slots`·`cats`·`SlotKoName`에서 `Shoes` 제거(`Coat` 이름 `갑옷`). `Stat2Value`를 `value2`로.
8. `DropScrolls`의 `eng:AddScrolls` → `gear:AddScrolls`.
9. `MainPanelLogic.SetupEquipRows`에서 `Shoes` 제거 + 줄 엔티티 enable false(필요하면 `Cape` 줄을 그 자리로 UIBuilder `patch`). 장갑 줄 글자 `크리티컬 확률 +x% · 데미지 +y%`.

`TowerRunManager` 수정(5-F):
1. 새 `PotBonus(actor, key)` = `gearRun.slots[ActorSlotNo(actor)].pot[key] or 0`.
2. `AccessoryBonus(actor, effect)`: `key`·`sign`을 정한 직후, `acc == nil` 조기 반환보다 **앞에서** 잠재능력 값을 구해 마지막에 더한다: `(장신구 합 + PotBonus(actor, key)) × sign`. `DefPct`·`HpPct`·`Speed`·`Crit`이 실제로 이 경로를 거치는지 grep으로 확인하고, 안 거치면 그 계산에 더한다.
3. 칸 스킬 피해 `MainDmg/SubDmg/TechDmg/HyperDmg/UltDmg`: 전투 3종(`HeroCombat`·`BishopCombat`·`JobCombat`)의 스킬 피해 배율 자리에 `mgr:PotBonus(self.Entity, 칸키 .. "Dmg")`(`CharSlotTable.SlotKey`). 기본 공격·패시브 추가타 제외.
4. `UltGainPct`: 세 파일 `AddUlt(n)`에서 `n × (1 + PotBonus("UltGainPct") / 100)`. 처음 적용 시 `[VRF] pot hook UltGainPct slot=.. v=..`.
5. 판 시작 로그: `[VRF] gear run s1 lv=W3,C0,A0,G1,K0 pot=AtkPct:9,Crit:4 | s2 ... | s3 ...`.

---

## 5. 화면 (성장창 1번 탭 재사용)

### 5-1. 공통

- 빌더: `.claude/skills/msw-ui-system/scripts/msw_ui_builder.cjs`, 경로는 node 안에서 `'ui/'+'GrowthUI'+'.'+'ui'`로 조립. 만들기 전 `Docs/UI-Design-System.md`, `builder-protocol.md` + `builder-protocol-ui.md` §3을 읽는다.
- 새 버튼은 `ButtonComponent`를 `Transition: 2` + `ImageRUIDs`(manifest `output/ui/growth_neutral/manifest.json`)로 패치. 글자는 버튼 자체 텍스트, 가로 Center·세로 Middle. 런타임은 `_GrowthUILogic:StyleButton` 또는 `_UIButtonLogic:UseTheme / SetSelected / SetDisabled`.
- 글자색: 일반 `Color(0.14,0.17,0.21)`, 보조 `Color(0.36,0.40,0.46)`, 유리 `Color(0.20,0.48,0.12)`, 부족 `Color(0.78,0.20,0.20)`. 등급 이름은 등급 색 태그.
- 부위 아이콘은 **어두운 실루엣 그대로**. 큐브 아이콘은 `GearCubeTable.IconRUID`(사용자 제공 4종).

### 5-2. 5-B에서 바꾸는 것 (주문서 모드)

| 대상 | 변경 |
|---|---|
| `Root/Tab1` 글자 | `슬롯 강화` → `장비` |
| `Root/Slot/Slot_Shoes` | enable false |
| `Root/Slot/Party1~Party3` (새 button, button 계열 상태) | pos x = -804 / -584 / -364, y -455, 크기 200×64. 글자 = `1칸 · 히어로`(지금 그 칸 캐릭터의 현재 차수 이름, 22, BestFit 최소 16). 편성 원본은 `PlayerEngrave.MainJob/CompJob/Comp2Job`(읽기만), 이름은 `PlayerRoster:TierName`. 선택 = Selected 유지. 요청 중 Disabled. 기본 선택 = 1칸 |
| `Root/Slot/Slot_{부위}/Info` (새 text, 5개) | pos 0,-52, 130×28, 20, 가운데, Raycast false. `+12` (5-E부터 `+12 · 에픽`) |
| `Root/Slot/ModeScroll` / `ModePot` (새 button) | pos 560,272 / 740,272, 170×56, `주문서 강화` / `잠재능력`, 22. 5-B에서는 `ModePot` 비활성 |
| `Root/Slot/Info` | pos 469,120, 640×220 |
| `Info` 첫 줄 | `1칸(히어로) · 무기 슬롯  +3` |

`SlotUILogic` 수정:
- 데이터 원본 → 로컬 플레이어 `PlayerGear`(`GearData`, `ScrollData`, `GearSeq`). 강화 = `gear:RequestScroll(slotNo, part, tier, count)`. 결과 연출은 `GearSeq` 변화 + `GearResult`가 `scroll:`로 시작할 때.
- `_T.slotNo`(선택 칸). 칸 버튼 클릭 → `Refresh`. 칸 이름은 1초마다 또는 탭을 열 때 갱신(편성이 바뀌면 이름만 바뀌고 강화는 그대로).
- `statName`에서 `Shoes` 제거, 장갑 = `치명타 확률·데미지`. 요청 중에는 칸·모드 전환도 막는다.

### 5-3. 5-E에서 더하는 것 (잠재능력 모드)

`Root/Slot/Pot` (새 empty, 처음 enable false). 잠재능력 모드에서 켜고, 주문서 전용 엔티티(`CardLabel, Card1~3, QtyLabel, Qty1~3, Have, RateTitle, Rate, RateDetail, RateBonus, SetHead`)는 끈다.

| 경로 (`Root/Slot/` 아래) | 종류 | pos | 크기 | 값 |
|---|---|---|---|---|
| `Pot/Grade` | text | 315,-100 | 960×36 | `잠재능력  에픽`, 26, 굵게(등급 색) |
| `Pot/Before` (+`Title`, `Line1~3`) | sprite card Sliced | 60,-200 | 490×150 | `현재 옵션` + 3줄 |
| `Pot/After` (+`Title`, `Line1~3`) | sprite card Sliced | 570,-200 | 490×150 | 대기 결과가 있을 때만. **블랙·플래티넘이 같은 묶음**. 플래티넘이면 `Before/Line1`·`After/Line1`에 잠금 표시(앞에 `[잠금]` 글자 또는 24×24 자물쇠 아이콘 — 리소스는 5-E에서 찾고 없으면 글자) |
| `Pot/Cube1~4` | button(card) | 4종 가로 배치(-150·60·270·480, -345) | 200×72 | 자식 `Icon`(`GearCubeTable.IconRUID`, 56×56), `Name`, `Count`. 고른 큐브 Selected |
| `Pot/Buy` | button | 620,-415 | 230×52 | 고른 큐브 `다이아 30`/`100`/`200`. 무료(`DiaPrice` 0)면 숨김, 부족 Disabled |
| `Pot/Rate` | text | 620,-310 | 400×30 | `에픽 상승 10% · 천장 6 / 30점` / 플래티넘 `첫 줄 잠금 · 등급 상승 없음` / `최고 등급` |
| `Pot/Keep` / `Pot/Take` | button / button(action) | 470,-403 / 700,-403 | 200×70 / 240×70 | `이전 유지` / `새 옵션 적용`. 대기 결과 있을 때만 |

- `Upgrade` 글자: 주문서 `강화하기`, 잠재능력 `큐브 사용`. 비활성: 고른 큐브 0개 / 대기 결과 있음 / 요청 중.
- `Info`(잠재능력): `1칸(히어로) · 무기\n주문서 +3 · 장비 공격력 +30%\n잠재능력 합계: 공격력 +12% · 스킬 피해 +4%`.
- 결과 메시지: 등급 상승 `에픽으로 등급 상승!`(유리색) / `옵션이 바뀌었습니다` / 대기 `새 옵션을 확인하고 선택하세요` / 선택 후 `새 옵션을 적용했습니다` · `이전 옵션을 유지했습니다`. 2.5초 뒤 지움.
- 클라이언트 함수: `SetMode("scroll"|"pot")`, `RefreshPot()`, `UseCube()`, `Choose(pick)`, `Buy()`.

---

## 6. 작업 단위 상세

### 5-A. 데이터 + `PlayerGear` 골격

**바꿀 파일**: 새 `Data/GearGradeTable`, `Data/GearCubeTable`, `Data/GearPotTable`, `Data/GearConfig`(각 `.csv` + `.userdataset`), 새 `Character/PlayerGear.mlua`, `Global/DefaultPlayer.model`(ModelBuilder로 `script.PlayerGear` 추가 — `PlayerRoster`가 붙은 모델인지 먼저 확인).

1. 표 읽기 `LoadTables()`(양쪽 공용): 부위 = `EquipSlotTable` 순서(`Shoes` 거름), 주문서 효율 = `SlotScrollTable`, 기본 확률 = `SlotRateTable`, 단계 보너스 = `SlotMilestoneTable`, 새 표 4개.
2. `OnBeginPlay`(ServerOnly) → `LoadTables` → `LoadGear`(4-1). `OnUpdate`: 불러오기 재시도, 저장 실패 재시도, 1초마다 `CheckCubeRewards`(지금은 빈 함수). `OnEndPlay`: dirty면 `SetAndWait`.
3. `Decode / Encode / SyncGear / SaveGear / Reply / CanEdit`(PlayerRoster 방식).
4. `CubeRule(kind)`, `RollPot`(4-3 일반형), `PotSum(slotNo)`, `ParseGear`, `ParseScrolls`, `ParseCubes`, `LineValue`, `LineLabel`.
5. `AddScrolls(part, tier, n)`, `AddCubes(kind, n)`, `Flush()`.
6. 디버그(ServerOnly): `DebugGearSet(slotNo, part, lv, grade)`(줄 다시 굴림), `DebugGearLines(slotNo, part, "AtkPct@4,Crit@4,Crit@3")`, `DebugGiveCubes(kind, n)`, `DebugGiveScrolls(part, tier, n)`, `DebugGearDump()`, `DebugGearReset()`(**테스트 전용**: 새 계정처럼 다시 이전), `DebugRollOnce(slotNo, part, kind)`(저장하지 않고 굴림 결과만 로그), `DebugRollSim(n, part)`(옵션 분포 로그).

**완료 조건**: 빌드 오류 0, `LEA-` 0, F1~F6.

### 5-B. 주문서 강화 칸별 전환

**바꿀 파일**: `Character/PlayerGear.mlua`(`RequestScroll`), `Tower/TowerRunManager.mlua`(`DropScrolls`·Flush 두 곳만), `Data/RunConfig.csv`(3행), `Data/EquipSlotTable.csv`, `Slot/SlotUILogic.mlua`, `ui/GrowthUI`(5-2), `Lobby/GrowthUILogic.mlua`(탭 이름이 코드에 있으면).

- 이 단위 뒤 판 능력치는 아직 옛 `SlotRunData`를 읽는다 — 5-C를 바로 이어서 한다.

**완료 조건**: 빌드 오류 0, F7~F11, 스크린샷으로 칸 버튼·신발 숨김·모드 버튼 정렬 확인.

### 5-C. 판 반영 1 (주문서·신발)

**바꿀 파일**: `Character/PlayerGear.mlua`(`GearRunData`), `Tower/TowerRunManager.mlua`(4-6 5-C 1~8), `Tower/MainPanelLogic.mlua`, `ui/MainPanel`, `Data/EquipTable.csv`(Stat2), `Data/SlotMilestoneTable.csv`(Cape 30).

**완료 조건**: 빌드 오류 0, F12~F15.

### 5-D. 큐브 서버

**바꿀 파일**: `Character/PlayerGear.mlua`(`RequestCube`, `RequestCubeChoose`, `RequestBuyCube`, `CheckCubeRewards`, `DebugCubeSim`), `Tower/TowerRunManager.mlua`(`DropCubes`, `RunCubes`, 스테이지 클리어 무료 큐브).

- `DebugCubeSim(n, kind, grade)`: 저장하지 않는 가상 칸으로 n회 → `[VRF] cube sim kind=R grade=1 n=10000 up=..% ceilHit=.. line2Prime=..% line1Kept=..%`(플래티넘은 line1Kept 100%).

**완료 조건**: 빌드 오류 0, F16~F22.

### 5-E. 잠재능력 UI

**바꿀 파일**: `ui/GrowthUI`(5-3), `Slot/SlotUILogic.mlua`.

**완료 조건**: 빌드 오류 0, F23~F27, 스크린샷 2장(즉시 결과 / 대기 결과).

### 5-F. 판 반영 2 (잠재능력)

**바꿀 파일**: `Tower/TowerRunManager.mlua`(`PotBonus`, `AccessoryBonus`), `Tower/HeroCombat.mlua`, `Tower/BishopCombat.mlua`, `Tower/JobCombat.mlua`(칸 피해·`AddUlt`), `Tower/ResultPopupLogic.mlua`(큐브 표시).

**완료 조건**: 빌드 오류 0, F28~F32.

### 5-G. 회귀

F33~F36 + 4단계 기본 흐름(1층 판 시작 → 보스 → 결과) 1회.

---

## 7. 기능 확인 항목 (짧게, 밸런스 실측 아님)

준비(server_main):
```lua
local p = nil
for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
local g = p:GetComponent("script.PlayerGear")
local r = p:GetComponent("script.PlayerRoster")
local eng = p:GetComponent("script.PlayerEngrave")
```
- 테스트로 바꾼 값은 **테스트 전 `DebugGearDump()` 값을 기록해 두고** `DebugGearSet`으로 되돌린다(`DebugGearReset`은 큐브를 또 지급하므로 F2·F33에서만).

| 번호 | 단위 | 절차 | 기대 |
|---|---|---|---|
| F1 | 5-A | 새로고침 후 플레이, 로그 | `[VRF] gear loaded` 또는 `gear migrated`, `GearLoaded = true`, 표 행 수 `grade=4 cube=4 pot=22 cfg=19` |
| F2 | 5-A | `eng.SlotLv` 기록 → `g:DebugGearReset()` | `gear migrated`, 칸 1~3 모두 옛 단계와 같음, 무료 30·플래티넘 1, 신발 단계 L이면 레드 `20 + 2L`·블랙 `3 + floor(L/5)` |
| F3 | 5-A | 재접속(stop → play) | 이전 로그 없음, 값 그대로(큐브 중복 지급 없음) |
| F4 | 5-A | `g:DebugGearDump()` | 15칸 모두 등급 1, 줄 3개, 망토 줄은 칸 피해·궁극기 게이지 옵션만 |
| F5 | 5-A | `g:DebugRollSim(2000, "Cape")`, `g:DebugRollOnce(1, "Weapon", "R")`·`"P"` | `MainDmg` ≈ 10/44, 직업 고유 옵션 없음 / 굴림 결과 로그만, 저장 값 그대로, 플래티넘은 첫 줄이 현재 줄과 같음 |
| F6 | 5-A | `DebugGiveCubes("R", 1)` 후 재접속 | 레드 +1 유지. 모르는 키 보존은 `Decode/Encode` 정적 검사 |
| F7 | 5-B | 2칸 무기 단계 기록 → `g:DebugGiveScrolls("Weapon",3,3)` → 클라이언트 2칸·무기·3단계·3장 강화 | `gear scroll s2 Weapon tier=3 x3`, 2칸만 변화, 1·3칸 그대로 |
| F8 | 5-B | 편성을 바꾼 뒤 장비 탭 | 칸 버튼 이름만 바뀌고 칸 단계는 그대로 |
| F9 | 5-B | 탑에서 `RequestScroll` 호출 | `err_lobby` |
| F10 | 5-B | 탑 1판 | `scroll drop` 부위에 Shoes 없음, 판 끝 `ScrollData` 증가·저장. PlayerEngrave `slot` 저장 로그 없음 |
| F11 | 5-B | 화면 | 탭 이름 `장비`, 신발 칸 없음, 칸 버튼 3개·모드 버튼 정렬(스크린샷) |
| F12 | 5-C | 1칸 무기 10단계·2칸 0단계(`DebugGearSet`) → 판 시작 | `[VRF] gear run s1 lv=W10...`, `DebugDumpParty`에서 메인 공격력의 무기 몫이 동료 1의 2배 |
| F13 | 5-C | 판 장비 | `EquipTier`에 Shoes 없음, 메인 패널 장비 줄 5개, 장갑 줄에 치명타 데미지, `CritDmg` = 150 + 장갑 값 |
| F14 | 5-C | 무기 20단계를 2칸에만 → 판 시작 | 판 시작 무기 `EquipEnh = 0`(파티 공용 효과는 1칸 기준). 1칸에 주면 2 |
| F15 | 5-C | 1칸 망토 30단계 | `Bonus("Evade") >= 3`, `ExpPct` 없음 |
| F16 | 5-D | `DebugCubeSim(10000,"F",1)`, `("R",1)`, `("B",1)`, `("P",2)` | 상승률 ≈ 표 확률(천장 포함, 무료 ≈ 레드의 절반), 블랙 2번째 줄 prime ≈ 30%, 플래티넘 상승 0·첫 줄 유지 100% |
| F17 | 5-D | 레드 1개 사용 | 큐브 −1, 줄 바뀜, 저장 |
| F18 | 5-D | 블랙 사용 → 같은 칸 다시 큐브 | 대기 결과, 두 번째는 `err_pending`, 다른 칸은 가능 |
| F19 | 5-D | 대기 상태로 재접속 | `PendData` 그대로, 적용 옵션은 이전 값 |
| F20 | 5-D | `Choose(0)` / 다시 블랙 → `Choose(1)` | 유지: 등급·줄 그대로, 횟수 누적 / 적용: 새 값 |
| F21 | 5-D | 다이아 100 → 블랙 구매 1 → 다시 | 다이아 0·블랙 +1 → `err_dia` |
| F22 | 5-D | 첫 클리어·별 소급 | 블랙 = 3 + `floor(ClearedFloor/10)` + 3×(구간 수) (+신발 환산) |
| F23 | 5-E | 잠재능력 모드 | 등급·3줄·큐브 4종·확률/천장 표시, 주문서 위젯 숨김 |
| F24 | 5-E | 레드 사용 | 줄 갱신, 메시지, 칸 버튼 `+n · 등급` |
| F25 | 5-E | 블랙 → `Take` / `Keep` | `After` 표시·숨김 |
| F26 | 5-E | 큐브 0개 / 다이아 부족 | `Upgrade`·`Buy` Disabled |
| F27 | 5-E | 칸·부위·모드 바꾸며 탭 닫고 열기 | 선택·표시 유지, 오류 0 |
| F28 | 5-F | `g:DebugGearLines(1,"Glove","Crit@4,Crit@4,Crit@4")` → 판 시작 | 메인 `ActorCrit` +24, 동료 변화 없음 |
| F29 | 5-F | 2칸 `HpPct` / `DmgReduce` 줄 | 동료 1만 변화 |
| F30 | 5-F | 1칸 망토 `MainDmg@4` ×3 | 메인 주력기 피해 1.6배(기본 공격 그대로) |
| F31 | 5-F | 망토 `UltGainPct` | `[VRF] pot hook UltGainPct` |
| F32 | 5-F | 보스 처치 판 결과 | 결과 팝업 `큐브 2` 이상, 판 끝 저장 |
| F33 | 5-G | `DebugGearReset` → 재접속 2번 | 이전 1회만 |
| F34 | 5-G | 옛 진입점 검색 | `RequestSlotUpgrade`·`AddScrolls`·`FlushSlots`·`SlotRunData` 호출이 PlayerEngrave 밖에 0개 |
| F35 | 5-G | 1층 판 1회 | 정상, `LEA-` 0 |
| F36 | 5-G | 4단계 HUD | 정상 |

---

## 8. 구현 주의

- 메서드 설명 주석은 본문 첫 줄. `member`는 예약어. `string.gsub` 금지 → `find`+`sub`. 빈 표 `JSONEncode` 금지. `JSONDecode`는 옛 `OrbisSlot` 읽기에만.
- `Request*`는 `@ExecSpace("Server")` + `senderUserId` 검사. 표 읽기·파싱 함수는 ExecSpace 없음(양쪽 공용).
- 새 `.mlua`·속성 추가 뒤 반드시 메이커 새로고침.
- `PlayerGear`는 탑에서 **로비 요청을 거부**하지만 `AddScrolls/AddCubes/Flush`와 보상 확인은 탑에서도 동작해야 한다.
- 밸런스 값은 표로만 바꾼다(코드 상수 금지).
- `ui/GrowthUI`는 다른 탭과 같은 파일이다. 1번 탭 엔티티만 고치고 빌더 저장 전후 엔티티 수·다른 탭 경로를 비교한다.
- `Docs/UI-Design-System.md` 7절: 장비 칸에 컬러 장비 썸네일을 넣지 않는다.

---

## 9. 메인 세션 결정

1. 다이아로 큐브 구매: **유지**(레드 30, 블랙 100, 플래티넘 200). 무료는 구매 없음.
2. 기존 공용 슬롯 단계: **편성 칸 3개 모두에 복사**(2026-11-22 개정. 직업 5벌 복사 안 폐기).
3. 판 안 장비(정비소 강화): **파티 공용 유지**. 단계 보너스·명중 등 파티 공용 효과는 **1칸 기준**.
4. 캐릭터 화면 → 장비 탭 바로가기: 이번엔 넣지 않음.
5. `PlayerEngrave.mlua`는 **수정하지 않는다**. 옛 경로는 `TowerRunManager`에서 호출만 끊는다.
6. 테스트는 기능만 짧게. 확률 모의는 서버 함수 1회 호출로 분포만 확인.
7. 큐브 4종(사용자 확정 2026-11-22): 무료 `358fb909…` / 레드 `009913f2…` / 블랙 `1edec01f…` / 플래티넘 `98ad6a59…`. 
   - 무료: 수량 소모·하루 제한 없음·상승 확률 레드의 절반(5/1.5/0.5%)·즉시 적용·천장 2배(점수제)·옵션 확률 레드와 같음. 획득: 첫 지급 30, 스테이지 클리어 50% 1개, 층 보스 3개.
   - 플래티넘: 첫 줄 잠금, 2·3번째 줄만 새로, 결과 1개를 대기 저장 후 비포/애프터 선택, 등급 상승 없음. 획득: 별 구간 마지막 상자 1개(소급), 다이아 200, 첫 지급 1.
   - 화면: 블랙·플래티넘은 같은 비포/애프터 묶음, 플래티넘은 첫 줄 잠금 표시.

## 10. 빠진 리소스

- 등급 표시용 작은 테두리(지금은 글자 색으로만).

## 5-A 후 메인 세션 결정
1. 단계 보너스 중 **그 구성원 능력치에 붙는 것(배율·고유 능력치 보너스)은 자기 칸 값**으로 각 구성원에게 적용한다. **진짜 파티 전체 효과(정비소 강화 확률, 명중 등)만 1칸 기준**으로 둔다. 2·3칸 투자도 단계 보너스를 받게 한다.
2. 플래티넘 2·3줄 등급 확률 30%, 첫 지급 1개: 승인.
3. 다음 단위 시작 때 F3(재접속 시 이전 1회만)·F5·F6을 먼저 짧게 확인한다.

## 5-C 후 메인 세션 결정
1. 회피(Evade)는 공격받은 구성원의 자기 칸 망토 단계를 따른다(구성원 능력치는 자기 칸 원칙과 일치).
2. 메인 패널 망토 줄을 실행 중에 신발 자리로 올리는 처리는 그대로 둔다(화면 파일 수정 불필요).
3. 직업별 헤어·얼굴(CharJobTable HairRUID/FaceRUID) 로비·탑 동료 적용 확인됨.

## 5-D 후 메인 세션 결정
1. 보스 스테이지는 보스 보상(레드 2 + 무료 3)만 주고, 일반 스테이지 클리어 무료 50%는 더하지 않는다.
2. 단일 몬스터 경로의 정예(보스 아님)도 레드 25% 대상에 포함한다.
3. 첫 클리어 보상 재지급 버그(Decode 중 보상 확인 끼어듦)는 수정됨.

## 5-E 후 메인 세션 결정
1. 잠재능력 합계는 그 칸 5부위 전체 합으로 둔다(판에 실제로 들어가는 값).
2. 블랙·플래티넘 새 옵션 카드는 파랑 선택 테두리 유지.
3. 큐브 구매는 1개 버튼만 둔다(10개 묶음은 필요해지면 추가).

## 5-F 후 메인 세션 결정
1. 결과 팝업 보상 칸이 5개 이상일 때 겹치는 문제는 최대 7종이 들어가도록 배치를 조정한다(메소 개편 작업에서 함께).
2. 공용 전투 직업(JobCombat)도 장신구 스킬 피해를 스킬 타격에 반영한다(중복 곱셈 금지).
3. 히어로 스킬 피해가 기본 공격·지속 피해에도 붙는 기존 동작은 유지.

## 메소 개편 후 메인 세션 결정
1. 지역 선택 보상 아이콘의 스킬북 자리는 금색 메소 아이콘으로 유지(새 클리어 보상).
2. 공용 주문서 하급·중급·상급 아이콘은 일반 주문서 아이콘을 임시 사용(등급별 리소스는 사용자에게 확인).
3. 스킬북을 아직 쓰는 곳: PlayerEngrave 별 구간 상자 지급(수정 금지 파일 — 사용자 확인 필요), 옛 스킬 화면 HeroSkillUILogic·GachaUILogic(6단계 옛 스킬 제거 때 정리).
4. 주문서 드랍 등급 비율은 기존 ScrollDropTable 유지.
5. 사용자 승인으로 PlayerEngrave 별 구간 상자의 스킬북 지급을 제거(깃털만 지급, 문구 수정). 남은 스킬북 사용처는 옛 스킬 화면뿐(6단계 정리).
