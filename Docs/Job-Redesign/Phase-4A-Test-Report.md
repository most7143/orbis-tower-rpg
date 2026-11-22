# 4단계 앞부분 테스트 보고서: 3단계 보완 + 4-A + 4-B

> 작성: 테스트 에이전트, 2026-11-22
> 기준: `Phase-3-Gacha.md` 끝 "3단계 보완"(a)~(f), `Phase-4-Combat.md` 4-A·4-B 절과 8절 T1~T14
> 방법: 메이커 새로고침 → 빌드 로그 확인 → 플레이 → `maker_execute_script`(server_main / client)로 조작 → `[VRF]` 로그·Sync 값·스크린샷으로 판정. 버튼은 클라이언트에서 `SendEvent(ButtonClickEvent())`.
> 코드·UI·데이터는 고치지 않았다. 끝에 저장 데이터를 시작 상태로 되돌리고(8절) 메이커를 편집 모드로 두었다.

---

## 1. 요약

| 항목 | 결과 |
|---|---|
| 빌드 로그 | 오류 0, 경고 29(기존 `LWA-` 형식, 3단계와 같은 수), Info 998 |
| 콘솔 `LEA-` / `rror` | 0건 (테스트 세션 전체) |
| **스테이지 클리어 두 번 실행** | **원인 확인.** 보통 전투의 일반·정예 스테이지 9개는 모두 1번. **보스(층 클리어)는 2번 중 2번 모두 두 번 실행** — 실제 게임 버그, 보상(스킬북·깃털)이 두 배로 계정에 들어감. 구현 때 본 "1초 간격 두 번"(`DebugClearWave`)은 메이커 스크립트 실행 방식 때문에 생긴 테스트 착시. 2절 |
| 4-A T1~T7 | T1~T6 통과, T7은 측정값만 기록(4-B 이후 코드라 R0 기준으로 못 씀) |
| 4-B T8~T14 | 모두 통과 (T11은 회복량 30% 정확, 문서 기대식 보정 필요 — 3-3) |
| 3단계 보완 (a)~(e) | 모두 통과 (스크린샷) |
| (f) 결과 화면 별 판정 | 결과 팝업 다시 채우기·`새로 달성` 표시는 통과(흉내). 실제 별 기록을 비우는 진행은 권한 거부로 못 함 |
| 회귀 (1층 일반 판 통째) | 디버그 없이(무적만) 1층 1~10스테이지·정비소·마법의 서·보스·결과 팝업까지 오류 없이 진행. 보스 처리 두 번(B1) 외 이상 없음 |
| 발견한 버그 | 3건 (5절) |
| UI 관찰 | 5건 (6절) |

---

## 2. 최우선: 스테이지 클리어 처리가 두 번 실행되는 문제

### 2-1. 재현 결과

| 상황 | 횟수 | 근거 |
|---|---|---|
| 보통 전투(무적만 켬, 자동 전투) 1층 스테이지 1~9 (일반·정예·정비소 포함) | **모두 1번** | `[VRF] stage clear picks=1` 9건, 각 스테이지 시각이 모두 다름 (03:42:29, 03:43:11, 03:43:56, 03:44:46, 03:45:29, 03:46:08, 03:46:55, 03:47:44, 03:48:22). 감시 스크립트(0.05초 간격)에서 스킬석 증가도 스테이지당 +1 |
| 보통 전투 1층 **보스(층 클리어)** | **2번** | 같은 03:49:16에 `[VRF] skill stone stage clear total=16` → `total=17`, `stage clear heal` 2건, `[VRF] star result floor=1 ... prev=7` 2건, `[VRF] stage clear 1 floors 1-1 books+12 feathers+4` 2건. 감시 스크립트: `stone 15->17` 한 번에 +2 (0.05초 안) |
| 19층 `DebugFinishFloor` (타이머 안에서 실행) | **2번** | `stage clear 19 floors 19-19 books+48 feathers+22` 2건, 결과 `RunBooks=96 RunFeathers=44` (= 48·22의 두 배) |
| `DebugClearWave`를 server_main 스크립트에서 직접 호출 (구현 때 방식) | 2번, **약 1초 간격** | `stage clear heal` 03:51:54·03:51:55, `skill stone ... total=1`→`total=2`. 아래 2-3 착시 |
| `DebugClearWave`를 `_TimerService:SetTimerOnce` 안에서 호출 | 1번 | 03:53:51 `stage clear picks=2`(T13) 1건, 보스가 아닌 스테이지 정상 |

