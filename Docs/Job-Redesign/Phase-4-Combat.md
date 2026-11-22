# 4단계 요구사항: 전투 개편 (판 안 레벨 삭제 + 캐릭터 스탯 + 칸 진화 스킬)

> 작성: 자료 조사 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(최우선, 3절 스킬 진화표) → `01-Agent-Rules.md` → `02-Roadmap.md` 4단계 → 이 문서
> 선행: 1~3단계 완료 + `Phase-3-Gacha.md` 끝 "3단계 보완"(4-A 구현 에이전트가 먼저 처리)
> 이 문서만 보고 구현·테스트할 수 있도록 바꿀 파일·함수·값·완료 조건·테스트를 적는다. 문서에 없는 판단이 필요하면 마스터 스펙 의도대로 정하고 보고에 적는다.
> **건드리지 않는 파일**: `Engrave/PlayerEngrave.mlua`, `Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md`(다른 세션 작업 중, 읽기만). 로비 스킬 화면(`Skill/HeroSkillUILogic.mlua`, `ui/GrowthUI` 스킬 탭)·스킬 뽑기는 **수정하지 않는다**(6단계 제거). 4단계는 "판에서 읽지 않게"만 한다.

---

## 0. 결정 사항 요약

| 번호 | 결정 | 근거 |
|---|---|---|
| C1 | 4단계를 **작업 단위 7개(4-A~4-G)** 로 나눈다. 앞 단위만 끝나도 게임이 돌아간다 | 1절 |
| C2 | 판 시작 편성은 `PlayerRoster`의 새 함수 `RunPartyData()`(잠김·중복·없는 직업을 `FixParty`와 같은 규칙으로 보정, **저장하지 않음**)로 정한다. 보정되면 `[VRF] run party fixed a,b,c` | 1단계 보완 Q8 |
| C3 | "메인 바이퍼인데 HUD가 나이트로드" 원인 = **`JobCombat.JobId`가 `@Sync`가 아니어서 클라이언트는 기본값 `NightLord`를 본다** + **`V01`(서머솔트 킥) 아이콘 RUID가 표에 비어 있음**. 서버 전투는 `HeroOf`가 `JobId`를 바꾼 뒤 `BeginRun`하므로 바이퍼로 돈다(`[VRF] JobCombat begin job=NightLord`는 생성 시 기본값 로그일 뿐) | 2절 2-2 |
| C4 | 판 안 레벨·레벨당 스탯·레벨업 회복·레벨업 카드 삭제. 능력치는 **구성원마다 자기 캐릭터 스탯**(`CharStat`: 레벨·차수)으로. 동료도 자기 직업 체력·공격·방어 | 마스터 스펙 3절 '삭제' |
| C5 | `Level` Sync 속성은 지우지 않고 **"메인 캐릭터 레벨(표시용, 판 중 고정)"** 으로 뜻을 바꾼다. 판 UI(`MainPanelLogic`, `TarotLogic`)가 그대로 읽어도 맞는 값이 나온다 | 수정 범위 최소화 |
| C6 | **캐릭터 공격력 표(`CharJobTable`의 `BaseAtk`·`AtkPerLv`)를 바꾼다**(체력·방어는 그대로). 1차 스킬이 1개뿐이라 지금 값(Lv1 공격 10)으로는 새 계정이 1~2층에서 막힌다 | 7절 계산 |
| C7 | 레벨업 전체 회복 대신 **전투 스테이지 클리어 때 살아 있는 구성원 최대 체력 30% 회복**(`RunConfig StageClearHealPct=30`, 0이면 끔) | 지금은 전투 스테이지마다 레벨업 = 전체 회복이라, 그냥 지우면 난이도가 급변한다. 7절 |
| C8 | 각인·은총·계정의 판 안 레벨 효과는 **값 데이터를 바꾸지 않고 뜻만 바꾼다**: `StartLevel`(선행 학습) = 판 시작 돌파 선택 +v회, `ExtraPick` = 전투 스테이지 클리어 때 v% 확률로 돌파 선택 +1. 설명 문구 교체는 6-C | 데이터·PlayerEngrave 무변경 |
| C9 | 칸 스킬 = `CharSlotTable`(1단계, 진화표)을 그대로 쓴다. 수치 조정은 새 표 **`CharSkillSlotTable`**(직업·칸·차수별 계수 배율·쿨타임·마나·발동 방식) | 3·4절 |
| C10 | 스킬 위력의 아웃게임 레벨 배율(`LvMul`, Lv.11 = 3배)은 **칸 표의 `PctMul`** 로 바꾼다. 진화했는데 약해지는 스킬(레이징 블로우 등)은 쿨타임·배율로 보정 | 5절 |
| C11 | 하이퍼 = 4차 해금, **쿨타임 40초 자동 시전, 마나 0**. 궁극기 = 5차 해금, **게이지 100: 기본 공격 적중 +4, 스킬 시전 +6**(약 30초에 1번), 가득 차면 다음 시전 차례에 자동, 마나 0. HUD는 5번째 스킬 칸의 쿨타임 덮개를 게이지로 쓴다(`.ui` 변경 없음) | 마스터 스펙 3절 |
| C12 | 고유 패시브 = 차수마다 패시브 레벨 1·3·5·7·9 + 3차에 **확률 발동 스킬**, 4차에 **위기 자동 발동 스킬**을 숨은 장착으로 흡수, 5차에 고유 강화 1개 | 6절 표 |
| C13 | 돌파는 **"직업.칸" 키 + 돌파 번호(B1~B6)** 로 저장한다. 칸의 스킬이 바뀌면 같은 번호 돌파가 새 스킬에 켜진다. 판 중 진화는 없지만 `DebugSwapSlot`으로 확인 | 마스터 스펙 3절 "돌파는 칸에 붙는다" |
| C14 | 마법의 서·돌파 카드·스킬석은 그대로. 판 안 스킬 카드(CHOICE 상태)는 파티 판에서 이미 쓰이지 않으며 코드만 남긴다(6단계 정리) | 마스터 스펙 3절 |
| C15 | 테스트용 디버그 함수(판 바로 시작·층 바로 클리어·무적·게이지 채우기·피해량 측정·파티 덤프)를 4-A에서 먼저 만든다 | 8절 |
| C16 | 캐릭터 경험치는 판에서 주지 않는다(경험의 서만). 계정 경험치(`SettleAccount`)는 그대로 | 마스터 스펙 2절 |

---

## 1. 작업 단위와 순서

| 단위 | 내용 | 끝나면 게임 상태 | 테스트 |
|---|---|---|---|
| (선행) | `Phase-3-Gacha.md` 끝 "3단계 보완" (B1·Q2~Q5) | 3단계 화면 정리 | 그 절의 확인 항목 |
| **4-A** | 판 시작 정리: `RunPartyData` 보정(Q8), 직업 표시 문제(Q7), 판 시작 순서, 아이콘 임시값, 디버그 함수, 기준 피해량 측정 | 지금 전투 그대로 + 잠긴 직업 방어 + HUD 직업 정상 | T1~T7 |
| **4-B** | 판 안 레벨 삭제 + 구성원별 캐릭터 스탯 + 공격력 표 조정 + 스테이지 클리어 회복 + 각인 뜻 변경 | 레벨업 없음. 스킬은 아직 로비 장착 5칸 | T8~T14 |
| **4-C** | 칸 진화 스킬: `CharSkillSlotTable`, 전투 3종 `BeginRun` 교체(로비 장착·레벨 안 읽음), `PctMul`·쿨·마나 덮어쓰기, HUD 칸 아이콘·이름·레벨 | 차수 스킬로 전투. 하이퍼는 긴 쿨 스킬, 궁극기는 일반 스킬처럼 동작 | T15~T20 |
| **4-D** | 하이퍼 자동 시전 방식 + 궁극기 게이지 + HUD 게이지 | 마스터 스펙 하이퍼·궁극기 완성 | T21~T25 |
| **4-E** | 고유 패시브 차수별 효과(`CharPassiveTable`), 확률·위기 스킬 흡수, 5차 고유 강화 | 패시브 완성 | T26~T30 |
| **4-F** | 돌파를 칸 키로 저장, 마법의 서 정보 칸 기준, `DebugSwapSlot` | 돌파 칸 연결 완성 | T31~T33 |
| **4-G** | 밸런스 측정(레벨 1·10·20·30 × 층), 표 값 미세 조정, 1층 통짜 회귀 | 4단계 완료 | T34~T38 |

- 순서는 위 표 그대로. 단위마다 `maker_run_command stop` → `maker_refresh_workspace` → `maker_build_logs` 오류 0 → 플레이 → 그 단위 테스트. 마지막(4-G)에 T1~T38을 한 번 더 훑는다(전부가 어려우면 4-G 표의 필수 항목만).
- 각 단위의 구현 에이전트는 시작 전에 이 문서 해당 절 + 2절(현재 구조)을 읽는다.

---

## 2. 현재 구조 조사 결과

### 2-1. 판 레벨에 걸려 있는 것 전부와 대체 규칙

| # | 위치 (`Tower/TowerRunManager.mlua` 기준, 행은 대략) | 지금 | 4단계 대체 | 단위 |
|---|---|---|---|---|
| 1 | `RecalcStats`(2753) `BasePlayerAtk + LevelAtkBonus × (Level-1)`, 체력·방어 동일. 동료는 메인과 같은 체력, 공격·방어는 메인 값 공유(`_T.goddessAtkBase`) | 판 레벨 스탯 | 구성원별 `CharRunData.atk/hp/def` → `_T.actorBase[actor]` | 4-B |
| 2 | 전투 스테이지 클리어(5370) `if self.StageLevelUp and not isBoss then self:DoLevelUp(1, 0) end` | 레벨 +1·카드·**체력 전체 회복**·레벨업 이펙트 | 호출 삭제. 회복은 `StageClearHealPct`(30%) | 4-B |
| 3 | `StartRun`(2847~2926) `startLevel = 1 + Bonus("StartLevel")`, `Level/Exp/ExpToNext`, `PendingPicks = startLevel` | 시작 레벨·시작 카드 | `Level = 메인 캐릭터 레벨`(표시용), `Exp = 0`, `ExpToNext = 1`, `PendingPicks = 0` | 4-B |
| 4 | `PickJob`(1958) 파티 판 `PendingPicks = 0` | 시작 카드 없음 | `PendingPicks = floor(Bonus("StartLevel"))` → 0보다 크면 첫 전투 전에 돌파 카드 | 4-B |
| 5 | `BeginWave`(3257) `Exp = 0, ExpToNext = size` | 하단 바 = 웨이브 진행 | 그대로 둔다(웨이브 표시용). 처치 수는 지금도 올라가지 않으므로 4-B에서 처치마다 `Exp + 1`(선택, 2줄) | 4-B |
| 6 | `DoLevelUp`(3614) | 레벨업 본체 | 호출 0. 함수는 남기고 첫 줄 주석 `-- 4단계부터 호출 없음 (6단계 삭제)` | 4-B |
| 7 | `Bonus("ExtraPick")` (각인 `Collector`, 은총 `Stone`, 계정 규칙 75레벨) | 레벨업 때 v% 확률 카드 +1 | 전투 스테이지 클리어 때 v% 확률로 `PendingPicks` +1 (돌파 1장 더) | 4-B |
| 8 | `Bonus("StartLevel")` (각인 `HeadStart` 선행 학습) | 시작 레벨 +v | 판 시작 돌파 선택 +v회 (#4) | 4-B |
| 9 | `HasSkillToLearn()`(1921) — `DoLevelUp`·거래 `SkillPick`에서 사용 | `SkillLv < max` | 새 `HasBreakLeft()`(구성원 장착 스킬 중 `RemainingBreaks`가 하나라도 있으면 true)로 교체 | 4-B |
| 10 | 거래 `DealTable` `A_Stone/D_Stone/M_Stone`(효과 `SkillPick`) | "스킬 카드 선택 n회" | 동작은 그대로(파티 판에서는 이미 돌파 선택). **설명만** `돌파 선택 n회를 얻습니다`로 (CSV 문구 3줄) | 4-B |
| 11 | `BattleHUDLogic.UpdateMember`(636) `"Lv." .. mgr.Level` | 구성원 셋 다 판 레벨 | 구성원 자기 `RunLabel`(차수 이름)·`RunLevel`(캐릭터 레벨) | 4-C |
| 12 | `MainPanelLogic` 하단 `Lv.`(668)·정보 탭 `레벨/경험치`(766~767)·경험치 바(713) | 판 레벨·경험치 | `Level`이 메인 캐릭터 레벨이므로 `Lv.`는 그대로. 정보 탭 `경험치` 줄 삭제, `레벨` 줄은 `레벨        Lv.x (메인)` | 4-B |
| 13 | `TarotLogic`(671) `"Lv." .. mgr.Level .. " — 스킬 강화"` | CHOICE 상태 부제 | 파티 판은 CHOICE가 오지 않음. 수정 없음 | — |
| 14 | 처치 경험치 | 이미 없음(`GiveKillReward` 주석) | 그대로. 캐릭터 경험치도 주지 않음 | — |
| 15 | 돌파 해금 조건 | 파티 판은 레벨과 무관(전투 스테이지 1~9 클리어마다 1번, 히어로식 스킬당 최대 3개) | 그대로 | — |
| 16 | `SkillLv[id]` | 장착 스킬은 판 시작부터 1(배움 표시) | 그대로(1). 레벨로 쓰지 않음 | — |
| 17 | 전투 3종 `LvMul(id)` = `1 + (아웃게임 레벨-1) × 0.2` | 스킬 레벨 위력 | `CharSkillSlotTable.PctMul` | 4-C |
| 18 | `ComboLv`(히어로)·`HolyLv`(비숍)·`PassiveLv`(직업 공용) ← PlayerEngrave `ComboAttackLv / HolyWaterLv / JobPassiveLv` | 아웃게임 패시브 레벨 | 차수 패시브 레벨 1·3·5·7·9 | 4-C(값), 4-E(표) |
| 19 | 전투 3종 `BeginRun` ← `eng:SkillRunData()` / `SkillRunDataFor(job)` (로비 장착 5칸·레벨) | 장착 5칸 | `mgr:RunDataFor(job).slots` (칸 6개 현재 차수) | 4-C |
| 20 | `RunConfig` `LevelAtkBonus / LevelDefBonus / LevelHpBonus / BaseExpToNext / ExpToNextGrowth / BaseExpReward / ExpGrowth / EliteExtraPicks` | 판 레벨용 | 읽기는 남기되 사용처 없음. 키 삭제는 6단계 | — |
| 21 | `StageLevelUp`(Sync, RunConfig 1) | 스테이지 레벨업 켜기 | 이름은 그대로 두고 **"하단 바를 웨이브 진행으로 표시"** 뜻으로만(`MainPanelLogic` 729행이 읽음). 레벨업은 이 값과 무관하게 없음 | 4-B |

### 2-2. 메인 바이퍼 → HUD 나이트로드 문제 (2단계 Q7) 원인

1. `JobCombat`의 `property string JobId = "NightLord"`에 `@Sync`가 없다. 서버 `HeroOf`(531행)가 메인 직업으로 `j.JobId = job`을 바꾸지만 클라이언트로 가지 않는다. `BattleHUDLogic.UpdateMember`(636~651행)는 클라이언트에서 `cc.JobId`로 이름·자원(`표식 0/100`)을 고르므로 나이트로드로 보인다. (`TowerPlayer.ProjectileLook`은 서버라 영향 없음.)
2. 첫 스킬 칸이 빈 이유: 메인 바이퍼의 첫 장착 `V01`(서머솔트 킥)의 `IconRUID`가 `HeroSkillTable`·`JobSkillTable` 둘 다 **빈 값**이다(데이터 확인). HUD는 아이콘이 빈 문자열이면 그림을 지운다.
3. `[VRF] JobCombat begin job=NightLord`는 `JobCombat.OnBeginPlay`(52행)가 엔티티 생성 때 기본값을 찍는 로그다. 실제 판 로그는 `[VRF] jobcombat run job=Viper equip=...`(266행)이다.
4. 부수 문제: `StartRun`은 `for _, h in ipairs(self:Heroes()) do h:ClearState() end`(2911행)를 `SpawnCompanion → LoadFormation`(2934행)보다 **먼저** 한다. 첫 판에는 `_T.mainJob`이 nil이라 `HeroOf`가 `HeroCombat`만 돌려주고, 판 사이에 메인 직업을 바꾸면 이전 직업 컴포넌트를 기준으로 정리한다. 지금은 우연히 문제가 안 되지만 4-A에서 순서를 바로잡는다.

### 2-3. 잠긴 직업 판 전달 (1단계 Q8)

- `TowerRunManager.LoadFormation`(584행)은 `PlayerEngrave.MainJob/CompJob/Comp2Job`을 읽고 **5직업이면 잠김과 무관하게** 쓴다. 로비 `PlayerRoster.FixParty`는 로비에서만 1초마다 돌기 때문에, 편성 원본이 탑에 있는 동안 바뀌거나 보정 전에 탑으로 가면 잠긴 직업으로 판이 시작될 수 있다.
- `PlayerRoster.PartyRunData()`(1115행)는 보정 없이 편성 순서대로 돌려준다.

### 2-4. 돌파 저장 구조

- 정의: `SkillBreakTable.csv` (`SkillId, OptionId(예 H01_B1), Name, Effect, Value`). 5직업 × 15스킬 × 5개(+ `H02_B6`).
- 판 안 저장: `TowerRunManager._T.breakTaken[스킬Id][돌파Id] = true`, 표시용 Sync `SkillBreaks[스킬Id]`(쉼표 목록), `SkillInfo[스킬Id]`.
- 판정: 전투 3종 `Has(id, n)` → `mgr:HeroBreak(id, id.."_B"..n)` (1197행). 남은 돌파 `RemainingBreaks(sk)`(1753행)는 `_T.breakTaken[sk.id]`를 본다(히어로식 최대 3개).
- 획득 경로: 파티 돌파 카드 `RollPartyBreaks`(1813) → `RequestPick`(BREAK), 마법의 서 `RequestBookUse(skillId)` → `PickBookBreak`, 디버그 `DebugGiveBreaks(ids)`.
- **스킬 Id에 붙어 있다** → 4-F에서 칸 키로 바꾼다.

### 2-5. 스킬 레벨·장착을 판이 읽는 곳 (4-C에서 끊을 곳)

| 파일 | 행 | 읽는 것 |
|---|---|---|
| `Tower/HeroCombat.mlua` | 542 | `eng:SkillRunData()` (장착·레벨·`comboLv`) |
| `Tower/BishopCombat.mlua` | 156 | `eng:SkillRunDataFor("Bishop")` (장착·레벨·`holyLv`) |
| `Tower/JobCombat.mlua` | 233 | `eng:SkillRunDataFor(self.JobId)` (장착·레벨·`passiveLv`) |
| 위 3개 `LvMul` | 각 1곳 | `_T.ownLv[id]` |

그 밖에 `Skill/HeroSkillUILogic.mlua`가 `SkillOwnLv/SkillEquip/ComboAttackLv…`을 읽지만 **로비 화면이라 그대로 둔다.**

---

## 3. 데이터

모두 `RootDesk/MyDesk/Data/`, **UTF-8 BOM** CSV. 새 표는 같은 이름 `.userdataset`(`RelicTable.userdataset` 복사, 새 uuid, `name`·`id` = 파일 이름).

### 3-1. `CharJobTable.csv` — 공격력 두 열만 바꾼다 (4-B)

| JobId | BaseAtk 전 → 후 | AtkPerLv 전 → 후 |
|---|---|---|
| Hero | 10 → **28** | 2 → **0.8** |
| Bishop | 9 → **25** | 1.8 → **0.72** |
| NightLord | 11 → **31** | 2.2 → **0.88** |
| Bowmaster | 11 → **31** | 2.1 → **0.84** |
| Viper | 10 → **28** | 2 → **0.8** |

- 체력·방어(`BaseHp/BaseDef/HpPerLv/DefPerLv`)는 그대로. 직업 차이(나이트로드·보우마스터 공격 높음, 비숍 낮음)는 비율 그대로 유지.
- 결과 공격력(`CharStat`, 차수 보너스 포함): 히어로 Lv1 1차 **28**, Lv7 2차 **36**, Lv13 3차 **45**, Lv19 4차 **57**, Lv25 5차 **71**, Lv30 5차 **77**. 캐릭터 화면(`CharacterUI`)도 같은 값을 보여 준다.
- **회귀 기대값이 바뀐다**: 1단계 T15 `CharRunData("Hero")` 2차 Lv7 `atk=24` → **`atk=36`**(hp 211·def 9 그대로). 2단계 캐릭터 화면 능력치 테스트도 이 표 기준으로 읽는다.

### 3-2. `CharSkillSlotTable.csv` — 칸·차수별 전투 수치 (신규, 4-C)

열: `JobId,SlotKey,Tier,SkillId,PctMul,CdSec,Mana,TriggerAs,#Desc`

- **칸에 어떤 스킬이 들어가는지의 원본은 `CharSlotTable`(1단계)** 이다. 이 표의 `SkillId`는 확인용으로, 로드할 때 `CharSlotTable`의 같은 (직업·칸·차수) 값과 다르면 `log_warning("[VRF] slot tune mismatch ...")`를 남기고 **`CharSlotTable` 값을 쓴다.**
- `PctMul`: 그 칸 스킬의 피해 계수 배율(아웃게임 레벨 배율 `LvMul` 대체). 빈 값 = 1.
- `CdSec`: 0 또는 빈 값 = `JobSkillTable` 쿨타임 그대로, 그 외 = 이 값으로 덮어씀(돌파의 쿨타임 변경은 그 위에 그대로 적용).
- `Mana`: -1 또는 빈 값 = 표 그대로, 그 외 = 이 값.
- `TriggerAs`: 빈 값 = 표 `Trigger` 그대로 / `Cooldown` = 쿨타임마다 자동 시전(마나 0) / `Ult` = 궁극기 게이지 시전(4-D부터). **4-C에서는 `Ult`를 무시**(일반 스킬처럼 동작)하고 4-D에서 켠다.
- 패시브 칸은 이 표에 없다(6절 `CharPassiveTable`).

```csv
JobId,SlotKey,Tier,SkillId,PctMul,CdSec,Mana,TriggerAs,#Desc
Hero,Main,1,H01,1,0,-1,,슬래시 블러스트
Hero,Main,2,H02,1,0,-1,,브랜디쉬
Hero,Main,3,H02,1,0,-1,,브랜디쉬
Hero,Main,4,H08,1.15,5,-1,,레이징 블로우 (쿨 10→5: 진화 후 약해지지 않게)
Hero,Main,5,H08,1.15,5,-1,,레이징 블로우
Hero,Sub,2,H05,1,0,-1,,인레이지
Hero,Sub,3,H11,1,0,-1,,에픽 어드벤쳐
Hero,Sub,4,H11,1,0,-1,,에픽 어드벤쳐
Hero,Sub,5,H11,1,0,-1,,에픽 어드벤쳐
Hero,Tech,3,H03,1,0,-1,,패닉
Hero,Tech,4,H03,1,0,-1,,패닉
Hero,Tech,5,H13,1.6,0,-1,,콤보 데스폴트
Hero,Hyper,4,H10,1.5,40,0,Cooldown,발할라 (하이퍼)
Hero,Hyper,5,H10,1.5,40,0,Cooldown,발할라 (하이퍼)
Hero,Ult,5,H12,1.5,0,0,Ult,소드 오브 버닝 소울 (궁극기)
Bishop,Main,1,B01,1,0,-1,,에너지 볼트
Bishop,Main,2,B02,1,0,-1,,홀리 애로우
Bishop,Main,3,B02,1,0,-1,,홀리 애로우
Bishop,Main,4,B06,1.25,5,-1,,엔젤레이 (쿨 10→5)
Bishop,Main,5,B06,1.25,5,-1,,엔젤레이
Bishop,Sub,2,B11,1,0,-1,,어드밴스드 블레스
Bishop,Sub,3,B09,1,0,-1,,인피니티
Bishop,Sub,4,B09,1,0,-1,,인피니티
Bishop,Sub,5,B09,1,0,-1,,인피니티
Bishop,Tech,3,B03,1,0,-1,,샤이닝 레이
Bishop,Tech,4,B03,1,0,-1,,샤이닝 레이
Bishop,Tech,5,B15,1.3,0,-1,,디바인 퍼니시먼트
Bishop,Hyper,4,B10,2,40,0,Cooldown,헤븐즈 도어 (하이퍼)
Bishop,Hyper,5,B10,2,40,0,Cooldown,헤븐즈 도어 (하이퍼)
Bishop,Ult,5,B13,1.5,0,0,Ult,엔젤 오브 제미니 (궁극기)
NightLord,Main,1,N01,1,0,-1,,럭키 세븐
NightLord,Main,2,N03,1,0,-1,,트리플 스로우
NightLord,Main,3,N03,1,0,-1,,트리플 스로우
NightLord,Main,4,N06,1.2,5,-1,,쿼드러플 스로우 (쿨 10→5)
NightLord,Main,5,N06,1.2,5,-1,,쿼드러플 스로우
NightLord,Sub,2,N02,1,0,-1,,드레인
NightLord,Sub,3,N04,1,0,-1,,쉐도우 파트너
NightLord,Sub,4,N04,1,0,-1,,쉐도우 파트너
NightLord,Sub,5,N04,1,0,-1,,쉐도우 파트너
NightLord,Tech,3,N07,1,0,-1,,써든레이드
NightLord,Tech,4,N07,1,0,-1,,써든레이드
NightLord,Tech,5,N14,1.6,8,-1,,풍마수리검 (쿨 12→8)
NightLord,Hyper,4,N11,1,40,0,Cooldown,블리딩 톡신 (하이퍼)
NightLord,Hyper,5,N11,1,40,0,Cooldown,블리딩 톡신 (하이퍼)
NightLord,Ult,5,N15,1.5,0,0,Ult,다크로드의 비전서 (궁극기)
Bowmaster,Main,1,A01,1,0,-1,,애로우 블로우
Bowmaster,Main,2,A02,1.5,0,-1,,애로우 봄
Bowmaster,Main,3,A02,1.5,0,-1,,애로우 봄
Bowmaster,Main,4,A06,1.5,0,-1,,폭풍의 시
Bowmaster,Main,5,A06,1.5,0,-1,,폭풍의 시
Bowmaster,Sub,2,A05,1,0,-1,,애로우 플래터
Bowmaster,Sub,3,A04,1,0,-1,,피닉스
Bowmaster,Sub,4,A04,1,0,-1,,피닉스
Bowmaster,Sub,5,A04,1,0,-1,,피닉스
Bowmaster,Tech,3,A07,1,0,-1,,애로우 일루전
Bowmaster,Tech,4,A07,1,0,-1,,애로우 일루전
Bowmaster,Tech,5,A14,1.5,0,-1,,실루엣 미라주
Bowmaster,Hyper,4,A10,1.5,40,0,Cooldown,애로우 레인 (하이퍼)
Bowmaster,Hyper,5,A10,1.5,40,0,Cooldown,애로우 레인 (하이퍼)
Bowmaster,Ult,5,A15,1.5,0,0,Ult,잔영의 시 (궁극기)
Viper,Main,1,V01,1,0,-1,,서머솔트 킥
Viper,Main,2,V02,1.4,0,-1,,백 스핀 블로우
Viper,Main,3,V02,1.4,0,-1,,백 스핀 블로우
Viper,Main,4,V07,1.4,0,-1,,데몰리션
Viper,Main,5,V07,1.4,0,-1,,데몰리션
Viper,Sub,2,V05,1,0,-1,,트랜스폼
Viper,Sub,3,V14,1,0,-1,,퓨리어스 차지
Viper,Sub,4,V14,1,0,-1,,퓨리어스 차지
Viper,Sub,5,V14,1,0,-1,,퓨리어스 차지
Viper,Tech,3,V04,1,0,-1,,에너지 버스트
Viper,Tech,4,V04,1,0,-1,,에너지 버스트
Viper,Tech,5,V12,1.2,0,-1,,하울링 피스트
Viper,Hyper,4,V10,1,40,0,,스티뮬레이트 (하이퍼. Auto 그대로 + 쿨 40)
Viper,Hyper,5,V10,1,40,0,,스티뮬레이트 (하이퍼)
Viper,Ult,5,V15,1.5,0,0,Ult,리버레이트 넵투누스 (궁극기)
```

`PctMul`·`CdSec` 근거(초당 계수 = 계수 × 타수 × (1 + 재발동 확률) ÷ 쿨타임, 데이터 기준):

| 칸 | 진화 전 → 후 (원래 값) | 보정 후 |
|---|---|---|
| 히어로 주력 | H01 24%/s → H02 57%/s → **H08 31%/s(약해짐)** | H08 쿨 5·×1.15 → 71%/s |
| 비숍 주력 | B01 24 → B02 57 → **B06 26** | B06 쿨 5·×1.25 → 65 |
| 나이트로드 주력 | N01 24 → N03 70 → **N06 32** | N06 쿨 5·×1.2 → 77 |
| 보우마스터 주력 | A01 28 → **A02 30** → A06 40 | A02 ×1.5 → 45, A06 ×1.5 → 60 |
| 바이퍼 주력 | V01 32 → **V02 32** → V07 44 | V02 ×1.4 → 45, V07 ×1.4 → 62 |
| 기술 | H03 70 → H13 51 / B03 43 → B15 51 / N07 36 → N14 21 / A07 34 → A14 30 / V04 25 → V12 38 | H13 ×1.6, B15 ×1.3, N14 쿨 8·×1.6, A14 ×1.5, V12 ×1.2 → 진화 후 1.2~1.4배 |
| 하이퍼 | 공격형(H10·B10·A10)은 쿨 40으로 늘어난 만큼 ×1.5~2, 버프형(N11·V10)은 ×1 | |
| 궁극기 | 게이지 약 30초 주기, 전부 ×1.5 | |

### 3-3. `CharPassiveTable.csv` — 고유 패시브 차수별 효과 (신규, 4-E)

열: `JobId,Tier,PassiveLv,Grant,Extra,#Desc` — `Grant` = 숨은 장착으로 넣을 스킬 Id(`|` 구분), `Extra` = 코드 처리 강화(`|` 구분). 내용은 6절.

```csv
JobId,Tier,PassiveLv,Grant,Extra,#Desc
Hero,1,1,,,콤보 어택 Lv1
Hero,2,3,,,콤보 어택 Lv3
Hero,3,5,H15,,콤보 어택 Lv5 + 콤보 인스팅트(확률 발동)
Hero,4,7,H15,ComboCap10,콤보 어택 Lv7 + 콤보 최대 10
Hero,5,9,H15,ComboCap10|ComboGain50,콤보 어택 Lv9 + 콤보 획득 확률 +50%
Bishop,1,1,,,성수 Lv1
Bishop,2,3,,,성수 Lv3
Bishop,3,5,B12,,성수 Lv5 + 프레이(확률 발동)
Bishop,4,7,B12|B04,,성수 Lv7 + 홀리 매직쉘(위기 자동)
Bishop,5,9,B12|B04,HolyMax10,성수 Lv9 + 성수 최대 10
NightLord,1,1,,,표식 Lv1
NightLord,2,3,,,표식 Lv3
NightLord,3,5,N13,,표식 Lv5 + 쉐도우 스피어(확률 발동)
NightLord,4,7,N13|N09,,표식 Lv7 + 다크 사이트(위기 자동)
NightLord,5,9,N13|N09,ResMax8,표식 Lv9 + 표식 최대 8
Bowmaster,1,1,,,강화 화살 Lv1
Bowmaster,2,3,,,강화 화살 Lv3
Bowmaster,3,5,A13,,강화 화살 Lv5 + 가이디드 애로우(확률 발동)
Bowmaster,4,7,A13|A09,,강화 화살 Lv7 + 일루전 스텝(위기 자동)
Bowmaster,5,9,A13|A09,ResMax15,강화 화살 Lv9 + 강화 화살 최대 15
Viper,1,1,,,에너지 Lv1
Viper,2,3,,,에너지 Lv3
Viper,3,5,V13,,에너지 Lv5 + 서펀트 스크류(확률 발동)
Viper,4,7,V13|V09,,에너지 Lv7 + 언위어링 넥타(위기 자동)
Viper,5,9,V13|V09,EnergyPlus3,에너지 Lv9 + 타격당 에너지 +3
```

### 3-4. `RunConfig.csv`에 행 추가 (4-B·4-D)

```csv
StageClearHealPct,30,전투 스테이지 클리어 시 살아 있는 구성원 최대 체력 회복(%). 0이면 끔 (4단계: 레벨업 회복 대체)
UltGaugeMax,100,궁극기 게이지 최대
UltGaugeBasic,4,기본 공격 적중 1회당 궁극기 게이지
UltGaugeSkill,6,스킬 시전 1회당 궁극기 게이지 (궁극기 자신 제외)
```
- `LoadRunConfig`에 같은 이름 속성으로 읽는다(`n("StageClearHealPct", ...)` 방식). `StageLevelUp` 행 설명은 `하단 바를 웨이브 진행으로 표시 (4단계부터 레벨업 없음)`으로 바꾼다.

### 3-5. 아이콘 임시값 (4-A, 빠진 리소스)

| 스킬 | 문제 | 임시 처리 (`HeroSkillTable.csv`·`JobSkillTable.csv`의 `IconRUID`) |
|---|---|---|
| V01 서머솔트 킥 | 두 표 모두 빈 값 → HUD·캐릭터 화면 칸 비어 보임 | `c06426910ef343ee99dcc0d6ccf61e07`(V03 스크류 펀치 아이콘, 4단계부터 V03은 칸에 없음) |
| N15 다크로드의 비전서 | N14 풍마수리검과 같은 아이콘 → 5차 HUD에 같은 그림 2개 | `2e459dfac38b463790f45e7b93318dac`(N05 다크 플레어 아이콘, 칸에 없음) |

- 같은 그림이지만 동시에 보이지 않는 쌍(V04/V05, A02/A03, N09/N10)은 그대로. 정식 아이콘은 빠진 리소스 목록에 남긴다.

### 3-6. `DealTable.csv` 설명 3줄 (4-B)

`A_Stone`: `돌파 선택 1회를 얻습니다` / `D_Stone`: `돌파 선택 3회를 얻습니다\n대가: 현재 체력의 50% 소모` / `M_Stone`: `돌파 선택 2회를 얻습니다`. 다른 열은 그대로.

---

## 4. 작업 단위 상세

### 4-A. 판 시작 정리 + 직업 표시 + 디버그

**바꿀 파일**: `Character/PlayerRoster.mlua`, `Tower/TowerRunManager.mlua`, `Tower/JobCombat.mlua`(속성 1줄), `Data/HeroSkillTable.csv`·`Data/JobSkillTable.csv`(아이콘 2칸)

1. **`PlayerRoster:RunPartyData()`** (`@ExecSpace("ServerOnly")`, 신규)
   ```
   -- 판 시작용 편성 3명: PlayerEngrave 편성을 FixParty와 같은 규칙(표에 없음·잠김·중복 → 표 순서의 해금된 미사용 직업)으로 보정한 CharRunData 3개.
   -- 탑에서도 동작하며 PlayerEngrave에는 쓰지 않는다(저장 없음). 반환 { list = {CharRunData×3}, fixed = boolean, jobs = "a,b,c" } / 불러오기 전이면 nil
   ```
   - `FixParty`의 보정 부분(`used`/`bad` 계산 + 표 순서 채우기)을 내부 함수 `FixedJobs(cur)`로 떼어 `FixParty`와 같이 쓴다(동작 동일 유지).
2. **`TowerRunManager:LoadRunParty()`** (신규, `ServerOnly`) — `StartRun`의 계정 확인 직후, **`Heroes()` 정리보다 먼저** 부른다.
   - `roster = self._T.player:GetComponent("script.PlayerRoster")`. `roster == nil` 또는 `not roster.RosterLoaded` → `self._T.runParty = nil`, `log_warning("[VRF] run party roster not ready, legacy formation")` 후 기존 `LoadFormation()` 그대로.
   - 아니면 `local rp = roster:RunPartyData()`; `self._T.runParty = rp.list`, `self._T.runData = { [job] = data }`. `rp.fixed`면 `log("[VRF] run party fixed " .. rp.jobs)`. 항상 `log("[VRF] run party " .. rp.jobs .. " lv=" .. 레벨들 .. " tier=" .. 차수들)`.
   - `LoadFormation()`은 `self._T.runParty`가 있으면 그 순서로 `_T.mainJob/_T.mateJobs/CompJobId/Comp2JobId`를 정하고 PlayerEngrave를 읽지 않는다(없을 때만 기존 코드).
   - 새 함수 `RunDataFor(string job) → table` (`ServerOnly`): `self._T.runData[job]` 또는 nil.
3. **판 시작 순서 정리 (`StartRun`)**: ① 계정 확인 → ② `LoadRunParty()` → ③ `LoadFormation()` → ④ 플레이어 엔티티의 `HeroCombat`·`BishopCombat`·`JobCombat` **셋 모두** `ClearState()` + 동료 `Heroes()` 정리 → (기존 흐름). `SpawnCompanion` 안의 `LoadFormation()` 재호출은 `_T.runParty`를 그대로 쓰므로 결과가 같다.
4. **Q7**: `Tower/JobCombat.mlua` `property string JobId`에 **`@Sync`** 추가. 서버에서 바꾸는 곳(`HeroOf`)은 그대로.
5. **아이콘**: 3-5 표 두 칸.
6. **3단계 보완 Q5 일부**: `StartRun`에서 `_T.startStars` 저장(그 절 (f)). (3단계 보완을 먼저 끝냈다면 이미 되어 있다.)
7. **디버그 함수** (`TowerRunManager`, 전부 `@ExecSpace("ServerOnly")`, 로그 `[VRF] debug ...`)

   | 함수 | 동작 |
   |---|---|
   | `DebugStartFloor(integer floor)` | `self._T.debugFloor = floor` 후 `StartRun()`. `StartRun`은 `_T.debugFloor`가 있으면 열린 층 제한 없이 그 층으로 시작하고 값을 지운다. 탑 맵에서 `RunState`가 `READY` 이후일 때 사용 |
   | `DebugClearWave()` | 대기열 몬스터 `KillQueued`, `_T.waveLeft = 0`, 남은 묶음 비우기, 맨 앞 몬스터에 `ApplyMonsterDamage(MonsterHp + 1)` |
   | `DebugFinishFloor()` | `FIGHT`일 때 `Stage = StagesPerFloor`, `StageKind = "boss"`, `_T.waveNo = WavesPerStage` 후 `DebugClearWave()` → 보스 처치 경로로 층 클리어·결과 화면. 경로가 막히면 3단계 테스트 방식(`ApplyMonsterDamage(999999)` 반복)으로 대체하고 보고 |
   | `DebugGod(boolean on)` | `_T.debugGod = on`. `ApplyPlayerDamage`·`ApplyMateDamage` 첫 줄에서 `if self._T.debugGod then return end` |
   | `DebugSetBonus(string effType, number v)` | `_T.debugBonus[effType] = v`(StartRun에서 지우지 않음). `Bonus()` 끝에 `total = total + ((self._T.debugBonus or {})[effType] or 0)` |
   | `DebugDumpParty()` | 구성원마다 `[VRF] party k=1 job=Viper tier=4 lv=20 label=바이퍼 atk=.. maxHp=.. def=.. equip=.. hidden=.. passiveLv=.. ult=..` (4-B~4-E 값이 생기면 함께 출력) |
   | `DebugDpsProbe(number sec)` | `_T.probe = { t0 = 지금, dealt = 0, taken = 0, floor, stage }`. `ApplyMonsterDamage`·`ApplySplashDamage`(실제 깎인 양)와 `ApplyPlayerDamage`·`ApplyMateDamage`(실제 받은 양)에서 누적. `sec` 뒤 `_TimerService:SetTimerOnce`로 `[VRF] probe floor=.. stage=.. sec=.. dealt=.. dps=.. taken=.. normalHp=..`(`normalHp = MonsterBaseHp(현재 층 첫 몬스터 정보)`) 출력 후 `_T.probe = nil` |
   | `DebugFillUlt()` | (4-D에서 추가) 구성원 전원 `UltGauge = UltMax` |
   | `DebugSwapSlot(string job, string slot, string skillId)` | (4-F에서 추가) |

8. **기준 측정(4-B 전에 1회)**: 지금 코드(판 레벨 있음)로 테스트 계정 편성 그대로 20층 1스테이지에서 `DebugGod(true)` + `DebugDpsProbe(30)` → `dps / normalHp` = **R0**를 보고서에 적는다(4-G 기준).

**완료 조건**: 빌드 오류 0, `LEA-` 0, T1~T7 통과.

### 4-B. 판 안 레벨 삭제 + 구성원별 캐릭터 스탯

**바꿀 파일**: `Tower/TowerRunManager.mlua`, `Tower/MainPanelLogic.mlua`(정보 탭 2줄), `Data/CharJobTable.csv`(3-1), `Data/RunConfig.csv`(3-4 중 `StageClearHealPct`), `Data/DealTable.csv`(3-6)

1. **레벨업 삭제** (2-1 #2·#3·#6): 5370행 `DoLevelUp(1, 0)` 줄 삭제. `StartRun`: `startLevel` 계산·`ExpToNext` 공식 삭제, `self.Level = 메인 runData.level`(없으면 1), `self.Exp = 0`, `self.ExpToNext = 1`, `self.PendingPicks = 0`. 로그 `run start`의 `lv=`는 메인 레벨.
2. **스테이지 클리어 처리** (5372행 부근, `not isBoss` 블록):
   ```lua
   if self:IsHeroRun() and not isBoss then
   	self.PendingPicks = 1
   	-- 각인·은총 '추가 카드 확률'(ExtraPick): 4단계부터 돌파 선택 +1 확률
   	if _UtilLogic:RandomDouble() * 100 < self:Bonus("ExtraPick") and self:HasBreakLeft() then self.PendingPicks = 2 end
   end
   ```
   그리고 `ReviveDowned()` 다음에 회복:
   ```lua
   -- 4단계: 레벨업 전체 회복 대신 전투 스테이지 클리어 회복 (RunConfig StageClearHealPct)
   if self.StageClearHealPct > 0 then
   	for _, a in ipairs(self:Actors()) do
   		if not self:IsDown(a) then self:HealActor(a, self:ActorMaxHp(a) * self.StageClearHealPct / 100) end
   	end
   end
   ```
3. **판 시작 돌파** (`PickJob` 파티 분기): `self.PendingPicks = 0` → `self.PendingPicks = math.max(0, math.floor(self:Bonus("StartLevel") + 0.5))`. 0보다 크면 기존 분기(`RollChoices` → `RollPartyBreaks`)로 첫 전투 전에 돌파 카드가 나온다. 로그 `[VRF] start breaks n=..`.
4. **`HasBreakLeft()`** 신규: `Heroes()`의 장착 스킬 중 `#RemainingBreaks(sk) > 0`이 하나라도 있으면 true. `HasSkillToLearn()`을 부르던 거래 `SkillPick`(4134행)·`DoLevelUp`을 이것으로 바꾼다.
5. **`RecalcStats` 구성원별** (전체 교체):
   ```
   -- 구성원별 기본값 = 캐릭터 스탯(PlayerRoster CharRunData) × 계정 % + 장비(공용 부위) 고정값
   for idx, actor in ipairs(self:Actors()) do
   	local d = 그 구성원 직업의 RunDataFor(job)  -- 없으면 { atk = BasePlayerAtk, hp = BasePlayerMaxHp, def = 0 } + 경고 1회
   	atkBase = d.atk * (1 + AccountAtk/100) + weapon          -- weapon = EquipStat("Weapon") × (1 + WeaponPct/100) (지금과 같음)
   	hpBase  = d.hp  * (1 + AccountHp/100)  + EquipStat("Cap") * armorMul
   	defBase = d.def * (1 + AccountDef/100) + EquipStat("Coat") * armorMul
   	self._T.actorBase[actor] = { atk = atkBase, hp = hpBase, def = defBase }
   end
   ```
   - 메인: `self._T.goddessAtkBase/HpBase/DefBase`는 **메인 값**으로 계속 채운다(여신 효과 등 기존 코드 호환). `PlayerMaxHp`는 지금 공식에서 `hp` 대신 메인 `hpBase`.
   - 동료: `maxHp = hpBase_i × (1 + (HpPct + GoddessBonus(comp,"HpPct"))/100)`, 증가분만큼 현재 체력 올리기(지금 코드 그대로).
   - `ActorAtk(actor)`: `local base = (self._T.actorBase[actor] or {}).atk or self._T.goddessAtkBase` (나머지 동일). `ActorDefense(actor)`도 `.def`로 같은 방식. → `ActorHitDamage`·전투 3종 `Strike`·받는 피해(3471행 `ActorDefense(victim)`)가 자동으로 구성원별 값이 된다.
   - `PlayerAtk/PlayerAtkMin/PlayerDef`(Sync, 판 패널 표시)는 메인 값.
   - `SpawnCompanion`이 `RecalcStats`보다 뒤에 오므로(StartRun 2930~2934행) **`SpawnCompanion()` 직후 `RecalcStats()`를 한 번 더** 부르고 `FullHealParty()`. (`PickJob`에서도 다시 부르므로 결과는 같다.)
6. **`MainPanelLogic`** 정보 탭(766~767행): `레벨` 줄 → `"레벨        Lv." .. mgr.Level .. " (메인)"`, `경험치` 줄 삭제. 하단 `Lv.`·경험치 바는 그대로(웨이브 표시).
7. (선택) 처치마다 `self.Exp = math.min(self.ExpToNext, self.Exp + 1)` — 웨이브 막대가 실제로 차게. `BeginWave`가 `ExpToNext = size`로 다시 맞춘다.
8. 데이터 3-1(공격력), 3-4 `StageClearHealPct`, 3-6(거래 문구).

**완료 조건**: 빌드 오류 0, `LEA-` 0, 판 중 `[VRF] level up` 로그 0, T8~T14 통과.

### 4-C. 칸 진화 스킬 (전투 3종 `BeginRun` 교체)

**바꿀 파일**: `Data/CharSkillSlotTable.*`(신규), `Tower/TowerRunManager.mlua`, `Tower/HeroCombat.mlua`, `Tower/BishopCombat.mlua`, `Tower/JobCombat.mlua`, `Tower/BattleHUDLogic.mlua`

1. **표 읽기** (`TowerRunManager.LoadJobTables` 끝): `CharSkillSlotTable` → `self._T.slotTune[job .. "." .. slot .. "." .. tier] = { id, pctMul, cdSec, mana, triggerAs }`. 함수 `SlotTune(job, slot, tier) → table`(없으면 `{ pctMul = 1, cdSec = 0, mana = -1, triggerAs = "" }`). 로그 `[VRF] slot tune rows=75`.
2. **칸 순서** 상수 `{ "Main", "Sub", "Tech", "Hyper", "Ult" }` (패시브 제외, HUD 칸 S1~S5와 같은 순서).
3. **전투 3종 `BeginRun(mgr)` 공통 흐름** (세 파일 같은 구조로):
   ```
   1) (JobCombat) JobId가 바뀌었으면 spec 초기화. 세 파일 모두 LoadSkills()를 매 판 다시 불러 원래 수치로 되돌린다(덮어쓰기가 판마다 쌓이지 않게).
   2) ClearState(); self._T.mgr = mgr
   3) local d = mgr:RunDataFor(self.JobId)
      d == nil → 기존 코드(SkillRunDataFor / StartEquip 대체)로 진행 + log_warning("[VRF] <job> run data missing, legacy equip")  ← 안전장치. 정상 판에서는 나오면 안 됨
   4) self._T.tier = d.tier; self.RunLevel = d.level; self.RunLabel = d.tierName
      패시브 레벨(4-C 임시: 2×tier−1 → 4-E에서 CharPassiveTable 값): 히어로 ComboLv / 비숍 HolyLv / 직업 공용 PassiveLv
   5) for _, key in ipairs(칸 순서) do
        local id = d.slots[key] or ""
        if id ~= "" and self._T.skills[id] ~= nil then
          table.insert(self._T.equip, id); self._T.slotOf[id] = key
          local t = mgr:SlotTune(self.JobId, key, d.tier)
          self._T.pctMul[id] = t.pctMul
          local sk = self._T.skills[id]
          if t.cdSec > 0 then sk.cooldown = t.cdSec end
          if t.mana >= 0 then sk.mana = t.mana end
          if t.triggerAs == "Cooldown" then sk.trigger = "Cooldown" end      -- 하이퍼 (B10만 실제로 바뀜)
          -- t.triggerAs == "Ult" 는 4-D에서 처리 (4-C에서는 무시)
          mgr.SkillLv[id] = 1
        end
      end
   6) (기존) Active = true, ManaMax, Mana, 리소스 미리 불러오기
   7) log("[VRF] slots run job=" .. JobId .. " tier=" .. tier .. " lv=" .. level .. " equip=" .. concat(equip) .. " pct=" .. concat(pctMul))
   ```
   - 스킬 레코드의 쿨타임·마나 필드 이름은 파일마다 다르다(Hero/Bishop: `sk.cooldown`, `sk.mana`, JobCombat: `sk.cooldown`, `sk.mana` — `LoadSkills`에서 확인 후 맞춘다). JobCombat `Params(sk)`는 `sk.cooldown/sk.mana`에서 시작하므로 덮어쓴 값이 돌파 전 기본이 된다.
   - `_T.ownLv` 사용 제거. **`LvMul(id)` → `return self._T.pctMul[id] or 1`** (세 파일).
   - 속성 추가(세 파일): `@Sync property integer RunLevel = 0`, `@Sync property string RunLabel = ""`. `ClearState`에서 0/"".
   - `HeroCombat`: `self.ComboLv = 2×tier−1`(콤보 어택은 1차부터 켜짐). `BishopCombat`: `self.HolyLv = 2×tier−1`. `JobCombat`: `self.PassiveLv = 2×tier−1`.
4. **로비 데이터 끊기**: 세 `BeginRun`에서 `eng:SkillRunData*` 호출은 3)의 안전장치 분기 안에만 남는다. 완료 후 `Tower/` 폴더에서 `SkillRunData` 검색 결과가 그 분기 3곳뿐이어야 한다.
5. **HUD** (`BattleHUDLogic.UpdateMember` 636행): `local label = (cc.RunLabel ~= "" and cc.RunLabel) or names[cc.JobId] or cc.JobId`, `local lv = cc.RunLevel > 0 and cc.RunLevel or mgr.Level` → `tostring(k) .. "  " .. label .. "  Lv." .. tostring(lv)`. 자원 줄(`cc.JobId` 분기)은 4-A의 `@Sync`로 맞게 나온다. 칸 아이콘은 `SlotInfo`가 `_T.equip` 순서라 S1=주력 … S5=궁극기. `Lv` 글자: `SlotInfo` 넷째 값을 0으로 써서 비운다(스킬 레벨 없음) — 세 파일 `UpdateStatus`의 `mgr.SkillLv[id]` 자리를 `0`으로.
6. `JobSkills()`·`RemainingBreaks`·`RollPartyBreaks`·`OpenBook`은 `h:Equipped(id)`를 쓰므로 자동으로 칸 스킬 기준이 된다(수정 없음).

**완료 조건**: 빌드 오류 0, `LEA-` 0, T15~T20 통과, `[VRF] slot tune mismatch` 0건.

### 4-D. 하이퍼 + 궁극기 게이지 + HUD

**바꿀 파일**: `Tower/HeroCombat.mlua`, `Tower/BishopCombat.mlua`, `Tower/JobCombat.mlua`, `Tower/TowerRunManager.mlua`(설정 3개·`DebugFillUlt`), `Tower/BattleHUDLogic.mlua`, `Data/RunConfig.csv`(3-4 궁극기 3행)

1. **하이퍼**: 4-C의 표 덮어쓰기로 이미 "쿨 40·마나 0·쿨타임형 자동"이다(B10은 `Skill`→`Cooldown`, V10은 `Auto` 그대로 쿨 40). 4-D에서는 확인과 로그만: 하이퍼 시전 때 `[VRF] hyper cast job=.. id=..`(세 파일의 `PickActed` 쿨타임 분기 / JobCombat `TickAuto`에서 `self._T.slotOf[sk.id] == "Hyper"`일 때).
2. **궁극기 속성** (세 파일): `@Sync property number UltGauge = 0`, `@Sync property number UltMax = 100`. `ClearState`에서 `UltGauge = 0`, `_T.ultId = nil`.
3. **`BeginRun`**: 표 `TriggerAs == "Ult"`인 칸 스킬 → `self._T.ultId = id`, `sk.trigger = "Ult"`(일반 시전 분기 `Cooldown/Skill/Auto/Guard/Instinct`에서 자동으로 빠짐), `sk.mana = 0`. `self.UltMax = mgr.UltGaugeMax`.
4. **게이지 충전** (세 파일 공통 메서드 `AddUlt(number n)`: `ultId`가 있고 `UltGauge < UltMax`일 때만 `math.min(UltMax, UltGauge + n)`):
   - 기본 공격 적중: `RollCasts(mgr)` 첫 부분(기본 공격 1회 명중 때 불림)에 `self:AddUlt(mgr.UltGaugeBasic)`.
   - 스킬 시전: `PickActed`에서 궁극기가 아닌 스킬을 `list`에 넣을 때마다 `self:AddUlt(mgr.UltGaugeSkill)`. JobCombat `TickAuto`의 `Auto` 발동에도 같은 값.
5. **궁극기 시전**: 세 파일 `PickActed` **맨 앞**:
   ```lua
   local uid = self._T.ultId
   if uid ~= nil and self.UltGauge >= self.UltMax then
   	local sk = self._T.skills[uid]
   	if sk ~= nil then
   		self.UltGauge = 0
   		table.insert(list, self:MakeCast(sk))   -- 마나 소모 없음, 쿨타임 기록 없음
   		log("[VRF] ult cast job=" .. self.JobId .. " id=" .. uid)
   		return                                  -- 이번 차례의 동작 스킬은 궁극기 1개
   	end
   end
   ```
   - 바이퍼 V15의 에너지 50 소비처럼 **자원 조건이 시전을 막으면 안 된다**: `MakeCast`/`Params` 안에서 `trigger` 또는 자원 부족으로 막는 곳이 있으면 궁극기일 때는 "있는 만큼 소비"로 처리한다(JobCombat `V15`의 `useEnergyN` 경로 확인).
   - 세 파일에서 `trigger == "Skill"`을 "직접 공격"의 뜻으로 쓰는 곳(발할라 충전·트라이엄프 페더 등)을 검색해 궁극기(`"Ult"`)도 포함할지 판단한다. 기본은 **포함**(궁극기도 직접 공격).
6. **HUD 게이지** (`.ui` 변경 없음): 세 파일 `UpdateStatus`가 궁극기 칸을 `id:(UltMax-UltGauge):UltMax:U` 형식으로 쓴다(넷째 값 `U` = 궁극기 표시). `BattleHUDLogic.UpdateMember`:
   - 파싱 패턴은 그대로 두고 넷째 원문을 `lvRaw`로 따로 받는다.
   - `lvRaw == "U"`이면 덮개 비율 = `left / max`(기존 계산 그대로 → 차오를수록 덮개가 줄어듦), `CdText` = 가득이면 `""`, 아니면 `math.floor((max - left) / max * 100) .. "%"`, `Lv` 글자 `""`.
7. `TowerRunManager`: 속성 `UltGaugeMax=100, UltGaugeBasic=4, UltGaugeSkill=6`(RunConfig로 덮어씀), `DebugFillUlt()`.
8. 게이지는 판 시작 때 0, 스테이지가 바뀌어도 유지, 쓰러진 구성원은 충전·시전하지 않는다(기존 시전 흐름이 쓰러진 구성원을 부르지 않으면 그대로).

**완료 조건**: 빌드 오류 0, `LEA-` 0, T21~T25 통과.

### 4-E. 고유 패시브 차수별 효과

**바꿀 파일**: `Data/CharPassiveTable.*`(신규), `Tower/TowerRunManager.mlua`(표 읽기), 전투 3종

1. **표 읽기**: `self._T.passive[job .. "." .. tier] = { lv, grants = {…}, extra = { 키 = true } }`, 함수 `PassiveOf(job, tier)`.
2. **`BeginRun`**(4-C 흐름 4)의 임시 공식을 표 값으로: `ComboLv / HolyLv / PassiveLv = p.lv`.
3. **숨은 장착(흡수)**: 칸 스킬을 넣은 뒤 `for _, id in ipairs(p.grants)`: 스킬이 있고 아직 장착이 아니면 `_T.equip`에 추가, `_T.hidden[id] = true`, `_T.slotOf[id] = "Passive"`, `mgr.SkillLv[id] = 1`. 기존 `Instinct`(확률 발동)·`Guard`(위기 자동) 처리 코드가 그대로 돈다.
   - `UpdateStatus`의 `SlotInfo`는 `_T.hidden` 스킬을 **빼고** 만든다(HUD 칸은 5개 그대로).
   - 숨은 스킬도 돌파 카드·마법의 서 대상이다(패시브 칸 돌파, 4-F 키 = 스킬 Id).
   - 발동 로그: 숨은 스킬이 처음 발동할 때마다 `[VRF] passive proc job=.. id=..`(Instinct/Guard 발동 지점 각 1줄).
4. **Extra 처리**
   | 키 | 파일·함수 | 처리 |
   |---|---|---|
   | `ComboCap10` | `HeroCombat.ComboCap` | 돌파 조건 검사 뒤 마지막에 `if self._T.pExtra.ComboCap10 then cap = math.max(cap, self.ComboMax) end` (ComboMax=10, 데스폴트 돌파 15는 그대로 우선) |
   | `ComboGain50` | `HeroCombat.RollComboGain` | `chance = chance * 1.5` |
   | `HolyMax10` | `BishopCombat.HolyMax` | `math.max(지금 값, 10)` |
   | `ResMax8` / `ResMax15` | `JobCombat.CalcResMax` | 나이트로드 `math.max(값, 8)`, 보우마스터 `math.max(값, 15)` |
   | `EnergyPlus3` | `JobCombat` 에너지 획득(타수당 `EnergyPerHit`) | `EnergyPerHit + 3` |
5. `DebugDumpParty`에 `passiveLv`, `hidden`, `extra` 출력.

**완료 조건**: 빌드 오류 0, `LEA-` 0, T26~T30 통과.

### 4-F. 돌파를 칸에 연결

**바꿀 파일**: `Tower/TowerRunManager.mlua`, 전투 3종(칸 등록 1줄씩)

1. **칸 등록**: 전투 3종 `BeginRun` 끝에 `mgr:RegisterSlots(self.JobId, self._T.slotOf)` → `self._T.slotKeyOf[skillId] = job .. "." .. slot`(패시브 숨은 스킬은 스킬 Id 그대로 키).
2. **키 함수**: `BreakKey(skillId)` = `self._T.slotKeyOf[skillId] or skillId`. `BreakNo(breakId)` = `_B` 뒤 숫자(`string.find` + `sub`, `gsub` 금지).
3. **저장**: `_T.breakTaken[BreakKey(skillId)][BreakNo(breakId)] = true` (지금은 `[스킬Id][돌파Id]`). 바꿀 곳: `RequestPick`(BREAK 분기), `PickBookBreak`, `DebugGiveBreaks`, `HeroBreak`, `RemainingBreaks`, `UpdateSkillInfo`, `OpenBook`(얻은 수 세기).
4. **판정**: `HeroBreak(skillId, breakId)` → `local t = self._T.breakTaken[self:BreakKey(skillId)]; return t ~= nil and t[self:BreakNo(breakId)] == true`. 전투 3종 `Has(id, n)`은 수정 없음.
5. **표시용 Sync**(`SkillBreaks`, `SkillInfo`)는 **현재 칸 스킬 Id**로 계속 쓴다(UI 호환).
6. **`DebugSwapSlot(job, slot, skillId)`**: 그 직업 구성원 전투 컴포넌트의 `_T.equip`에서 그 칸 스킬을 새 Id로 바꾸고 `slotOf`/`pctMul`/`slotKeyOf` 갱신(판 중 진화 흉내). 로그 `[VRF] debug swap job.slot old->new`.
7. 규칙 문장(코드 주석에도): **"칸에 n번 돌파가 붙어 있으면, 그 칸에 지금 들어 있는 스킬의 n번 돌파가 켜진 것으로 본다. 새 스킬에 n번 돌파가 없으면(예: B6) 효과 없이 기록만 남는다."** 히어로식 스킬당 최대 3개 제한은 칸 기준으로 센다.

**완료 조건**: 빌드 오류 0, `LEA-` 0, T31~T33 통과.

### 4-G. 밸런스 측정 + 회귀

**바꿀 파일**: 원칙적으로 데이터만(`CharJobTable` 공격력, `CharSkillSlotTable` `PctMul`, `RunConfig` `StageClearHealPct`). 코드 수정은 버그일 때만.

1. 7절 표의 4개 조합을 `DebugDpsProbe(30)`으로 측정해 `dps / normalHp`를 4-A의 R0와 비교한다(T34).
2. 기준을 벗어나면 **공격력 열만** 일괄 배율 조정(예: `BaseAtk`·`AtkPerLv` ×1.1)하고 다시 측정. 조정 내용은 보고서에 표로.
3. 1층 통짜 회귀(T37): 디버그 없이 1층을 처음부터 끝까지(무적 끔) — 돌파 카드·정비소·보스·결과 팝업·다이아(첫 클리어가 아니면 0)·별.

**완료 조건**: T34~T38 통과, 전체 `LEA-` 0, `git diff --stat`에 금지 파일 변경 없음.

---

## 5. 진화표 ↔ 스킬 Id 매핑 (5직업 × 모든 단계)

출처: 마스터 스펙 3절 표 + `CharSlotTable.csv` + `JobSkillTable.csv`(발동 방식·쿨·마나). **진화표의 스킬은 모두 기존 데이터에 있고 전투 코드도 있다 → 임시 대체 스킬은 없다.** 문제는 아이콘 2개(3-5)뿐.

| 직업 | 칸 | 1차 | 2차 | 3차 | 4차 | 5차 |
|---|---|---|---|---|---|---|
| 히어로 | 패시브 | P_Hero 콤보 어택 Lv1 | Lv3 | Lv5 + H15 | Lv7 + 콤보 10 | Lv9 + 획득 +50% |
| | 주력기 | **H01** 슬래시 블러스트 (Skill, 쿨5, 마나8) | **H02** 브랜디쉬 (Skill, 5, 10) | H02 | **H08** 레이징 블로우 (Skill, 10→**5**, 20) | H08 |
| | 보조기 | — | **H05** 인레이지 (Cooldown 25, 버프) | **H11** 에픽 어드벤쳐 (Cooldown 40) | H11 | H11 |
| | 기술 | — | — | **H03** 패닉 (Skill, 6, 20) | H03 | **H13** 콤보 데스폴트 (Skill, 20, 100) |
| | 하이퍼 | — | — | — | **H10** 발할라 (Cooldown 20→**40**) | H10 |
| | 궁극기 | — | — | — | — | **H12** 소드 오브 버닝 소울 (게이지) |
| 비숍 | 패시브 | P_Bishop 성수 Lv1 | Lv3 | Lv5 + B12 | Lv7 + B04 | Lv9 + 성수 10 |
| | 주력기 | **B01** 에너지 볼트 (5, 8) | **B02** 홀리 애로우 (5, 10) | B02 | **B06** 엔젤레이 (10→**5**, 20) | B06 |
| | 보조기 | — | **B11** 어드밴스드 블레스 (Cooldown 30) | **B09** 인피니티 (Auto 25) | B09 | B09 |
| | 기술 | — | — | **B03** 샤이닝 레이 (7, 25) | B03 | **B15** 디바인 퍼니시먼트 (20, 100) |
| | 하이퍼 | — | — | — | **B10** 헤븐즈 도어 (Skill 30·마나60 → **Cooldown 40·마나0**) | B10 |
| | 궁극기 | — | — | — | — | **B13** 엔젤 오브 제미니 (게이지) |
| 나이트로드 | 패시브 | P_NightLord 표식 Lv1 | Lv3 | Lv5 + N13 | Lv7 + N09 | Lv9 + 표식 8 |
| | 주력기 | **N01** 럭키 세븐 (5, 8) | **N03** 트리플 스로우 (5, 15) | N03 | **N06** 쿼드러플 스로우 (10→**5**, 20) | N06 |
| | 보조기 | — | **N02** 드레인 (Skill 5, 10) | **N04** 쉐도우 파트너 (Auto 25) | N04 | N04 |
| | 기술 | — | — | **N07** 써든레이드 (5, 12) | N07 | **N14** 풍마수리검 (12→**8**, 50) |
| | 하이퍼 | — | — | — | **N11** 블리딩 톡신 (Cooldown 20→**40**) | N11 |
| | 궁극기 | — | — | — | — | **N15** 다크로드의 비전서 (게이지, 아이콘 임시) |
| 보우마스터 | 패시브 | P_Bowmaster 강화 화살 Lv1 | Lv3 | Lv5 + A13 | Lv7 + A09 | Lv9 + 화살 15 |
| | 주력기 | **A01** 애로우 블로우 (5, 8) | **A02** 애로우 봄 (5, 10) | A02 | **A06** 폭풍의 시 (10, 25, 4초 연사) | A06 |
| | 보조기 | — | **A05** 애로우 플래터 (Cooldown 25, 설치) | **A04** 피닉스 (Cooldown 30, 소환) | A04 | A04 |
| | 기술 | — | — | **A07** 애로우 일루전 (8, 20) | A07 | **A14** 실루엣 미라주 (12, 50) |
| | 하이퍼 | — | — | — | **A10** 애로우 레인 (Cooldown 30→**40**) | A10 |
| | 궁극기 | — | — | — | — | **A15** 잔영의 시 (게이지) |
| 바이퍼 | 패시브 | P_Viper 에너지 Lv1 | Lv3 | Lv5 + V13 | Lv7 + V09 | Lv9 + 에너지 +3 |
| | 주력기 | **V01** 서머솔트 킥 (5, 8, 아이콘 임시) | **V02** 백 스핀 블로우 (5, 10) | V02 | **V07** 데몰리션 (8, 20) | V07 |
| | 보조기 | — | **V05** 트랜스폼 (Cooldown 30) | **V14** 퓨리어스 차지 (Cooldown 30) | V14 | V14 |
| | 기술 | — | — | **V04** 에너지 버스트 (8, 20) | V04 | **V12** 하울링 피스트 (12, 40) |
| | 하이퍼 | — | — | — | **V10** 스티뮬레이트 (Auto 25→**40**) | V10 |
| | 궁극기 | — | — | — | — | **V15** 리버레이트 넵투누스 (게이지, 에너지 소비는 있는 만큼) |

- 4단계부터 어떤 칸에도 없는 스킬(돌파 포함, 판에 나오지 않음): H04·H06·H07·H09·H14, B05·B07·B08·B14, N05·N08·N10·N12, A03·A08·A11·A12, V03·V06·V08·V11. 데이터는 지우지 않는다(6단계 각성에서 재활용 검토).
- 패시브로 흡수되는 스킬: 확률 발동 H15·B12·N13·A13·V13, 위기 자동 B04·N09·A09·V09(히어로는 위기 스킬이 없어 콤보 최대치 강화로 대신).

---

## 6. 고유 패시브 1~5차 효과표

패시브 레벨 효과는 지금 전투 코드 공식 그대로다(레벨만 차수로 정해짐).

| 직업 | 레벨 공식(기존 코드) | 1차 (Lv1) | 2차 (Lv3) | 3차 (Lv5) | 4차 (Lv7) | 5차 (Lv9) |
|---|---|---|---|---|---|---|
| 히어로 콤보 어택 | 콤보 1당 공격력 `1 + 0.2×(Lv−1)`%, 기본 공격 적중 시 30% 콤보, 최대 5 | 콤보당 1% | 1.4% | 1.8% + **콤보 인스팅트**(H15: 기본 공격 적중 시 발동, 10초간 콤보당 피해 +5%·균열) | 2.2% + **콤보 최대 10** | 2.6% + **콤보 획득 확률 30→45%** |
| 비숍 성수 | 성수 1당 공격력 `1 + 0.2×(Lv−1)`%, 최대 5 | 1% | 1.4% | 1.8% + **프레이**(B12: 기본 공격 적중 10% 발동, 파티 회복 +50%·기도의 빛) | 2.2% + **홀리 매직쉘**(B04: 파티원 체력 50% 아래면 자동, 피해 3회 막기·회복) | 2.6% + **성수 최대 10** |
| 나이트로드 표식 | 표식 1중첩당 피해 `1.2 + 0.2×(Lv−1)`%, 폭발 `60 + 10×(Lv−1)`%, 최대 5 | 1.2% / 60% | 1.6% / 80% | 2.0% / 100% + **쉐도우 스피어**(N13: 10% 발동, 그림자 창 추가타) | 2.4% / 120% + **다크 사이트**(N09: 체력 40% 이하 순간 2초 은신·다음 공격 치명타) | 2.8% / 140% + **표식 최대 8** |
| 보우마스터 강화 화살 | 화살 1개당 공격력 `0.6 + 0.1×(Lv−1)`%, 가득 차면 강화 사격 `50 + 10×(Lv−1)`%, 최대 10 | 0.6% / 50% | 0.8% / 70% | 1.0% / 90% + **가이디드 애로우**(A13: 10% 발동, 유도 화살) | 1.2% / 110% + **일루전 스텝**(A09: 체력 40% 이하 순간 3초 회피 60%·넉백) | 1.4% / 130% + **화살 최대 15** |
| 바이퍼 에너지 | 에너지 50 이상 받는 피해 −`5 + 0.5×(Lv−1)`%, 100이면 공격력 +`6 + (Lv−1)`% | −5% / +6% | −6% / +8% | −7% / +10% + **서펀트 스크류**(V13: 10% 발동, 소용돌이·받는 피해 −10%) | −8% / +12% + **언위어링 넥타**(V09: 체력 40% 이하 순간 20% 회복·6초 피해 −30%) | −9% / +14% + **타격당 에너지 5→8** |

(공식은 `HeroCombat.ComboAtkPerStack`, `BishopCombat.HolyAtkPct`, `JobCombat` 565·576·577·676·1278행 기준. 구현 에이전트는 실제 공식이 표와 다르면 코드 값을 정본으로 보고하고 표를 고친다.)

---

## 7. 밸런스 기준

### 7-1. 지금 판의 강함 (근거)

- 몬스터(층 N, `FloorsPerStage=1`): 능력치 레벨 = `1층 몬스터 레벨(1) + (N−1) × StageLevelStep(1)` = **N**. 일반 몬스터 체력 `(20 + 10N) × 타입 × 스테이지 배율(1 → 2.5)`, 공격 `(5 + N) × 타입 × 스테이지 배율`. 정예 체력 ×5·공격 ×2, 보스 체력 ×20·공격 ×3.
- 지금 플레이어: 층과 무관하게 **판 레벨 1에서 시작해 전투 스테이지마다 +1, 보스 앞 Lv10**. 공격 `10 + 2(L−1)`, 체력 `100 + 10(L−1)`, 방어 `L−1` + 1단계 장비(무기 6.4·투구 52.8·옷 3.2) + 계정 %. 층 평균(Lv5.5) 공격 약 25, 보스 때 34. 장착 5스킬을 처음부터 다 쓴다(아웃게임 레벨 배율 ×1~3).
- 즉 지금 층 진행은 판 레벨이 아니라 **아웃게임 성장(스킬 레벨·슬롯·각인·유물·계정)** 이 결정한다. 테스트 계정(계정 Lv2~3, 스킬 낮은 레벨)이 20층을 깼다.

### 7-2. 환산 기준

- 공격 지수 `E = (구성원 평균 공격력 + 무기 6.4) × S(차수) × M(아웃게임 배율)`.
  - `S` = 차수별 스킬 구성의 초당 피해 비율(지금의 "5스킬 전부"를 1로): **1차 0.4 / 2차 0.7 / 3차 1.0 / 4차 1.2 / 5차 1.5** (주력기 1개 + 기본 공격 → 5칸 + 궁극기, 마나 한계 반영 대략값).
  - 기준점: 지금 테스트 계정 20층 보스 = 공격 34 × S 1 × M 1.3 ≈ 44, 몬스터 체력 지수 `20 + 10×20 = 220` → **깰 수 있는 경계 `E ≈ 0.2 × (20 + 10N)` → `N ≈ E/2 − 2`**.
- 지금 캐릭터 표(Lv1 공격 10)를 그대로 쓰면 Lv1 1차 `E = 16.4 × 0.4 = 6.6` → **2층 근처에서 막힌다.** 그래서 공격력 표를 3-1처럼 바꾼다(체력은 지금 표로도 충분: 생존 지수 `체력 ÷ (5 + N)`이 아래 모든 경우 8 이상).

### 7-3. 레벨별 층 진행 기대 (시작 3직업 평균, 3-1 공격력 표 기준)

| 캐릭터 레벨 (차수) | 평균 공격력 + 무기 | S | E | 캐릭터만 (M 1.0) | 지금 테스트 계정 수준 외부 성장 (M 1.3) | 생존 지수 (비숍 기준) |
|---|---|---|---|---|---|---|
| Lv1 (1차) | 34.4 | 0.4 | 13.8 | **약 5층** | 약 7층 | 143 ÷ 10 = 14 |
| Lv10 (2차) | 45.4 | 0.7 | 31.8 | **약 14층** | 약 19층 | 241 ÷ 19 = 13 |
| Lv20 (4차) | 64.4 | 1.2 | 77.3 | **약 37층** | 약 48층 | 405 ÷ 42 = 9.6 |
| Lv30 (5차) | 83.4 | 1.5 | 125.1 | **약 60층** | 약 79층 | 580 ÷ 65 = 8.9 |

- 나머지 층(60~97층)은 5단계 직업별 장비·큐브, 6단계 각성, 유물·각인으로 채운다.
- 전직을 미룬 경우(관문 레벨에서 1차 그대로 등)는 S가 낮아 위 표보다 크게 약하다 → 전직의 체감이 크다(의도).
- 참고(지금 표 그대로였다면): Lv10 2차 37.4 → 약 11층, Lv30 5차 108 → 약 79층 — 초반이 너무 막히고 후반이 너무 빠르다.

### 7-4. 4-G 측정 방법과 합격 기준

- 4-A에서 지금 코드로 **R0 = (20층 1스테이지 30초 피해량 ÷ 30) ÷ normalHp(20층)** 를 잰다.
- 4-G에서 아래 4개 조합을 같은 방법(1스테이지, `DebugGod(true)`, 30초, 웨이브가 비면 측정 중단 없이 다음 웨이브 포함)으로 재고 `R = dps ÷ normalHp`:

  | 조합 | 편성(모두 같은 레벨·차수) | 층 |
  |---|---|---|
  | P1 | 히어로·비숍·나이트로드 Lv1 1차 | 5 |
  | P2 | 같은 3직업 Lv10 2차 | 14 |
  | P3 | 같은 3직업 Lv20 4차 | 37 |
  | P4 | 같은 3직업 Lv30 5차 | 60 |

- **합격**: 네 조합 모두 `0.7 × R0 ≤ R ≤ 1.5 × R0`. 그리고 P1·P4를 무적 끄고 1스테이지(웨이브 2개)를 끝까지 버팀(쓰러져도 전멸만 아니면 됨).
- 벗어나면 3-1 공격력 두 열을 같은 배율로 조정(P1만 낮으면 `BaseAtk`, P4만 높으면 `AtkPerLv`)하고 다시 잰다. 계정 성장(M)은 테스트 계정 그대로 두고 보고서에 계정 레벨·각인 합계를 적는다.

---

## 8. 테스트

### 8-1. 공통 준비와 빠른 진행 방법

server_main (한 스크립트 안에서 기다림은 `wait()`):
```lua
local p = nil
for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
local r = p:GetComponent("script.PlayerRoster")
local eng = p:GetComponent("script.PlayerEngrave")
-- (로비에서) 차수·편성 설정
r:DebugSetChar("Viper", 1, 19, 4)
r:DoSetParty("Viper", "Hero", "Bishop")
_TeleportService:TeleportToMapPosition(p, Vector3(0, 0, 0), "OrbisTower")
wait(5)
local mgr = p.CurrentMap:GetComponent("script.TowerRunManager")
log("[VRF] t state=" .. mgr.RunState)
```
- **디버그 함수 실행 규칙 (4-B 보완)**: 메이커 스크립트에서 매니저 함수(`StartRun`·`ApplyMonsterDamage` 등)를 직접 부르면 여러 프레임에 걸쳐 실행되어 그사이 `OnUpdate`가 판 상태를 덮어쓸 수 있다. `Debug*` 함수는 내부에서 `_TimerService:SetTimerOnce(..., 0)`로 다음 틱에 한 번에 돌도록 바뀌었으니 그대로 부르면 되고, 그 밖의 매니저 함수는 스크립트에서 `_TimerService:SetTimerOnce(function() ... end, 0)` 안에서 실행한다. `Debug*`는 다음 틱에 실행되므로 결과 로그·값은 `wait(0.5)` 이상 뒤에 확인한다.
- 판을 다시 시작: 탑 안에서 `mgr:DebugStartFloor(층)` (열린 층 제한 없음). 차수를 바꿀 때는 `r:DebugSetChar(...)`(탑에서도 동작) 후 `DebugStartFloor` — 판 시작 때 다시 읽는다.
- 층 바로 끝내기: `mgr:DebugFinishFloor()`. 웨이브만: `mgr:DebugClearWave()`. 무적: `mgr:DebugGod(true)`.
- 판 결과 확인은 `[VRF]` 로그 + `mgr:DebugDumpParty()`. 클라이언트 값(HUD)은 `client` 문맥에서 `_EntityService:GetEntityByPath("/ui/BattleHUD/Members/P1/Job").TextGUIRendererComponent.Text` 등.
- 테스트 끝에 되돌리기: `r:DebugReset()`, `mgr:DebugGod(false)`, `mgr:DebugSetBonus` 쓴 키 0, 편성 `Hero,Bishop,NightLord`(로비에서 `DoSetParty`), 로비로 이동. 계정 진행(20층)은 바꾸지 않는다(3단계 보완 Q6).

### 8-2. 테스트 케이스

**4-A**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T1 잠긴 직업 방어 | 로비에서 `r:DebugSetChar("Viper",0,1,1)`(잠금) → 탑 이동 → 탑에서 `eng.MainJob = "Viper"` 직접 대입(로비 보정이 안 도는 곳) → `mgr:DebugStartFloor(1)` | `[VRF] run party fixed Hero,Bishop,NightLord`(또는 표 순서상 해금된 미사용 직업), `[VRF] party ready jobs=`에 Viper 없음. `eng.MainJob`은 바뀌지 않음(저장 안 함) |
| T2 중복 방어 | 탑에서 `eng.CompJob = eng.MainJob` → `DebugStartFloor(1)` | `run party fixed`, 세 직업이 서로 다름 |
| T3 메인 바이퍼 표시 | 로비에서 바이퍼 해금(`DebugSetChar("Viper",1,1,1)`), `DoSetParty("Viper","Hero","Bishop")` → 탑 이동 → 전투 시작 후 client에서 HUD 읽기 | `P1/Job` = `1  바이퍼  Lv.…`, `P1/State` = `에너지 …`, `P1/S1/Icon` 켜짐·RUID `c0642691…`(임시). 서버 `[VRF] jobcombat run job=Viper` |
| T4 메인 보우마스터 표시 | 보우마스터 해금, `DoSetParty("Bowmaster","Hero","Bishop")` → 판 시작 | `1  보우마스터`, `화살 0/10`, client `JobCombat.JobId == "Bowmaster"` |
| T5 롤스터 미로드 안전장치 | (정적 검사) `LoadRunParty`의 nil 분기 | 기존 `LoadFormation` 경로 + `run party roster not ready` 경고 |
| T6 디버그 함수 | `DebugStartFloor(5)` → `DebugGod(true)` → `DebugClearWave()` → `DebugFinishFloor()` | `run start stage=5`, 웨이브 정리 후 다음 웨이브, 마지막에 `stage clear`·결과 화면(`RunState RESULT`) |
| T7 기준 측정 R0 | 20층 `DebugStartFloor(20)` → `DebugGod(true)` → 전투 시작 후 `DebugDpsProbe(30)` | `[VRF] probe floor=20 ... dps=.. normalHp=..` → R0 기록 |

**4-B**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T8 레벨업 없음 | 1층 시작 → 전투 스테이지 3개 진행(`DebugClearWave` 반복) | `[VRF] level up` 0건, `mgr.Level` = 메인 캐릭터 레벨 고정, 스테이지마다 `party break choices`(돌파 1장 선택) |
| T9 구성원별 스탯 | `DebugSetChar("Hero",1,1,1)`, `("Bishop",1,7,2)`, `("NightLord",1,30,5)`, 편성 Hero·Bishop·NightLord → 판 시작 → `DebugDumpParty()` | 기본 공격 = 28 / 32 / 85 에 각각 `× (1 + AccountAtk/100) + 무기`(같은 계정 배율·같은 무기값), 최대 체력 120 / 158 / 556 기준으로 서로 다름. `PlayerMaxHp ≠ CompanionMaxHp ≠ Companion2MaxHp` |
| T10 레벨 차이 | 같은 3직업 Lv1 1차 판 → Lv30 5차 판 각각 `DebugDumpParty` | 히어로 공격 기본값 비 = (77+6.4)/(28+6.4) ≈ 2.4 (계정 % 동일) |
| T11 스테이지 클리어 회복 | 전투 중 server에서 `mgr.PlayerHp = 1` → `DebugClearWave()`로 스테이지 끝 | 스테이지 끝 후 `PlayerHp ≈ 1 + 0.3 × PlayerMaxHp` |
| T12 선행 학습 → 시작 돌파 | `mgr:DebugSetBonus("StartLevel", 2)` → `DebugStartFloor(1)` | 첫 전투 전 `[VRF] start breaks n=2`, 돌파 카드 2번 |
| T13 추가 돌파 확률 | `DebugSetBonus("ExtraPick", 100)` → 전투 스테이지 1개 클리어 | 그 스테이지 후 돌파 선택 2번(`left=1` 로그) |
| T14 거래 문구·동작 | 정적 검사 `DealTable` 3줄 + `HasBreakLeft` 사용 | 문구 `돌파 선택 n회…`, 거래 `SkillPick`이 `HasSkillToLearn`을 부르지 않음 |

**4-C**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T15 차수별 장착 (1) | 편성 Hero·Bishop·NightLord를 차수 t=1~5로(`DebugSetChar(job,1,StartLevel(t),t)`) 바꿔 가며 `DebugStartFloor(1)` 5번 | 매번 `[VRF] slots run job=.. tier=t equip=..`가 5절 표의 그 차수 열과 같음 (예: t=3 히어로 `H02,H11,H03`, t=5 비숍 `B06,B09,B15,B10,B13`) |
| T16 차수별 장착 (2) | 편성 Bowmaster·Viper·Hero로 같은 방법 | 보우마스터·바이퍼 5단계 모두 표와 같음 |
| T17 HUD 칸 | 5차 편성 판 → client `P1~P3/S1~S5/Icon` | 다섯 칸 모두 아이콘 켜짐, 같은 구성원 안에 같은 RUID 없음(N14·N15 다름), 1차 편성이면 S1만 |
| T18 HUD 이름·레벨 | 메인 바이퍼 Lv20 4차 | `P1/Job` = `1  바이퍼  Lv.20`, 2·3번도 자기 차수 이름·레벨 |
| T19 로비 장착 무시 | server에서 `eng.SkillEquip`의 히어로 1칸을 다른 스킬로 바꿈(데이터만, 저장 안 함) → 판 시작 | 장착이 칸 표대로(변화 없음), `run data missing` 경고 0. `Tower/` 검색에서 `SkillRunData`는 안전장치 3곳뿐 |
| T20 쿨·배율 덮어쓰기 | 4차 히어로 판, 무적, 40초 전투 | H08 시전 간격 약 5초, `[VRF] slots run ... pct=1.15,...`, 발할라는 40초에 1번 |

**4-D**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T21 하이퍼 해금 | 3차 판 / 4차 판 각각 60초 | 3차: `hyper cast` 0건. 4차: 구성원마다 `[VRF] hyper cast` 시간 간격 ≥ 40초 |
| T22 헤븐즈 도어 마나 0 | 4차 비숍 판, 시전 직전·직후 `Mana` | B10 시전에 마나가 줄지 않음 |
| T23 궁극기 게이지 | 5차 판, 무적, 60초 | `UltGauge`가 기본 공격·스킬마다 오르고 100에서 `[VRF] ult cast job=.. id=H12/B13/N15` 후 0. 구성원당 60초에 1~3회 |
| T24 게이지 즉시 시전 | `mgr:DebugFillUlt()` | 다음 시전 차례에 세 구성원 모두 `ult cast`. 4차 이하 판에서는 아무 일 없음(`ultId` 없음) |
| T25 HUD 게이지 | 5차 판 client에서 `P1/S5/Cd` 높이와 `P1/S5/CdText` | 게이지 g일 때 덮개 높이 ≈ 72×(1−g/100), 글자 `g%`, 가득이면 빈 글자. 바이퍼 V15가 에너지 부족에도 시전됨 |

**4-E**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T26 패시브 레벨 | 차수 1~5 판마다 `DebugDumpParty` | `passiveLv` = 1/3/5/7/9 (히어로 ComboLv·비숍 HolyLv·직업 PassiveLv) |
| T27 확률 발동 흡수 | 3차 판 | `hidden=H15`(히어로)·`B12`·`N13`/`A13`/`V13`, HUD 칸 수는 칸 스킬 수 그대로(3칸), 60초 안에 `[VRF] passive proc` 1건 이상 |
| T28 위기 자동 흡수 | 4차 나이트로드 판, 무적 끄고 server에서 그 구성원 체력을 최대의 30%로 | `[VRF] passive proc job=NightLord id=N09`, 은신 표시. 비숍 4차는 파티원 체력 50% 아래에서 B04 |
| T29 5차 강화 | 5차 판 server에서 각 컴포넌트 함수 호출 | 히어로 `ComboCap() == 10`, 비숍 `HolyMax() == 10`, 나이트로드 `CalcResMax() == 8`, 보우마스터 15, 바이퍼 타격당 에너지 8 |
| T30 2차 이하 | 1·2차 판 | `hidden` 비어 있음 |

**4-F**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T31 칸 키 저장 | 2차 히어로 판에서 `mgr:DebugGiveBreaks("H02_B1")` → `mgr:HeroBreak("H02","H02_B1")` | true. 내부 저장 키 `Hero.Main`·번호 1 (`DebugDumpParty` 또는 로그) |
| T32 진화 흉내 | 이어서 `mgr:DebugSwapSlot("Hero","Main","H08")` → `HeroBreak("H08","H08_B1")`, `RemainingBreaks(H08)` | true, 남은 돌파 목록에 `H08_B1` 없음. 히어로식 최대 3개가 칸 기준으로 셈 |
| T33 카드·마법의 서 | 5차 판에서 돌파 카드 2번·마법의 서 1번 | 카드는 칸 스킬(+숨은 패시브 스킬) 돌파만, `BookInfo`의 얻은 수가 카드로 얻은 수와 같음 |

**4-G**

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T34 밸런스 측정 | 7-4의 P1~P4 | `0.7 R0 ≤ R ≤ 1.5 R0`, 보고서에 표 |
| T35 생존 | P1·P4 무적 끄고 1스테이지 끝까지 | 전멸 없음 |
| T36 결과 화면 별 | 3단계 보완 (f) 확인(별 기록 비운 20층) | `새로 달성`·`신규 별 +3`, 다시 깨면 `기존 달성` |
| T37 1층 통짜 회귀 | 디버그 없이 1층 처음~보스·결과 | 돌파 카드·정비소·마법의 서·보스·결과 팝업 정상, `LEA-` 0 |
| T38 금지 파일 | `git diff --stat` | `PlayerEngrave.mlua`·`RelicUILogic.mlua`·`ui/RelicUI`·`UI-Design-System.md`·`HeroSkillUILogic.mlua`에 4단계 변경 없음 |

---

## 9. 구현 주의

- **메서드 설명 주석은 본문 첫 줄.** `member`는 예약어. 선언 안 한 필드 대입은 조용히 실패 → 내부 상태는 `self._T`. 문자열 치환 `string.gsub` 금지(LEA-1103).
- `@Sync`를 새로 붙이거나 속성을 추가한 뒤 반드시 메이커 새로고침. 클라이언트 값은 늦게 오므로 HUD 확인은 1~2초 뒤 다시 읽는다.
- 전투 3종은 같은 수정을 세 번 한다. 한 파일만 고치고 끝내지 않도록 단위마다 세 파일 `BeginRun` 로그(`slots run`)를 모두 확인한다.
- 동료 엔티티(`towercompanion` 등)에도 같은 전투 컴포넌트가 있다. 메인 엔티티는 세 컴포넌트를 모두 가지므로 **메인 직업이 아닌 두 컴포넌트는 `Active=false`** 여야 HUD가 맞는 컴포넌트를 읽는다(4-A 3번).
- `PlayerRoster`는 탑에서 **저장하지 않는다**(`RunPartyData`는 읽기 전용). `DebugSetChar`만 예외(테스트).
- 4-C 이후 로비 스킬 화면에서 장착·레벨을 바꿔도 판에 반영되지 않는 것이 정상이다(6단계에서 화면 제거).
- 하이퍼·궁극기·패시브 스킬도 `SkillLv[id] = 1`로 둬야 `Learned()`가 true다.
- 밸런스 수치는 데이터로만 고친다(코드 상수 하드코딩 금지).

---

## 10. 이후 단계로 넘기는 것

- 6단계: `DoLevelUp`·`HasSkillToLearn`·CHOICE(판 안 스킬 카드)·`RunConfig` 판 레벨 키 삭제, 각인 `HeadStart`/`Collector`·은총 `Stone`·계정 75레벨 `ExtraPick` 설명 문구 교체(6-C), 로비 스킬 화면·`SkillRunData*` 제거(6-B), 테스트 함수 정리, 각성에서 칸 스킬 각성(칸 키 재사용).
- 6-D 밸런스: 60층 이후(장비·각성), 하이퍼·궁극기 `PctMul`, `StageClearHealPct` 재검토.
- 빠진 리소스: V01 서머솔트 킥·N15 다크로드의 비전서 전용 아이콘, 궁극기 게이지 전용 HUD 막대(지금은 5번째 칸 덮개 재사용).

## 4-B 보완 (4단계 1차 테스트 뒤 메인 세션 결정)

1차 테스트 보고서: `Phase-4A-Test-Report.md`. 4-C 작업 전에 먼저 처리한다.

1. **보스(층 클리어) 처리 두 번 실행 — 지금 고친다.** `TowerRunManager`에서 재진입을 막는다.
   - OnUpdate CLEAR 분기의 스테이지 끝 블록에 들어가자마자 재진입 방지 플래그(예: `_T.stageEnding = true`, 판 시작·다음 스테이지 시작 때 해제)를 세우고, 플래그가 서 있으면 블록 전체를 건너뛴다.
   - `StageClear()`의 중복 방지(`RunState="RESULT"` 또는 플래그)를 `SettleAccount` 호출보다 앞으로 옮긴다.
   - `Defeat()`도 같은 방식(첫 줄에서 플래그 확인·설정 후 진행).
   - PlayerEngrave(대기 없는 저장)는 건드리지 않는다.
2. **웨이브 막대**: `MainPanel/BottomBar`는 전부터 꺼져 있었다. 다시 켜지 않는다. 처치마다 `Exp+1`은 해가 없으니 그대로 둔다.
3. **기준값**: 4-B 코드 기준 R=0.675를 4-G 비교 기준으로 쓴다.
4. **T11 기대식**: 스테이지 클리어 회복 = 최대 체력 × (30% × HealRecv 배율) + 유물 '스테이지 클리어 회복' 분. 흡혈 등 직전 회복은 제외하고 본다.
5. **HUD 구성원 레벨**: 지금 고친다. `P1~P3/Job`이 `mgr.Level` 대신 각 구성원의 캐릭터 레벨(판 시작 때 받은 RunDataFor 값)을 보여 준다. 클라이언트가 읽을 수 있게 구성원별 레벨을 Sync 값(예: `@Sync PartyLevels` 문자열 "lv1,lv2,lv3")으로 보낸다. 스킬 칸 아래 의미 없는 `Lv.` 표시는 4-C에서 칸 스킬 표시로 바꿀 때 지운다.
6. **여신 제안 문구**: 지금 고친다. `GoddessEffectText`의 "스킬 카드 선택 +n회"를 바뀐 뜻("돌파 선택 +1 확률 n%" 등, 4-B의 ExtraPick 의미)에 맞게.
7. **디버그 함수 실행 규칙**: `Debug*` 함수는 내부에서 `_TimerService:SetTimerOnce`로 감싸 다음 틱에 돌게 바꾸고, 8-1 테스트 방법에 "메이커 스크립트에서 직접 부르면 프레임이 끊겨 상태를 덮어쓸 수 있으니 타이머 안에서 실행"을 적는다.

## 4-D 보완 (4단계 2차 테스트 뒤 메인 세션 결정)

2차 테스트 보고서: `Phase-4C-Test-Report.md`. 4-E 작업 전에 먼저 처리한다.

1. **궁극기 "약 30초"는 전투 시간 기준**이다. 스테이지 전환·몬스터 걸어오는 시간은 세지 않는다.
2. **근접 구성원 문제(B1)는 칸 전투 규칙을 바꾸지 않고 푼다.** 사용자가 확정한 "몬스터는 자기 사거리 칸에 멈춘다", `PullIfUnreachable`(아무도 닿지 않을 때만 당김), 사거리 표(`JobRanges`)는 그대로 둔다(A1·A2 채택 안 함).
   - **C안 채택**: 궁극기, 쿨타임형 자기 버프, 하이퍼는 `CanReach` 검사 전에 시전한다. 대상이 필요한 궁극기·하이퍼는 사거리와 무관하게 앞쪽 살아 있는 몬스터(없으면 아무 몬스터)를 대상으로 쓴다.
   - **전투 중 시간 충전 추가**: `RunConfig`에 `UltGaugePerSec`(전투 상태일 때만, 기본 1.5)를 넣고, 구성원마다 매 틱 `AddUlt(delta × UltGaugePerSec)`. 닿지 않는 근접도 전투 시간 약 50~60초면 궁극기가 나가고, 행동하는 구성원은 약 25~30초가 되도록 4-G에서 값 확인.
   - 근접이 닿지 않을 때 할 일이 없는 것은 원래 규칙의 결과로 본다. 4-G에서 근접 패시브(예: 히어로 콤보 어택 단계 효과에 "닿지 않는 동안 받는 피해 감소" 등)로 보완할지 검토만 한다.
3. **하이퍼 쿨에 계정 쿨 감소 적용**은 맞다(지금 38.4초 유지).
4. **B3**: 디버그로 시작한 판(`DebugStartFloor` 사용)은 패배·클리어 때 최고 층(`SaveBestFloor`)과 첫 클리어 다이아·별 기록을 저장하지 않는다. `_T.debugRun = true` 플래그로 막는다.
5. **U1**: 궁극기 % 표시는 `math.floor(g + 0.0001)` 등으로 소수 오차를 없앤다.
6. **U3**: 여신 제안 카드 줄 정렬만 고친다. 패널 색 테마 전환은 6단계로 넘긴다.

## 4-G 보완 / 밸런스 결정 (Phase-4E-Test-Report 반영, 메인 세션 결정)

칸 전투 규칙(사용자 확정: 몬스터는 자기 사거리 칸에서 멈춤, PullIfUnreachable, JobRanges)은 바꾸지 않는다.

1. **디버그 판 저장 전부 막기**: `_T.debugRun` 판은 최고 층·별·클리어 층·다이아에 더해 **계정 경험치와 판 보상(깃털·스킬북·주문서·열쇠 등)도 저장하지 않는다.** 결과 팝업에는 "테스트 판: 저장 안 됨" 문구를 띄운다.
2. **캐릭터 능력치 상향 (기대표 유지)**: `CharJobTable.csv` 공격력 두 열 ×1.6, 체력 두 열 ×1.5.

   | 직업 | BaseAtk | AtkPerLv | BaseHp | HpPerLv |
   |---|---|---|---|---|
   | Hero | 45 | 1.28 | 180 | 18 |
   | Bishop | 40 | 1.15 | 135 | 13.5 |
   | NightLord | 50 | 1.41 | 143 | 14.25 |
   | Bowmaster | 50 | 1.34 | 143 | 14.25 |
   | Viper | 45 | 1.28 | 165 | 16.5 |

   그리고 `CharTierTable` `StatPct`를 2차 10→20, 4차 35→45로 올린다(다른 차수는 그대로). 재측정 후 기대표 대비 0.8~1.25배면 합격, 벗어나면 테스트 에이전트가 다음 조정안을 낸다.
3. **궁극기 시간 충전 범위 넓힘**: `TowerRunManager.TickUltTime`에서 `fc.Moving`(몬스터가 칸에 멈춰 있음) 조건을 빼고 **전투 상태(웨이브 진행 중, 살아 있는 몬스터 있음) 전체 시간**에 충전한다. `RunConfig` `UltGaugePerSec` 1.5 → 1.1. 목표: 원거리 25~32초, 닿지 않는 근접 45~60초.
4. **발할라 재시전**: `HeroCombat.PickCooldownCast`에서 충전이 남아 있어도 쿨타임이 끝나면 다시 시전하고, 시전 시 충전을 최대치로 새로 채운다. 다른 하이퍼와 같이 약 쿨타임 간격으로 쓴다.
5. **마법의 서 줄 늘림**: `MagicBookUILogic.Refresh`와 마법의 서 UI를 구성원당 최대 **7줄**(칸 5 + 숨은 패시브 스킬 2)로 늘린다. 화면에 안 들어가면 줄 높이·글자 크기를 줄여 맞춘다. 테마 전환은 6단계에서 하므로 이번엔 배치만.
6. **근접 피해 비중(약 7%)**: 이번 단계에서는 그대로 둔다. 닿지 않는 근접은 하이퍼·버프·궁극기로 기여하는 구조로 확정. 5단계 장비 밸런스 때 다시 본다.
7. **테스트 중 생긴 UI 이상**(마법의 서 강제 오픈 시 몬스터 이름 겹침, 궁극기 % 미갱신)은 정상 흐름에서 안 생기므로 고치지 않는다.

### 4-G 구현 후 추가 결정
- 발할라 판 안 쿨타임 약 38초는 의도한 값이다(4-D 보완: 하이퍼 쿨타임에 계정 쿨감 반영).
- 디버그 판 결과 화면의 별 표시("신규 별 +3", "누적 별 -3")와 빈 계정 경험치 막대는 디버그 전용이라 다음 구현 묶음(5단계 첫 구현)에서 "테스트 판" 표시로 같이 정리한다.
- 측정 시 탑 입장 직후 자동으로 시작되는 일반 판 보상이 저장될 수 있으므로, READY 상태에서 바로 DebugStartFloor를 부르고 끝에 원문 되쓰기 후 재확인한다.

### 4-G 재측정 후 결정 (Phase-4G-Test-Report 반영)
- 판정 기준은 **M1.0 열(5/14/37/60층)**, 합격은 **층 진행 우선**, R값은 참고만 한다(R0는 예전 구조 기준).
- `CharTierTable` StatPct: 1차 0→10, 2차 20→35, 3차 20→35 적용. 4·5차는 그대로.
- 사용자 요청으로 측정을 중단했다. 남은 확인(다음 테스트 묶음에서): Lv30 층 진행, Lv10 3판 이상, 궁극기 주기, 발할라 재시전 간격, 일반 판 1층 회귀, 마법의 서 7줄 화면, 디버그 판 결과 팝업 문구 화면.