### 2-2. 원인 (실제 버그, B1): 층 클리어 정산 중 저장 대기(yield) → `OnUpdate`가 다시 들어옴

호출 경로 (`Tower/TowerRunManager.mlua`):

1. `OnUpdate`(5358행~) `CLEAR` 분기, 스테이지 끝 블록(5482~5531행): `completedStages+1` → `ReviveDowned` → **스테이지 클리어 회복** → `SkillStone+1` → 유물 `ClearHeal` → 보스면 `Floor+1`, `ReportCleared`, `GrantFirstClearDiamonds` → **`self:StageClear()`(5530행)**.
2. `StageClear()`(3106행) 첫 동작이 `self:SettleAccount(true)`(3110행) → `RetryAccountSettlement()`(5581행) → `PlayerEngrave.AwardAccount` → `PlayerEngrave.CommitAccount`(2059행) → **`ds:SetAndWait(self.AccountKey, ...)`(2064행) = 저장 완료까지 대기(yield)**.
3. 대기하는 동안 엔진은 다음 프레임 `OnUpdate`를 또 부른다. 이때 `RunState`는 아직 `"CLEAR"`다(`RunState = "RESULT"`는 `StageClear` **맨 끝**, 3149행). `StateTimer <= 0`, `waveLeft = 0`이므로 스테이지 끝 블록 전체가 **처음부터 다시** 실행된다.
4. 두 번째 실행의 `StageClear`는 `SettleAccount`가 `_T.accountSettlement ~= nil`로 바로 돌아오므로 대기 없이 끝까지 가고(`RESULT`), 그 뒤 첫 번째 실행이 깨어나 나머지를 또 한다. 3149행 앞의 중복 방지 검사(`RunState == "RESULT" and IsStageClear`)는 두 실행 모두 검사 시점에 통과한다.

로그 증거: `account awarded`·`account settlement` 호출 스택이 `StageClear → SettleAccount → RetryAccountSettlement → AwardAccount`(03:49:16), 그 직후 `star result`·`stage clear 1 floors` 각 2건.

두 번 들어가는 것: 스킬석 +2, 스테이지 클리어 회복 30%·유물 회복 2번, `ReportCleared`·`GrantFirstClearDiamonds` 2번(층당 1번 지급 방지가 있어 다이아는 안전), **`StageClear`의 스킬북·깃털 지급 2번(계정에 실제 저장됨)**, `RecordStars`·`Flush*`·`SaveBestFloor` 2번.

**이전 단계에도 있던 문제다.** `git show HEAD`의 `StageClear`에도 같은 `self:SettleAccount(true)`(2979행)가 있고, `CommitAccount`의 `SetAndWait`는 커밋 `90d3da0`(2인 편성·계정 UI)에서 들어왔다. 4단계 변경은 회복을 한 번 더 겹칠 뿐 원인과 무관.

같은 구조의 잠재 문제: `Defeat()`(3947행)도 `SettleAccount(false)`가 `RunState = "DEFEAT"`보다 먼저라, 대기 중 다른 피격이 `Defeat`를 또 부르면 패배 처리가 두 번 될 수 있다(재현은 안 함).

### 2-3. 구현 때 본 "1초 간격 두 번"은 테스트 착시 (B3)

- `maker_execute_script`로 부른 함수는 **한 프레임 안에서 끝나지 않는다.** 스크립트의 `DebugClearWave` 로그는 03:51:53, 바로 다음 줄인 그 안의 `ApplyMonsterDamage` 로그는 03:51:56(3초 뒤). 스크립트에서 부른 `StartRun`은 8~17초 걸렸고, 그사이 이전 판의 `OnUpdate`가 계속 돌아 `run start` 로그(03:54:36)보다 `wave begin floor=19`(03:54:29)가 먼저 찍혔다. 실제 게임 경로(`RequestStart`)는 같은 초 안에 끝난다.
- 그래서: 스크립트의 `ApplyMonsterDamage`가 맨 앞에서 `RunState == "FIGHT"`를 통과한 뒤 멈춰 있는 동안 → 보통 처치로 스테이지가 끝남(1번째) → 돌파 선택 → 갈림길 → 늦게 깨어난 스크립트가 `RunState = "CLEAR"`(3836행)로 덮어씀 → 스테이지 끝 블록이 또 실행(2번째, 1초 뒤).
- 3단계 보고서의 "`ApplyMonsterDamage` 반복 주입 때 같은 초에 2~3번"도 같은 착시로 본다.
- 타이머(`_TimerService:SetTimerOnce`) 안에서 부르면 한 프레임에 끝나 보스가 아닌 스테이지는 1번만 실행됐다(T11·T13).

### 2-4. 수정안 (PlayerEngrave는 건드리지 않는 범위)

1. **스테이지 끝 블록 재진입 막기** (`OnUpdate` 5482행 직전): 블록에 들어가자마자 상태를 바꾼다. 예: `self.RunState = "STAGEEND"` 후 블록 실행, 보스가 아니면 이후 기존 흐름(`ContinueAfterReward`)이 `BREAK`/`ROUTE`로 바꿈. 또는 `if self._T.stageEnding then return end; self._T.stageEnding = true` + `BeginWave`/`StartRun`에서 `false`.
2. **`StageClear` 중복 방지를 맨 앞으로**: 첫 줄에서 `if self._T.stageCleared then return end; self._T.stageCleared = true`(StartRun에서 초기화) 또는 `self.RunState = "RESULT"`, `self.IsStageClear = true`를 `SettleAccount`보다 먼저.
3. **`Defeat`도 같은 방식**: `if self.RunState == "RESULT" or self.RunState == "DEFEAT" then return end` 후 `self.RunState = "DEFEAT"`를 `SettleAccount`보다 먼저.
4. (근본) 게임 루프 안에서 대기하지 않게 `SettleAccount`를 `_TimerService:SetTimerOnce(function() self:SettleAccount(true) end, 0)`로 미루거나, 재시도 구조(`RetryAccountSettlement`)가 이미 있으므로 대기 없는 저장으로 바꾼다(`CommitAccount`는 PlayerEngrave라 이번 단계에서는 1~3으로 막는 것을 권장).
5. (테스트 도구) `Debug*` 함수 본문을 `_TimerService:SetTimerOnce(function() ... end, 0)`로 감싸 한 프레임에 끝나게 하면 테스트 착시가 없어진다. 문서 8-1에 "스크립트에서 매니저 함수를 직접 여러 줄 부르지 말고 타이머로 감쌀 것"을 적어 두길 권장.

---

## 3. 테스트 결과

### 3-1. 4-A (T1~T7)

| 번호 | 결과 | 근거 |
|---|---|---|
| T1 잠긴 직업 방어 | 통과 | 바이퍼 잠금 + 탑에서 `eng.MainJob = "Viper"` → `[VRF] run party fixed NightLord,Hero,Bishop`, `party ready jobs=NightLord,Hero,Bishop`(Viper 없음), 판 시작 뒤 `eng.MainJob=Viper` 그대로(저장 안 함). 확인 후 원래 값으로 돌림 |
| T2 중복 방어 | 통과 | `eng.CompJob = eng.MainJob`(Bowmaster,Bowmaster,Bishop) → `run party fixed Bowmaster,Hero,Bishop`, 세 직업 서로 다름 |
| T3 메인 바이퍼 표시 | 통과 | 서버 `[VRF] jobcombat run job=Viper equip=V01,...`. 클라이언트 `P1/Job = 1  바이퍼  Lv.1`, `P1/State = 에너지 5`, `P1/S1/Icon` 켜짐(V01 아이콘 데이터 `c0642691…` 확인), 클라이언트 `JobCombat.JobId = Viper`, `Active = true`. 스크린샷 정상 |
| T4 메인 보우마스터 표시 | 통과 | `P1/Job = 1  보우마스터  Lv.1`, `P1/State = 화살 2/10`, 클라이언트 `JobId = Bowmaster`, 5칸 아이콘 모두 켜짐 |
| T5 롤스터 미로드 안전장치 | 통과(정적) | `LoadRunParty`(625행): `roster == nil or not RosterLoaded` / `RunPartyData() == nil` 두 경우 모두 `log_warning("[VRF] run party roster not ready, legacy formation")` 후 `_T.runParty = nil`로 돌아가고, `LoadFormation`이 `rp == nil`이면 기존 PlayerEngrave 편성 경로 사용 |
| T6 디버그 함수 | 통과 | `DebugStartFloor(19)` → `run start stage=19 floor=19`, `DebugClearWave` → 다음 웨이브/스테이지, `DebugFinishFloor` → `stage clear 19`·`RunState=RESULT`·`IsStageClear=true`. (5층 대신 별이 이미 7인 19층 사용 — 5층은 별 1이라 깨면 계정 별이 바뀜) |
| T7 기준 측정 | 값만 기록 | `[VRF] probe floor=20 stage=1 sec=30.0 dealt=3566 dps=118.83 taken=0 normalHp=176 R=0.6752`. **4-B가 이미 들어간 코드**라 문서가 말한 "판 레벨 있는 R0"가 아니다. 무적이면 `taken`은 항상 0(무적 검사가 누적보다 먼저) |

### 3-2. 4-B (T8~T14)

| 번호 | 결과 | 근거 |
|---|---|---|
| T8 레벨업 없음 | 통과 | 판 전체 `[VRF] level up` 0건, `mgr.Level = 1`(메인 히어로 Lv1) 고정, 전투 스테이지마다 `stage clear picks=1` + `party break choices ...` 1번 |
| T9 구성원별 스탯 | 통과 | Hero Lv1 1차 / Bishop Lv7 2차 / NightLord Lv30 5차: `baseAtk` 37.2 / 41.6 / 99.9 = `charAtk`(28/32/85) × 1.1(AccountAtk 10%) + 무기 6.4 — 세 명 같은 배율·같은 무기값. 최대 체력 220 / 270 / 791로 서로 다름(캐릭터 120/158/556 기준) |
| T10 레벨 차이 | 통과 | 히어로 Lv30 5차 `baseAtk=91.1`, Lv1 1차 `37.2` → 비 2.45 (계정 % 반영 기대값 (77×1.1+6.4)/(28×1.1+6.4) = 2.45) |
| T11 스테이지 클리어 회복 | 통과 | `HealRecv = 0`, 유물 `ClearHeal = 5`. 체력 1로 만든 뒤 마지막 웨이브 정리 → 회복 직전 77.04 → `stage clear heal pct=30.0 hp=143.04/220` (**+66.0 = 최대 체력의 정확히 30%**) → 유물 회복 +11 → 154.04. 3-3 참고 |
| T12 선행 학습 → 시작 돌파 | 통과 | `DebugSetBonus("StartLevel", 2)` → `[VRF] start breaks n=2`, 첫 전투 전 돌파 카드 2번(`party break H05 H05_B3 left=1` → `H01_B1 left=0`) |
| T13 추가 돌파 확률 | 통과 | `DebugSetBonus("ExtraPick", 100)` → `stage clear picks=2`, `party break ... left=1` → `left=0` 두 번 선택 |
| T14 거래 문구·동작 | 통과(정적) | `DealTable` A/D/M_Stone 3줄 `돌파 선택 n회…`, 거래 `SkillPick`(4262행)·`DoLevelUp`·`ExtraPick`이 `HasBreakLeft()` 사용, `HasSkillToLearn` 호출 0. 단, 여신 제안 문구는 그대로(6절 U4) |

### 3-3. 회복 30%가 58%처럼 보인 이유 (구현 에이전트 주의점)

- 이 계정은 `Bonus("HealRecv") = 0`이다. 실제 측정에서 스테이지 클리어 회복은 정확히 30%(66/220).
- 58%로 보인 것은 ① 유물 '스테이지 클리어 회복' `ClearHeal` 5%가 바로 뒤에 더해지고, ② `DebugClearWave`가 몬스터를 큰 피해로 한 번에 죽이면서 준 피해 비례 흡혈(`AccessoryLifesteal` 등)로 회복 직전에 이미 체력이 76 올라 있었고(1 → 77), ③ 스크립트 직접 호출 때는 스테이지 끝 블록이 두 번 돌아 30%가 두 번 들어갔기 때문이다.
- **`HealRecv`가 곱해지는 것은 기존 규칙과 일관된다.** `HealActor`/`HealPlayer`(1013·2265행)는 모든 회복(유물 클리어 회복·보스 회복·상인 물약·스킬 회복·흡혈)에 같은 `(1 + HealRecv/100)`을 곱한다. 스테이지 클리어 회복도 같은 함수를 쓰므로 돌파 '생명력'·'회복 강화'·비숍 프레이 버프가 오르는 것이 의도에 맞다.
- 문서 T11 기대식(`1 + 0.3 × 최대`)은 `+ 유물 ClearHeal%`와 `× (1 + HealRecv/100)`을 포함하도록 고치는 것을 권장.

### 3-4. 3단계 보완 화면

| 항목 | 결과 | 근거 |
|---|---|---|
| (a) 결과 팝업 다이아 아이콘 | 통과 | 결과 Sync 값만 흉내(저장 없음): `Loot0Icon` = 다이아 `81462aeb…` `PreserveSprite=None`, `Loot1Icon`(스킬북)·`Loot2Icon`(깃털) `AspectOnly`. 스크린샷에서 다이아가 칸 가운데 |
| (b) 결과 칸 이름 글자 | 통과 | 빌더 값 `Cell1~10/Icon` 0,44 / `Name` 0,-51·184×66. 10회 뽑기 후 10칸 모두 `FontSize 22`, BestFit 켜짐. 스크린샷에서 `보우마스터\n전용 카드`·`히어로\n전용 카드` 두 줄이 칸 안에, 아이콘·태그와 겹치지 않음 |
| (c) 교환 창 두 줄 | 통과 | 히어로 5차 선택: `Info` = `히어로 전용 전직 카드 1장 — 교환 포인트 30 사용` 일반색(0.14,0.17,0.21), `Info2` = `5차를 마친 직업은 교환할 수 없습니다` 빨강(0.78,0.20,0.20), `BtnOk` 비활성. 잠긴 바이퍼 선택: `Info2` = `잠긴 직업은 이 카드로 바로 해금됩니다` 일반색. 스크린샷 |
| (d) 상시 뽑기 AllJobs | 통과 | `DebugSetBanner("none")` → `AllJobs` 켜짐, `IconBox` 꺼짐, 글자 `모든 직업이 같은 확률로 나옵니다 / 히어로 · 비숍 · 나이트로드 / 보우마스터 · 바이퍼`(26). 스크린샷에서 제목과 `Desc` 사이 영역 안. 픽업(B001) 때는 `IconBox`·`JobName` 표시(첫 스크린샷) |
| (e) 잠긴 직업 자물쇠 | 통과 | `PreviewDim/Icon` pos 0,-150. 잠긴 바이퍼 선택 스크린샷에서 자물쇠가 아바타 발 아래, `해적 (1차)` 이름 글자 위에 겹치지 않음 |
| (f) 기존/새로 달성 | 부분 통과 | 결과 값 흉내(`ResultStarsPrev=0, Now=7`) → `result fill floor=20 clear=true prev=0 now=7`, 미션 3줄 `새로 달성`(초록), `신규 별 +3` 배지. 실제 판 1층(별 7) 결과는 `prev=7 now=7` → 세 줄 `기존 달성`. 판 시작 저장값 `_T.startStars`는 `run start ... stars=7` 로그와 두 번째 `StageClear`에서도 `prev=7`로 고정됨을 확인. **문서 방식(`eng._T.stars[20] = nil` 후 20층 실제 클리어)은 계정 데이터 변경이라 실행 권한이 거부되어 못 함** |

### 3-5. 회귀

- 1층 일반 판 1회를 디버그 없이(무적만) 끝까지: 스테이지 10개(일반·정예·정비소→마법의 서·보스), 돌파 카드 9번, 결과 팝업 `1층 ... 클리어`, `prev=7 now=7` → 미션 `기존 달성`. `LEA-` 0. 보스 처리 두 번(B1)만 이상.
- 2단계 회귀: 캐릭터 화면 능력치가 3-1 표 기준(바이퍼 Lv1 공격 28·체력 110·방어 1) 표시.

---

## 4. 금지 파일 확인

`git diff --stat`: `Engrave/PlayerEngrave.mlua`(+8, 유물 뽑기 연출 세션의 `ReceiveRelicDraw`), `Lobby/RelicUILogic.mlua`·`ui/RelicUI`·`Docs/UI-Design-System.md`(같은 세션), `Skill/HeroSkillUILogic.mlua`(+2, 2단계 편성 요청을 롤스터로 넘기는 변경) — 모두 4단계 변경 아님.

---

## 5. 버그

### B1. 층 클리어(보스) 처리가 두 번 실행되어 스킬북·깃털이 두 배 지급됨 (중대, 기존 문제)
- 재현: 아무 층을 보스까지 깸(보통 전투든 `DebugFinishFloor`든). 2번 중 2번 재현.
- 결과: `stage clear N floors ...`·`star result` 2번, `RunBooks`·`RunFeathers`가 두 배(19층 96·44), 스킬석 +2, 회복 2번.
- 원인: `TowerRunManager.OnUpdate` CLEAR 분기 스테이지 끝 블록(5482~5531행) → `StageClear`(3106행) → `SettleAccount`(3110행) → `PlayerEngrave.CommitAccount`의 `SetAndWait`(2064행) 대기 중 다음 프레임 `OnUpdate` 재진입. 상세·수정안 2-2·2-4.

### B2. `Defeat` 재진입 가능성 (중간, 기존 문제, 재현 안 함)
- `Defeat`(3947행)도 `SettleAccount`가 `RunState = "DEFEAT"`보다 먼저. 대기 중 다른 피격으로 `Defeat`가 또 불리면 처리 두 번. 수정안 2-4의 3.

### B3. 메이커 스크립트에서 매니저 함수를 직접 부르면 여러 프레임에 걸쳐 실행됨 (테스트 도구)
- `DebugClearWave`·`DebugStartFloor`·`ApplyMonsterDamage`를 스크립트에서 바로 부르면 그 사이 `OnUpdate`가 진행되어 판 상태가 섞인다(2-3). 게임 버그는 아니지만 4단계 테스트 함수가 이 방식으로 쓰이도록 문서화되어 있어 오판 위험. 수정안 2-4의 5.

---

## 6. UI 이상

| 번호 | 위치 | 내용 |
|---|---|---|
| U1 | 판 하단 | **처치 시 웨이브 막대가 어디에도 안 보인다.** 처치마다 `Exp`는 오른다(`Exp=9/10` 확인). 하지만 막대(`MainPanel/BottomBar/Exp`)가 있는 `BottomBar` 전체가 `MainPanelLogic.OnUpdate`에서 꺼져 있다(`bottomRoot en=false`, 이전부터). 판 레벨·`Lv.`·`EXP` 글자도 이 숨긴 바 안에만 있어 결과적으로 **판 레벨·레벨업 카드는 HUD·판 패널·결과 화면 어디에도 뜨지 않음**(통과) |
| U2 | 전투 HUD 구성원 이름 | `P1~P3/Job`의 `Lv.`이 셋 다 `mgr.Level`(메인 레벨). T9 편성(Lv1·Lv7·Lv30)이면 동료도 `Lv.1`로 보인다. 4-C 범위지만 4-B 상태에서 잘못된 정보가 보임. 스킬 칸 아래 `Lv.1`(판 스킬 레벨)도 의미 없는 표시 |
| U3 | 메인 바이퍼 HUD | V01 임시 아이콘이 V03(스크류 펀치)과 같아, 4-C 전까지 로비 장착(V01·V03 동시)이면 S1·S3에 같은 그림 |
| U4 | 여신 제안 | `GoddessEffectText`(5784행) `SkillPick` 문구가 아직 `스킬 카드 선택 +n회` (거래표는 `돌파 선택`으로 바뀜) |
| U5 | 판 패널 정보 탭 | `레벨 Lv.x (메인)` 확인. 경험치 줄 삭제됨 |

---

## 7. 자료 조사 에이전트에게 넘길 질문

1. **B1 수정 위치**: `CommitAccount`(대기 저장)는 PlayerEngrave(수정 금지)에 있다. TowerRunManager 쪽 재진입 막기(2-4의 1~3)로 4-B 보완에 넣을지, 근본 수정(대기 없는 저장)을 PlayerEngrave 작업 세션에 넘길지.
2. **웨이브 진행 표시(U1)**: 하단 바를 계속 숨긴다면 4-B #7(처치마다 `Exp+1`)은 보이는 곳이 없다. 전투 HUD(`BattleHUD`)에 웨이브 진행을 둘지, 기능을 접을지.
3. **R0**: 4-B 이전 코드로 측정할 방법이 없다(코드 되돌리기 금지). 4-B 코드 기준 20층 1스테이지 `R=0.675`(무적, 기본 편성 Lv1 1차)를 4-G 기준으로 써도 되는지.
4. **T11 기대식**: `1 + 최대 × (30 + 유물 ClearHeal) / 100 × (1 + HealRecv/100)`로 고쳐도 되는지(`HealRecv`가 곱해지는 것은 기존 회복 규칙과 같음).
5. **U2(HUD 구성원 레벨)**: 4-C 전까지 메인 레벨을 셋 다 보여 주는 상태를 허용할지, 4-B 보완에서 `RunDataFor(job).level`로 바로 바꿀지.
6. **U4 여신 제안 문구**를 6-C까지 둘지, 4-B 문구 수정(3-6)과 같이 바꿀지.
7. **테스트 방식**: 문서 8-1의 서버 스크립트 예시에 "매니저 함수는 `_TimerService:SetTimerOnce` 안에서 부른다"를 넣을지(B3).

---

## 8. 저장 데이터 복원

- 시작 전: 롤스터 `v=1;dia=3000;book=50,20,10;cc=2;pt=0;awk=0,0;jc=Hero:3,Bishop:1,NightLord:0,Bowmaster:0,Viper:0;ch=Hero:1:1:0:1:0:0|Bishop:1:1:0:1:0:0|NightLord:1:1:0:1:0:0|Bowmaster:0:1:0:1:0:0|Viper:0:1:0:1:0:0;gift=1;gt=0;rf=20;rs=`, 편성 `Hero,Bishop,NightLord`, `ClearedFloor=20`, 별 1·19·20층 7, 계정 Lv3 총 633, 깃털 563, 스킬북 539, 열쇠 4, 주문서 14종(시작 로그 값), 에너지 30.
- 테스트 중 바뀐 것: 캐릭터 차수·해금·다이아·교환 포인트(디버그), 판 보상으로 깃털 +156·스킬북 +183·주문서, 계정 경험치 +268(Lv4).
- 되돌림: `r:DebugSetBanner("")`, `r:DebugReset()`(덤프가 시작 전과 글자 단위로 같음), 각인 깃털·스킬북·주문서를 시작 값으로 넣고 `SaveGrace/SaveSkills/SaveSlots`, 계정 문서 `total=633`으로 `CommitAccount`.
- 확인: stop → play 후 불러오기 로그가 시작 전과 같음(`roster loaded` 동일, `grace loaded feathers=563`, `skills ... "b":539`, `slots loaded` 14종 같은 값, `account loaded lv=3 total=633`, `energy 30`, `relic keys=4`, `engrave loaded cleared=20`). 계정 문서의 `last`(마지막 정산 판 Id)만 테스트 판 Id로 남음(중복 정산 방지용 값, 진행 영향 없음).
- 메이커: 편집 모드.
