# 1단계 요구사항: 캐릭터 핵심 (데이터 + 계정 저장 + 서버 로직)

> 작성: 자료 조사 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(최우선) → `01-Agent-Rules.md` → 이 문서
> 범위: **UI 없음**. 데이터 표, 새 계정 컴포넌트 `PlayerRoster`, 서버 함수, 테스트용 `[VRF]` 로그까지.
> 이 문서만 보고 구현할 수 있도록 값과 규칙을 모두 적는다. 문서에 없는 판단이 필요하면 마스터 스펙 의도대로 정하고 보고에 적는다.

---

## 0. 결정 사항 요약

| 번호 | 결정 | 근거 |
|---|---|---|
| D1 | 직업 Id는 기존 그대로 `Hero / Bishop / NightLord / Bowmaster / Viper` | `HeroSkillTable.JobId`, `TowerRunManager.CompanionModels`, `PlayerEngrave.MainJob`과 호환 |
| D2 | 차수 시작 레벨 **1·7·13·19·25**, 차수 최대 레벨 **7·13·19·25·30** | 30레벨을 6레벨 간격으로 고르게 나눔. 5차는 25~30(5레벨 구간, 대신 필요 경험치가 가장 큼) |
| D3 | 차수 최대 레벨에 도달하면 **경험치가 더 쌓이지 않는다**(전직 관문). 전직하면 같은 레벨에서 다음 차수가 시작된다 | 스타레일·명일방주의 승급 관문 방식. 레벨과 전직이 서로를 막아 성장 속도 조절 |
| D4 | 전직 필요 전용 카드: 2차 **1장**, 3차 **2장**, 4차 **3장**, 5차 **5장** (합계 11장). 해금은 별도 1장 | 픽업 기준 약 150~180회에 1직업 완성. 3단계 뽑기 확률과 함께 조정 가능 |
| D5 | 공용 전직 카드는 전용 카드 부족분을 **1:1** 대체. 대체는 요청 인자 `useCommon = true`일 때만 | 희귀 재화를 실수로 쓰지 않게 |
| D6 | 공용 카드 1장으로 **미해금 직업 해금 가능** (`RequestUnlockWithCommon`) | 공용 카드 = "가장 좋은 결과" |
| D7 | 경험의 서 소·중·대 = **100 / 500 / 2,000** 경험치 | 1:5:20 (스타레일과 같은 비율) |
| D8 | 1→30레벨 총 경험치 **46,520** (대 약 23권) | 아래 레벨 표 |
| D9 | 차수 능력치 보너스 1~5차 = **+0 / +10 / +20 / +35 / +50 %** (레벨 스탯에 곱함) | 전직 순간 체감 상승 |
| D10 | 5차 이름 = `4차 이름 + " (5차)"` 예: `히어로 (5차)` | 마스터 스펙 예시 그대로 |
| D11 | 고유 패시브는 스킬 Id가 없으므로 새 Id `P_Hero / P_Bishop / P_NightLord / P_Bowmaster / P_Viper` | 4단계에서 차수별 효과를 이 Id에 연결 |
| D12 | **편성의 원본은 1~5단계 동안 PlayerEngrave(`MainJob/CompJob/Comp2Job`)**. PlayerRoster는 서버에서 그 값을 바꾸고 `SaveSkills()`를 호출한다. PlayerEngrave 코드는 **한 줄도 고치지 않는다** | PlayerEngrave는 다른 세션이 수정 중. `TowerRunManager.LoadFormation`도 그대로 동작 |
| D13 | 시작 지급(테스트 편의): 다이아 3,000, 경험의 서 소 50·중 20·대 10, 공용 카드 2, 전용 카드 히어로 3·비숍 1 | 테스트 T1~T14를 시작 재화만으로 돌릴 수 있게. 정식 출시 전 3단계·6단계에서 낮춘다 |
| D14 | 저장은 JSON이 아닌 **문자열 형식**(키 `OrbisRosterV1`) | `JSONEncode` 빈 테이블 실패 회피 (`01-Agent-Rules.md`) |
| D15 | 모든 요청은 `Do*`(서버 전용, 결과 코드 반환) + `Request*`(클라이언트→서버, 본인·로비 검사 후 `Do*` 호출) 쌍으로 만든다 | 테스트 에이전트가 `execute_script(server_main)`에서 `Do*`를 바로 부를 수 있음 (`senderUserId` 없음 문제 회피) |

---

## 1. 데이터 표

모두 `RootDesk/MyDesk/Data/` 에 **UTF-8 BOM** CSV + 같은 이름 `.userdataset`. `.userdataset`은 `RelicTable.userdataset` 내용을 복사해 `EntryKey`의 uuid, `name`, `id`를 새 값으로 바꾼다 (uuid는 새로 생성, `name` = 파일 이름). 메이커 새로고침 후 `_DataService:GetTable("이름")`으로 읽는다. 값은 문자열로 오므로 `tonumber` 처리.

### 1-1. `CharJobTable.csv` — 직업

열: `JobId,Order,JobGroup,Name1,Name2,Name3,Name4,Name5,BaseAtk,BaseHp,BaseDef,AtkPerLv,HpPerLv,DefPerLv,StartUnlocked,StartCards,ModelId,#Desc`

```csv
JobId,Order,JobGroup,Name1,Name2,Name3,Name4,Name5,BaseAtk,BaseHp,BaseDef,AtkPerLv,HpPerLv,DefPerLv,StartUnlocked,StartCards,ModelId,#Desc
Hero,1,전사,검사,파이터,크루세이더,히어로,히어로 (5차),10,120,1,2,12,1.2,1,3,towercompanion,근접 전사. 체력·방어 높음
Bishop,2,마법사,매지션,클레릭,프리스트,비숍,비숍 (5차),9,90,0,1.8,9,0.8,1,1,towerbishop,회복·지원 마법사
NightLord,3,도적,로그,어쌔신,허밋,나이트로드,나이트로드 (5차),11,95,0,2.2,9.5,0.9,1,0,towernightlord,표창 원거리 딜러
Bowmaster,4,궁수,아처,헌터,레인저,보우마스터,보우마스터 (5차),11,95,0,2.1,9.5,0.9,0,0,towerbowmaster,화살 원거리 딜러 (뽑기 해금)
Viper,5,해적,해적,인파이터,버커니어,바이퍼,바이퍼 (5차),10,110,1,2,11,1.1,0,0,towerviper,근접 격투 (뽑기 해금)
```

- `StartUnlocked` 1 = 처음부터 해금(Lv1, 1차). `StartCards` = 첫 지급 시 받는 그 직업 전용 카드 수.
- `ModelId`는 현재 `TowerRunManager.CompanionModels` 값. 2단계 외형 작업에서 참고만 한다.
- 기본 수치는 현재 판 기본값(공격 10, 체력 100, 레벨당 공 2·방 1·체 10) 주변으로 직업 차이만 둔 출발값. 4단계에서 조정한다.

### 1-2. `CharTierTable.csv` — 차수

열: `Tier,StartLevel,CapLevel,PromoteCards,StatPct,#Desc`

```csv
Tier,StartLevel,CapLevel,PromoteCards,StatPct,#Desc
1,1,7,0,0,1차. 해금 시 시작 (해금 카드 1장은 별도)
2,7,13,1,10,Lv7에서 전직. 보조기 해금·주력기 1차 진화
3,13,19,2,20,Lv13에서 전직. 기술 해금·보조기 진화
4,19,25,3,35,Lv19에서 전직. 하이퍼 해금·주력기 2차 진화
5,25,30,5,50,Lv25에서 전직. 궁극기 해금·기술 진화. Lv30 = 5차 완성
```

- `PromoteCards` = **그 차수로 올라갈 때** 필요한 전용 카드 수.
- `StatPct` = 그 차수일 때 레벨 스탯에 곱하는 보너스(%).

### 1-3. `CharLevelTable.csv` — 레벨

열: `Level,NeedExp,TotalExp`
- `NeedExp` = 그 레벨에서 다음 레벨까지 필요 경험치 (30은 0).
- `TotalExp` = 레벨 1 경험치 0에서 그 레벨에 도달할 때까지의 누적 경험치.
- 공식(참고): `NeedExp(L) = round((120 + 30(L-1) + 4(L-1)^2) / 10) × 10`

```csv
Level,NeedExp,TotalExp
1,120,0
2,150,120
3,200,270
4,250,470
5,300,720
6,370,1020
7,440,1390
8,530,1830
9,620,2360
10,710,2980
11,820,3690
12,930,4510
13,1060,5440
14,1190,6500
15,1320,7690
16,1470,9010
17,1620,10480
18,1790,12100
19,1960,13890
20,2130,15850
21,2320,17980
22,2510,20300
23,2720,22810
24,2930,25530
25,3140,28460
26,3370,31600
27,3600,34970
28,3850,38570
29,4100,42420
30,0,46520
```

차수별 필요 경험치: 1차(1→7) 1,390 / 2차(7→13) 4,050 / 3차(13→19) 8,450 / 4차(19→25) 14,570 / 5차(25→30) 18,060.

### 1-4. `CharSlotTable.csv` — 칸 6개 × 차수별 스킬 (진화표)

열: `JobId,SlotKey,SlotName,Order,T1,T2,T3,T4,T5,#Desc`
- `SlotKey`: `Passive / Main / Sub / Tech / Hyper / Ult` (돌파 연결·저장에 쓰는 고정 키)
- `T1~T5` = 그 차수일 때 칸의 스킬 Id. 빈 값 = 아직 잠김.
- 스킬 Id는 기존 `HeroSkillTable` / `JobSkillTable`의 SkillId. 패시브만 새 Id(D11).

```csv
JobId,SlotKey,SlotName,Order,T1,T2,T3,T4,T5,#Desc
Hero,Passive,고유 패시브,1,P_Hero,P_Hero,P_Hero,P_Hero,P_Hero,콤보 어택 (차수마다 효과 추가)
Hero,Main,주력기,2,H01,H02,H02,H08,H08,슬래시 블러스트 → 브랜디쉬 → 레이징 블로우
Hero,Sub,보조기,3,,H05,H11,H11,H11,인레이지 → 에픽 어드벤쳐
Hero,Tech,기술,4,,,H03,H03,H13,패닉 → 콤보 데스폴트
Hero,Hyper,하이퍼,5,,,,H10,H10,발할라
Hero,Ult,궁극기,6,,,,,H12,소드 오브 버닝 소울
Bishop,Passive,고유 패시브,1,P_Bishop,P_Bishop,P_Bishop,P_Bishop,P_Bishop,성수
Bishop,Main,주력기,2,B01,B02,B02,B06,B06,에너지 볼트 → 홀리 애로우 → 엔젤레이
Bishop,Sub,보조기,3,,B11,B09,B09,B09,어드밴스드 블레스 → 인피니티
Bishop,Tech,기술,4,,,B03,B03,B15,샤이닝 레이 → 디바인 퍼니시먼트
Bishop,Hyper,하이퍼,5,,,,B10,B10,헤븐즈 도어
Bishop,Ult,궁극기,6,,,,,B13,엔젤 오브 제미니
NightLord,Passive,고유 패시브,1,P_NightLord,P_NightLord,P_NightLord,P_NightLord,P_NightLord,표식
NightLord,Main,주력기,2,N01,N03,N03,N06,N06,럭키 세븐 → 트리플 스로우 → 쿼드러플 스로우
NightLord,Sub,보조기,3,,N02,N04,N04,N04,드레인 → 쉐도우 파트너
NightLord,Tech,기술,4,,,N07,N07,N14,써든레이드 → 풍마수리검
NightLord,Hyper,하이퍼,5,,,,N11,N11,블리딩 톡신
NightLord,Ult,궁극기,6,,,,,N15,다크로드의 비전서
Bowmaster,Passive,고유 패시브,1,P_Bowmaster,P_Bowmaster,P_Bowmaster,P_Bowmaster,P_Bowmaster,강화 화살
Bowmaster,Main,주력기,2,A01,A02,A02,A06,A06,애로우 블로우 → 애로우 봄 → 폭풍의 시
Bowmaster,Sub,보조기,3,,A05,A04,A04,A04,애로우 플래터 → 피닉스
Bowmaster,Tech,기술,4,,,A07,A07,A14,애로우 일루전 → 실루엣 미라주
Bowmaster,Hyper,하이퍼,5,,,,A10,A10,애로우 레인
Bowmaster,Ult,궁극기,6,,,,,A15,잔영의 시
Viper,Passive,고유 패시브,1,P_Viper,P_Viper,P_Viper,P_Viper,P_Viper,에너지
Viper,Main,주력기,2,V01,V02,V02,V07,V07,서머솔트 킥 → 백 스핀 블로우 → 데몰리션
Viper,Sub,보조기,3,,V05,V14,V14,V14,트랜스폼 → 퓨리어스 차지
Viper,Tech,기술,4,,,V04,V04,V12,에너지 버스트 → 하울링 피스트
Viper,Hyper,하이퍼,5,,,,V10,V10,스티뮬레이트
Viper,Ult,궁극기,6,,,,,V15,리버레이트 넵투누스
```

확인한 Id ↔ 이름 (HeroSkillTable 기준): H01 슬래시 블러스트, H02 브랜디쉬, H08 레이징 블로우, H05 인레이지, H11 에픽 어드벤쳐, H03 패닉, H13 콤보 데스폴트, H10 발할라, H12 소드 오브 버닝 소울 / B01 에너지 볼트, B02 홀리 애로우, B06 엔젤레이, B11 어드밴스드 블레스, B09 인피니티, B03 샤이닝 레이, B15 디바인 퍼니시먼트, B10 헤븐즈 도어, B13 엔젤 오브 제미니 / N01 럭키 세븐, N03 트리플 스로우, N06 쿼드러플 스로우, N02 드레인, N04 쉐도우 파트너, N07 써든레이드, N14 풍마수리검, N11 블리딩 톡신, N15 다크로드의 비전서 / A01 애로우 블로우, A02 애로우 봄, A06 폭풍의 시, A05 애로우 플래터, A04 피닉스, A07 애로우 일루전, A14 실루엣 미라주, A10 애로우 레인, A15 잔영의 시 / V01 서머솔트 킥, V02 백 스핀 블로우, V07 데몰리션, V05 트랜스폼, V14 퓨리어스 차지, V04 에너지 버스트, V12 하울링 피스트, V10 스티뮬레이트, V15 리버레이트 넵투누스.

### 1-5. `CharConfig.csv` — 상수

열: `Key,Value,#Desc` (RunConfig와 같은 형식)

```csv
Key,Value,#Desc
BookExpS,100,경험의 서 (소) 경험치
BookExpM,500,경험의 서 (중) 경험치
BookExpL,2000,경험의 서 (대) 경험치
MaxLevel,30,캐릭터 최대 레벨 (각성 전)
MaxTier,5,최대 차수
StartDiamonds,3000,첫 지급 다이아 (테스트용)
StartBookS,50,첫 지급 경험의 서 소
StartBookM,20,첫 지급 경험의 서 중
StartBookL,10,첫 지급 경험의 서 대
StartCommonCards,2,첫 지급 공용 전직 카드
AwakenMaxLevel,40,각성 최대 레벨 (6단계에서 사용)
```

---

## 2. 계정 저장: `PlayerRoster`

### 2-1. 파일과 부착

- 스크립트: `RootDesk/MyDesk/Character/PlayerRoster.mlua` (`@Component`, `script PlayerRoster extends Component`). 폴더 `Character`가 없으면 만든다(.directory는 메이커가 생성).
- 부착: **PlayerEngrave가 붙어 있는 플레이어 모델**(기대: `Global/DefaultPlayer.model`)에 `script.PlayerRoster` 컴포넌트를 추가한다. 반드시 msw-general의 `ModelBuilder`로 먼저 읽어서 PlayerEngrave가 있는 모델을 확인한 뒤 같은 모델에 추가한다 (셸로 `.model`을 읽거나 고치지 않는다). 기존 `Global/*.model`은 ModelBuilder로 제자리 수정이 허용된다.
- 다른 스크립트에서 찾기: `player:GetComponent("script.PlayerRoster")`.

### 2-2. 속성

```
property string RosterKey = "OrbisRosterV1"
property string LobbyMapName = "Lobby"
property number LoadRetryDelay = 10
property number SaveRetryDelay = 5

@Sync property boolean RosterLoaded = false
@Sync property integer Diamonds = 0
@Sync property integer BookS = 0
@Sync property integer BookM = 0
@Sync property integer BookL = 0
@Sync property integer CommonCards = 0
@Sync property integer ExchangePoints = 0      -- 3단계 뽑기 교환 포인트 (1단계는 저장만)
@Sync property integer AwakenBooks = 0         -- 6단계 각성의 서 (1단계는 저장만)
@Sync property integer AwakenStones = 0        -- 6단계 각성의 돌 (1단계는 저장만)
-- 직업 전용 카드 "Hero:3,Bishop:1,NightLord:0,Bowmaster:0,Viper:0" (CharJobTable 순서, 5직업 모두 기록)
@Sync property string JobCards = ""
-- 캐릭터 "Job:해금:레벨:경험치:차수:각성레벨:각성경험치|..." (CharJobTable 순서, 5직업 모두 기록)
@Sync property string CharData = ""
@Sync property integer RosterSeq = 0           -- 요청 처리마다 +1 (UI 갱신 신호)
@Sync property string RosterMsg = ""           -- 마지막 결과 코드 (아래 2-6)
@Sync property string RosterResult = ""        -- 마지막 결과 상세 "동작:직업:값..." (예 "level:Hero:1:7")

@HideFromInspector property boolean RosterDirty = false
@HideFromInspector property number RosterRetryTimer = 0
@HideFromInspector property number PartyCheckTimer = 0
```

- 서버 내부 상태는 `self._T`(PlayerEngrave와 같은 방식의 일반 테이블)에 둔다: `self._T.chars[job] = { unlocked, level, exp, tier, awkLv, awkExp }`, `self._T.cards[job] = n`, `self._T.extra = { 모르는 저장 키 = 값 }`, 표 캐시 `self._T.jobs / jobList / tiers / levels / slots / cfg`.
- **SyncTable은 쓰지 않는다.** 클라이언트는 `CharData`, `JobCards` 문자열을 파싱한다 (2단계 UI용 파서 `ParseCharData(string)`를 같은 컴포넌트에 실행 공간 지정 없이 둔다).
- 서버 상태가 바뀔 때마다 `SyncRoster()`로 Sync 속성을 다시 쓴다.

### 2-3. 저장 형식 (DataStorage 키 `OrbisRosterV1`)

`;`로 구역을 나누고 각 구역은 `키=값`. 값 안에는 `;`, `=`가 들어가지 않는다.

```
v=1;dia=3000;book=50,20,10;cc=2;pt=0;awk=0,0;jc=Hero:3,Bishop:1,NightLord:0,Bowmaster:0,Viper:0;ch=Hero:1:1:0:1:0:0|Bishop:1:1:0:1:0:0|NightLord:1:1:0:1:0:0|Bowmaster:0:1:0:1:0:0|Viper:0:1:0:1:0:0;gift=1
```

| 키 | 값 | 설명 |
|---|---|---|
| `v` | 1 | 형식 버전 |
| `dia` | 정수 | 다이아 |
| `book` | `소,중,대` | 경험의 서 |
| `cc` | 정수 | 공용 전직 카드 |
| `pt` | 정수 | 뽑기 교환 포인트 (3단계) |
| `awk` | `각성의서,각성의돌` | 6단계 재화 |
| `jc` | `Job:n,...` | 직업 전용 카드 |
| `ch` | `Job:해금(0/1):레벨:경험치:차수:각성레벨:각성경험치` 를 `|`로 연결 | 캐릭터. `경험치` = 현재 레벨 안에서 쌓인 값(0 ≤ 경험치 < NeedExp). 각성레벨 0 = 각성 전 |
| `gift` | 1 | 첫 지급 완료 표시 |

파싱 규칙:
- 구역 분리는 `string.find`로 직접 자른다 (`string.gsub` 사용 금지: LEA-1103). 각 구역은 첫 `=` 위치로 키/값을 나눈다.
- **모르는 키는 `_T.extra`에 보관했다가 저장할 때 그대로 다시 쓴다** (이후 단계에서 키를 추가해도 이전 코드가 지우지 않게).
- 숫자는 `math.floor(tonumber(x) or 0)`, 음수는 0으로.
- 표에 있는데 저장에 없는 직업(이후 추가 직업)은 `해금 = StartUnlocked`가 아니라 **0(미해금)**, Lv1, 1차로 채운다. 단 첫 지급(신규 계정)일 때만 `StartUnlocked`를 따른다.
- 값 범위 보정: 레벨 1~30, 차수 1~5, 레벨은 `StartLevel(차수) ≤ 레벨 ≤ CapLevel(차수)` 로 자르고, 경험치는 `0 ≤ 경험치 < NeedExp(레벨)` (최대 레벨이면 0).

### 2-4. 불러오기·저장 흐름

- `OnBeginPlay` (`@ExecSpace("ServerOnly")`): `LoadTables()` → `LoadRoster()`.
- `LoadRoster()`:
  - `local ds = _DataStorageService:GetUserDataStorage(self.Entity.PlayerComponent.ProfileCode)`
  - `local code, raw = ds:GetAndWait(self.RosterKey)`
  - `code == 0`이고 `raw`가 비어 있지 않음 → 파싱. `gift`가 없으면 첫 지급 적용.
  - `code == 1000002`(데이터 없음) 또는 `code == 0`이고 빈 문자열 → **첫 지급**(2-5) 후 즉시 저장.
  - 그 밖의 오류 → `log_warning`, `RosterRetryTimer = LoadRetryDelay`, `RosterLoaded = false` 유지. **불러오기 전에는 절대 저장하지 않는다.**
  - 성공 시 `RosterLoaded = true`, `SyncRoster()`, `log("[VRF] roster loaded " .. Encode())`.
- `OnUpdate` (`ServerOnly`): 미로드면 재시도 타이머. `RosterDirty`이고 저장 실패 상태면 `SaveRetryDelay` 후 재저장. 편성 보정 검사(2-8) 1초마다.
- `SaveRoster()` (`ServerOnly`): `RosterDirty = true` 후 `ds:SetAsync(self.RosterKey, self:Encode(), function(errorCode, key) ... end)`. 성공이면 `RosterDirty = false`, 실패면 `log_warning` + 재시도 타이머.
- **모든 재화·캐릭터 변경 직후 `SaveRoster()`** (뽑기·전직은 희귀 재화라 즉시 저장).
- `OnEndPlay` (`ServerOnly`): `RosterLoaded and RosterDirty`면 `ds:SetAndWait(...)`.

### 2-5. 첫 지급

- 직업마다: `StartUnlocked = 1`이면 해금, Lv1, 경험치 0, 1차, 각성 0. 그 외는 미해금(Lv1, 1차).
- 전용 카드 = `StartCards`. 다이아·경험의 서·공용 카드 = `CharConfig`의 `Start*`.
- 결과: 히어로·비숍·나이트로드 해금 Lv1, 다이아 3,000, 서 50/20/10, 공용 2, 카드 히어로 3·비숍 1.
- `gift=1` 기록 후 저장. 로그 `[VRF] roster first gift`.

### 2-6. 결과 코드 (`RosterMsg`)

| 코드 | 의미 |
|---|---|
| `ok_level` | 경험의 서 사용·레벨 상승 성공 (레벨이 안 올라도 경험치가 늘면 성공) |
| `ok_promote` | 전직 성공 |
| `ok_unlock` | 해금 성공 |
| `ok_party` | 편성 저장 성공 |
| `err_notready` | 불러오기 전 |
| `err_lobby` | 로비가 아님 / 본인 요청 아님 |
| `err_job` | 없는 직업 Id |
| `err_locked` | 미해금 직업 |
| `err_unlocked` | 이미 해금된 직업 (해금 요청 시) |
| `err_cap` | 차수 최대 레벨이라 경험치를 더 받을 수 없음 (전직 필요) |
| `err_max` | 최대 레벨·최대 차수 |
| `err_books` | 경험의 서 부족 / 사용 개수 0 |
| `err_level` | 전직 레벨 미달 |
| `err_cards` | 전직 카드 부족 |
| `err_party` | 편성 잘못됨 (중복·미해금·편성 원본 미로드) |

UI 문구는 2단계에서 이 코드를 한국어로 바꿔 표시한다. 모든 처리 후 `RosterSeq += 1`, `RosterMsg`, `RosterResult`를 쓰고 `[VRF]` 로그를 남긴다.

### 2-7. 서버 함수

공통 검사 `CanEditRoster(fromUserId)` (`ServerOnly`): `fromUserId == self.Entity.PlayerComponent.UserId` and `RosterLoaded` and 현재 맵 이름 == `LobbyMapName` (PlayerEngrave `CanEditSkill`과 같은 방식).

| 클라이언트 요청 (`@ExecSpace("Server")`) | 서버 본체 (`@ExecSpace("ServerOnly")`, 결과 코드 반환) |
|---|---|
| `RequestUseBooks(string jobId, integer s, integer m, integer l)` | `DoUseBooks(jobId, s, m, l)` |
| `RequestLevelTo(string jobId, integer targetLevel)` | `DoLevelTo(jobId, targetLevel)` |
| `RequestPromote(string jobId, boolean useCommon)` | `DoPromote(jobId, useCommon)` |
| `RequestUnlockWithCommon(string jobId)` | `DoUnlockWithCommon(jobId)` |
| `RequestSetParty(string j1, string j2, string j3)` | `DoSetParty(j1, j2, j3)` |

`Request*`는 `if not self:CanEditRoster(senderUserId) then self:Reply("err_lobby", "") return end` 후 `Do*`를 부른다. `Do*`는 `RosterLoaded`가 아니면 `err_notready`.

#### (1) 경험치 반영 `AddExp(jobId, gain)` — 내부

1. `cap = CapLevel(tier)`, `capTotal = TotalExp(cap)`, `cur = TotalExp(level) + exp`.
2. `new = math.min(cur + gain, capTotal)`. (차수 최대 레벨을 넘는 경험치는 버린다)
3. `level` = `TotalExp(L) ≤ new`인 가장 큰 L (단 `≤ cap`), `exp = new - TotalExp(level)`.
4. 반환: 실제 반영된 경험치 `new - cur`.

#### (2) 경험의 서 직접 사용 `DoUseBooks(jobId, s, m, l)`

- 검사: 직업 존재(`err_job`), 해금(`err_locked`), 음수는 0, `s+m+l == 0` 또는 보유 초과 → `err_books`, 이미 차수 최대 레벨 → (최대 차수·최대 레벨이면 `err_max`, 아니면 `err_cap`).
- 처리 순서: **대 → 중 → 소**, 한 권씩 `AddExp`. 차수 최대 레벨에 도달하면 **즉시 멈추고 남은 책은 소모하지 않는다.** 마지막 한 권의 초과분만 버려진다.
- 결과 `ok_level`, `RosterResult = "level:Job:이전레벨:새레벨"`, 로그 `[VRF] roster books job=.. used=s,m,l lv a->b exp=..`.

#### (3) 목표 레벨까지 자동 `DoLevelTo(jobId, targetLevel)` + 계산 함수 `PlanBooks`

- `PlanBooks(jobId, targetLevel)` — **실행 공간 지정 없음**(클라이언트 미리보기에서도 같은 계산을 쓰도록 표와 Sync 값만 사용). 인자로 현재 레벨·경험치·차수·보유 책 수를 받는 순수 함수 `PlanBooksFor(level, exp, tier, haveS, haveM, haveL, targetLevel)`로 만들고, `PlanBooks`는 그것을 감싼다. 반환 `{ s, m, l, gain, newLevel }`.
  1. `target = math.min(targetLevel, CapLevel(tier))`. `R = TotalExp(target) - (TotalExp(level) + exp)`. `R ≤ 0`이면 모두 0.
  2. 넘치지 않게 채우기: 대 `k = min(haveL, floor(R/2000))`, `R -= k*2000`; 중, 소 순서로 같은 방식.
  3. `R > 0`이 남으면 남은 책 중 **가장 작은 책 1권**(소→중→대 순으로 보유한 것)을 추가 → 최소 초과.
  4. 그래도 `R > 0`(책이 모자람)이면 그대로 둔다 → 목표보다 낮게 오른다 (가능한 만큼).
  5. `gain`, `newLevel`은 `AddExp` 규칙(차수 최대 레벨에서 잘림)으로 계산.
- `DoLevelTo`: 검사는 (2)와 같음 + `targetLevel ≤ level` → `err_level`. 계획의 `s+m+l == 0` → `err_books`. 계획대로 책을 빼고 `AddExp(gain)`. 결과 `ok_level`.
- 2단계 UI 버튼: `+1레벨` = `DoLevelTo(level+1)`, `최대` = `DoLevelTo(CapLevel(tier))`.

#### (4) 전직 `DoPromote(jobId, useCommon)`

1. 직업 존재·해금 검사. `tier == 5` → `err_max`.
2. `level < CapLevel(tier)` → `err_level`.
3. `need = PromoteCards(tier + 1)`, `have = 전용 카드`.
   - `have ≥ need` → 전용 카드 `need` 소모. (전용 카드가 충분하면 `useCommon`이어도 공용은 쓰지 않는다)
   - `have < need` 이고 `useCommon` 이고 `have + CommonCards ≥ need` → 전용 카드 전부(`have`) + 공용 `need - have` 소모.
   - 그 외 → `err_cards` (아무것도 소모하지 않음).
4. `tier += 1`. 레벨 그대로, 경험치 0 (레벨은 이미 새 차수의 StartLevel과 같다).
5. 결과 `ok_promote`, `RosterResult = "promote:Job:새차수:전용사용:공용사용"`, 로그 `[VRF] roster promote job=.. tier a->b job=.. common=..`.

#### (5) 해금

- `GiveJobCard(jobId, n)` (`ServerOnly`, 3단계 뽑기가 호출): `n ≤ 0`이면 무시. 미해금이면 **1장을 해금에 쓰고**(Lv1, 경험치 0, 1차, 각성 0) 나머지 `n-1`장은 카드로 적립. 해금이면 `n`장 적립. 반환 `"unlock"` 또는 `"card"`. 로그 `[VRF] roster card job=.. n=.. result=..`. 저장.
- `DoUnlockWithCommon(jobId)`: 해금이면 `err_unlocked`, `CommonCards < 1` → `err_cards`, 아니면 공용 1장 소모 후 해금. `ok_unlock`.

#### (6) 재화 지급·사용 (`ServerOnly`, 다른 단계가 호출)

| 함수 | 동작 |
|---|---|
| `GiveDiamonds(integer n)` | n > 0일 때 더하고 저장 |
| `SpendDiamonds(integer n) → boolean` | 보유 ≥ n이면 빼고 저장 후 true |
| `GiveBooks(integer s, integer m, integer l)` | 음수 무시, 더하고 저장 |
| `GiveCommonCard(integer n)` | 더하고 저장 |
| `GiveExchangePoints(integer n)` / `SpendExchangePoints(integer n) → boolean` | 3단계용 |
| `IsUnlocked(string jobId) → boolean` | 해금 여부 |
| `GetChar(string jobId) → table` | `{ unlocked, level, exp, tier, awkLv, awkExp }` 복사본 |

#### (7) 편성 `DoSetParty(j1, j2, j3)`

1. `eng = self.Entity:GetComponent("script.PlayerEngrave")`. `eng == nil` 또는 `not eng.SkillLoaded` → `err_party`.
2. 세 직업 모두 표에 있고, 서로 다르고, 모두 해금 → 아니면 `err_party`.
3. `eng.MainJob = j1; eng.CompJob = j2; eng.Comp2Job = j3; eng:SaveSkills()` — PlayerEngrave의 기존 서버 함수 호출만 하고 코드는 고치지 않는다.
4. `ok_party`, 로그 `[VRF] roster party a,b,c`.

### 2-8. 편성 자동 보정 (기존 계정 대비)

기존 계정은 PlayerEngrave 편성에 미해금 직업(보우마스터·바이퍼)이 들어 있을 수 있다.

- `OnUpdate`에서 1초마다: `RosterLoaded`이고 `eng.SkillLoaded`이면 `eng.MainJob / CompJob / Comp2Job`을 검사.
- 미해금이거나 겹치는 칸은 **CharJobTable 순서로 해금된 직업 중 아직 안 쓴 것**으로 바꾼다. 하나라도 바뀌면 `eng:SaveSkills()` + 로그 `[VRF] roster party fixed a,b,c`.
- 해금 직업은 항상 3개 이상(첫 지급 3직업, 해금은 줄지 않음)이므로 항상 채울 수 있다.
- 탑 맵에서는 검사하지 않는다(판 도중 편성 변경 방지): 현재 맵 이름이 `LobbyMapName`일 때만.

### 2-9. 판에 넘길 값 (4단계에서 사용, 1단계에서 만들어 둔다)

```
@ExecSpace("ServerOnly")
method table CharRunData(string jobId)
-- 반환:
-- {
--   loaded = true/false, unlocked = bool, jobId = "Hero",
--   level = 1~30, tier = 1~5, tierName = "파이터",
--   atk = 정수, hp = 정수, def = 정수,
--   awakenLv = 0,
--   slots = { Passive = "P_Hero", Main = "H02", Sub = "H05", Tech = "", Hyper = "", Ult = "" },
--   skills = { "P_Hero", "H02", "H05" }   -- slots 중 빈 값이 아닌 것 (SlotOrder 순)
-- }

@ExecSpace("ServerOnly")
method table PartyRunData()
-- PlayerEngrave 편성 순서(MainJob, CompJob, Comp2Job)대로 CharRunData 3개를 담은 배열.
```

- 능력치 공식(실행 공간 지정 없는 `CharStat(jobId, level, tier)`로 만들어 UI도 같은 값을 쓴다):
  - `atk = round((BaseAtk + AtkPerLv × (level-1)) × (1 + StatPct(tier)/100))`
  - `hp`, `def`도 같은 방식. `round(x) = math.floor(x + 0.5)`.
  - 예: 히어로 Lv1 1차 = 공 10 / 체 120 / 방 1. 히어로 Lv7 2차 = 공 round(22×1.1)=24 / 체 round(192×1.1)=211 / 방 round(8.2×1.1)=9. 히어로 Lv30 5차 = 공 round(68×1.5)=102 / 체 round(468×1.5)=702 / 방 round(35.8×1.5)=54.
- 1단계에서는 **TowerRunManager를 고치지 않는다.** 4단계에서 `RecalcStats`와 전투 `BeginRun`이 이 함수를 부른다.

### 2-10. 테스트 전용 함수 (`ServerOnly`)

| 함수 | 동작 |
|---|---|
| `DebugGrant(integer dia, integer s, integer m, integer l, integer common)` | 재화 지급 후 저장 |
| `DebugSetChar(string jobId, integer unlocked, integer level, integer tier)` | 캐릭터 상태 강제 설정(경험치 0, 범위 보정) 후 저장 |
| `DebugSetCards(string jobId, integer n)` | 전용 카드 수 설정 |
| `DebugReset()` | 첫 지급 상태로 되돌리고 저장 (`extra`는 유지) |
| `DebugDump() → string` | `Encode()` 결과를 로그 `[VRF] roster dump ...`로 출력하고 반환 |

모든 함수는 로그를 남긴다. 6단계 정리 때 삭제하거나 남겨 둘지 결정한다.

---

## 3. 구현 순서

1. **1-A 데이터 표**: CSV 5개 + userdataset 5개 → 메이커 정지 → 새로고침 → 빌드 로그 확인.
2. **1-B 컴포넌트 골격**: `PlayerRoster.mlua` (속성·표 읽기·인코드/디코드·불러오기·저장·첫 지급) → ModelBuilder로 플레이어 모델에 부착 → 새로고침 → 플레이.
3. **1-C 레벨업**: `AddExp`, `PlanBooksFor/PlanBooks`, `DoUseBooks`, `DoLevelTo`, `Request*` 래퍼.
4. **1-D 전직·해금·편성·판 전달**: `DoPromote`, `GiveJobCard`, `DoUnlockWithCommon`, 재화 함수, `DoSetParty`, 편성 자동 보정, `CharStat`, `CharRunData`, `PartyRunData`, 테스트 함수.
5. 각 단계마다 `[VRF]` 로그 확인, 마지막에 아래 테스트 전부를 한 번의 플레이에서 확인.

주의:
- 메서드 설명 주석은 메서드 본문 첫 줄에 쓴다.
- 컴포넌트에 선언하지 않은 필드 대입은 조용히 실패하므로 내부 상태는 `self._T`에 둔다. `_T`는 mLua가 인스턴스마다 제공하는 내부 테이블이며 PlayerEngrave도 선언 없이 `self._T.xxx = ...`로 쓴다. 같은 방식으로 쓴다.
- `member`는 예약어. 변수 이름으로 쓰지 않는다.
- `PlayerEngrave.mlua`는 **읽기만** 한다. 필요한 것은 `MainJob / CompJob / Comp2Job / SkillLoaded` 읽기·쓰기와 `SaveSkills()` 호출뿐이다.

---

## 4. 완료 조건과 테스트 케이스

완료 조건:
- 메이커 빌드 오류 0, LSP 진단 오류 0, 콘솔에 `LEA-` 오류 없음.
- 아래 T1~T15를 `maker_execute_script`(`server_main`)로 확인하고 `[VRF]` 로그를 보고서에 붙인다.
- PlayerEngrave·TowerRunManager·기존 UI는 수정하지 않았음을 `git diff --stat`로 확인.

테스트 준비 (server_main):
```lua
local p = nil
for _, e in pairs(_UserService.Users) do p = e break end
local r = p:GetComponent("script.PlayerRoster")
r:DebugReset()
```
(`_UserService.Users`의 형태는 `.d.mlua`로 확인해 맞춘다. 플레이어가 로비에 있어야 한다.)

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T1 첫 지급 | `DebugReset()` 후 `DebugDump()` | 히어로·비숍·나이트로드 `1:1:0:1:0:0`, 보우마스터·바이퍼 `0:...`, `dia=3000;book=50,20,10;cc=2`, `jc=Hero:3,Bishop:1,...` |
| T2 직접 사용 | `DoUseBooks("Hero", 2, 0, 0)` | `ok_level`, 히어로 Lv2 경험치 80 (200-120), 소 48 |
| T3 관문 멈춤 | `DoUseBooks("Hero", 0, 0, 1)` (대 1권 = 2000) | 히어로 Lv7 경험치 0, 대 9. 초과분 버려짐 |
| T4 관문에서 거부 | `DoUseBooks("Hero", 1, 0, 0)` | `err_cap`, 소 개수 변화 없음 |
| T5 자동 계산 | `DebugSetChar("Bishop",1,1,1)` → `PlanBooks("Bishop", 7)` → `DoLevelTo("Bishop", 7)` | 계획 `l=0, m=2, s=4` (R=1390 → 중 2 = 1000, 소 3 = 300, 남은 90 → 소 1 추가), 비숍 Lv7 경험치 0, 소 −4·중 −2 |
| T6 전직 레벨 미달 | `DebugSetChar("NightLord",1,5,1)` → `DoPromote("NightLord", false)` | `err_level` |
| T7 전직 성공(전용) | 히어로 Lv7 1차, 카드 3 → `DoPromote("Hero", false)` | `ok_promote`, 히어로 2차 Lv7, 카드 2 |
| T8 카드 부족 | `DebugSetChar("Bishop",1,13,2)`, `DebugSetCards("Bishop",1)` → `DoPromote("Bishop", false)` | `err_cards` (3차는 2장 필요), 카드 1 그대로 |
| T9 공용 대체 | 이어서 `DoPromote("Bishop", true)` | `ok_promote`, 비숍 3차, 전용 0, 공용 2→1 |
| T10 공용도 부족 | `DebugSetChar("NightLord",1,7,1)`, `DebugSetCards("NightLord",0)`, 공용 0으로 만든 뒤 `DoPromote("NightLord", true)` | `err_cards`, 아무것도 소모 안 됨 |
| T11 해금(카드) | `GiveJobCard("Bowmaster", 3)` | 반환 `"unlock"`, 보우마스터 해금 Lv1 1차, 카드 2 |
| T12 해금(공용) | 공용 1 이상에서 `DoUnlockWithCommon("Viper")` → 다시 한 번 | 첫 번째 `ok_unlock`·공용 −1, 두 번째 `err_unlocked` |
| T13 편성 | `DoSetParty("Hero","Hero","Bishop")` → `DoSetParty("Viper","Hero","Bishop")`(미해금 상태에서) → 해금 후 같은 요청 | 1·2번째 `err_party`, 3번째 `ok_party` 및 `PlayerEngrave.MainJob == "Viper"` |
| T14 편성 보정 | 바이퍼 미해금 상태(`DebugSetChar("Viper",0,1,1)`)에서 `eng.MainJob = "Viper"` 직접 대입 → 2초 뒤 확인 | `[VRF] roster party fixed`, MainJob이 해금 직업으로 바뀜 |
| T15 저장 유지 | 위 상태에서 `DebugDump()` 기록 → `maker_run_command stop` → `play` → 다시 `DebugDump()` | 두 덤프가 같다 (재접속 후 유지). `CharRunData("Hero")`가 `tier=2, tierName="파이터", atk=24, hp=211, def=9, slots.Main="H02", slots.Sub="H05"` |

추가 확인:
- 클라이언트(`client` 컨텍스트)에서 `CharData`, `JobCards`, `Diamonds` Sync 값이 서버와 같은지 1회 확인.
- 로비가 아닌 맵(탑)에서 클라이언트가 `RequestUseBooks`를 불렀을 때 `err_lobby`(가능하면 확인, 어렵다면 정적 검사로 대체하고 보고).

---

## 5. 2단계 이후로 넘기는 것

- UI 문구(결과 코드 → 한국어), 캐릭터 화면, 차수 외형 → 2단계.
- 뽑기에서 `GiveBooks / GiveJobCard / GiveCommonCard / GiveExchangePoints` 호출, 다이아 지급 → 3단계.
- `CharRunData` 사용, 판 안 레벨 삭제 → 4단계.
- 각성 레벨·경험치 사용 → 6단계 (저장 칸만 1단계에서 준비).
- 시작 재화 축소(D13) → 3단계 또는 6단계 밸런스.

---

## 1단계 보완 (2단계 조사에서 결정)

> 작성: 자료 조사 에이전트(2단계), 2026-11-22
> 1단계 테스트 에이전트가 남긴 질문 13개에 대한 결정. 코드 수정이 필요한 것은 맨 아래 "구현 에이전트가 2단계 시작 때 할 1단계 보완 작업"에 정확히 적었다. 이 작업은 `Phase-2-Character-UI.md`의 작업 단위 **2-A**다.

| 번호 | 질문 | 결정 | 코드 수정 |
|---|---|---|---|
| Q1 | 플레이어 찾기 방법 | 테스트 준비 코드는 `_UserService.Users`가 아니라 **`for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end`** (server_main)를 쓴다. 위 4절의 테스트 준비 코드도 이 방식으로 읽는다 | 없음 (문서만) |
| Q2 | 테스트용 `DebugSetCurrency` 필요 여부 / `DebugGrant` 음수 허용 여부 | **둘 다 허용.** `DebugGrant`는 지금처럼 음수를 받아 "현재값 + 인자"를 0 아래로 내리지 않는다(이미 `math.max(0, ...)` 동작이므로 그대로 둔다). 2단계 UI 테스트(재화 0 → 버튼 비활성)를 위해 **절대값 설정 함수 `DebugSetCurrency`를 추가**한다 | 있음 (작업 a) |
| Q3 | `DoUseBooks` 대→중→소 순서가 경험치를 낭비함 | **순서를 바꾸지 않는다.** 차수 관문에 닿으면 남은 책을 소모하지 않으므로 대→중→소가 "넘치는 책이 최대 1권"이라 오히려 낭비가 적은 경우가 많다. 그리고 **2단계 UI는 `DoUseBooks`를 쓰지 않는다** — 레벨업 버튼은 모두 `PlanBooksFor`(넘침 최소 계산) + `RequestLevelTo`를 쓴다. `DoUseBooks`는 테스트·이후 직접 입력용으로만 남긴다 | 없음 |
| Q4 | 책이 모자라 목표 레벨 미달이어도 `ok_level` 반환 | **새 결과 코드(`ok_level_partial`)는 만들지 않는다.** 대신 `DoLevelTo`의 `RosterResult`에 목표 레벨을 5번째 값으로 붙인다: `"level:Job:이전레벨:새레벨:목표레벨"`. UI는 `새레벨 < 목표레벨`이면 "경험의 서가 부족해 Lv.b까지 올렸습니다"를 띄운다. 목표레벨 = `math.min(targetLevel, CapLevel(tier))` | 있음 (작업 c) |
| Q5 | `err_lobby`일 때 `RosterResult`가 빈 문자열 | **동작 이름을 넣는다.** `Request*` 5개가 `Reply("err_lobby", "level:"..jobId)` 형식으로 보낸다 (아래 작업 b 표) | 있음 (작업 b) |
| Q6 | `JobCards` Sync에는 모르는 직업 원문이 포함, `CharData`에는 없음 | **필터하지 않는다.** UI는 항상 `CharJobTable` 순서로 직업을 돌고 `ParseJobCards(JobCards)[jobId]`로 찾으므로 모르는 직업 항목은 무시된다. 저장 보존(이후 직업 추가 대비)이 우선 | 없음 |
| Q7 | `DebugReset`이 `extra`(모르는 키)를 유지 | **유지한다.** 2단계 테스트에는 완전 초기화가 필요 없다. 3단계 이후 키가 늘어 완전 초기화가 필요해지면 그 단계에서 `DebugWipe()`를 추가한다 | 없음 |
| Q8 | 탑에서는 `FixParty`가 돌지 않아 `PartyRunData`가 잠긴 직업을 넘길 수 있음 | **4단계 요구사항으로 기록한다.** 4단계 `BeginRun`/`LoadFormation` 교체 때: `PartyRunData()`의 각 항목이 `unlocked == false`이거나 중복이면 `FixParty`와 같은 규칙(CharJobTable 순서로 해금된 미사용 직업)으로 바꾼 뒤 판을 시작하고 `[VRF] run party fixed a,b,c` 로그를 남긴다. 2단계에서는 손대지 않는다 | 4단계 |
| Q9 | 차수 보너스 적용 범위 표현 | 문서 표현을 바로잡는다: **차수 보너스는 `(기본값 + 레벨당 × (레벨-1))` 합계 전체에 곱한다.** (구현 `CharStat`이 이미 이 방식이며, D9의 "레벨 스탯에 곱함"은 이 뜻이다) | 없음 (문서만) |
| Q10 | 각성 데이터 필드 | **각성은 `awkLv / awkExp` 필드만 쓴다.** `level`은 30에 고정된 채 둔다(`NormalizeChar`가 5차 레벨을 25~30으로 자르므로 `level`을 31 이상으로 올리면 잘린다). 6단계는 각성 레벨 표시를 `30 + awkLv`로 계산한다 | 6단계 |
| Q11 | 차수 관문에서 넘치는 경험치 소실 표시 | **UI 미리보기에 표시한다.** 레벨업 미리보기에서 `책 경험치 합 - plan.gain > 0`이면 "(넘치는 경험치 N은 사라집니다)"를 붙인다. `PlanBooksFor`는 넘침을 최소화하므로 대부분 0이다. 전직 자체는 경험치를 0으로 만들지만 관문에서는 경험치가 0이므로 전직 때 사라지는 값은 없다 | 없음 (2단계 UI) |
| Q12 | 뽑기 보상 대비 밸런스 | **3단계로 넘긴다.** 3단계 뽑기 확률·교환 포인트 확정 때 "1직업 5차 완성까지 필요한 뽑기 횟수"를 계산해 문서에 적는다 | 3단계 |
| Q13 | 파티 기준 데이터를 나중에 PlayerRoster로 옮길지 | **6단계에서 옮긴다.** 1~5단계는 PlayerEngrave(`MainJob/CompJob/Comp2Job`)가 원본이다. 6단계 옛 시스템 정리 때 `OrbisRosterV1`에 `pt3=Job,Job,Job` 같은 키를 추가하고 PlayerEngrave 편성 읽기·쓰기를 PlayerRoster로 바꾼다 | 6단계 |

### 구현 에이전트가 2단계 시작 때 할 1단계 보완 작업 (`RootDesk/MyDesk/Character/PlayerRoster.mlua`)

**(a) 테스트 함수 추가 — `DebugSetCurrency`**

```lua
@ExecSpace("ServerOnly")
method void DebugSetCurrency(integer dia, integer s, integer m, integer l, integer common)
	-- 테스트: 재화를 절대값으로 설정 (음수는 0) 후 저장.
	if not self.RosterLoaded then return end
	self.Diamonds = math.max(0, dia)
	self.BookS = math.max(0, s)
	self.BookM = math.max(0, m)
	self.BookL = math.max(0, l)
	self.CommonCards = math.max(0, common)
	self:SaveRoster()
	log("[VRF] roster debug currency dia=" .. tostring(self.Diamonds) .. " books=" .. tostring(self.BookS) .. "," .. tostring(self.BookM) .. "," .. tostring(self.BookL) .. " common=" .. tostring(self.CommonCards))
end
```

**(b) `err_lobby` 결과에 동작 이름 넣기** — `Request*` 5개의 첫 줄만 바꾼다.

| 함수 | 바꿀 `Reply` 두 번째 인자 |
|---|---|
| `RequestUseBooks(jobId, s, m, l)` | `"level:" .. jobId` |
| `RequestLevelTo(jobId, targetLevel)` | `"level:" .. jobId` |
| `RequestPromote(jobId, useCommon)` | `"promote:" .. jobId` |
| `RequestUnlockWithCommon(jobId)` | `"unlock:" .. jobId` |
| `RequestSetParty(j1, j2, j3)` | `"party:" .. j1 .. "," .. j2 .. "," .. j3` |

**(c) `DoLevelTo` 결과에 목표 레벨 추가** — 마지막 `Reply` 한 줄만 바꾼다.

```lua
local target = math.min(targetLevel, self:CapLevel(c.tier))
return self:Reply("ok_level", "level:" .. jobId .. ":" .. tostring(before) .. ":" .. tostring(c.level) .. ":" .. tostring(target))
```

- `DoUseBooks`의 결과 형식(`"level:Job:a:b"`, 4개 값)은 그대로 둔다. UI 파서는 값이 4개면 목표 비교를 하지 않는다.

**(d) 문서 표현 반영** — Q1·Q9는 이 절이 정본이다(위 본문은 그대로 둔다).

**완료 조건 (2-A):** 메이커 빌드 오류 0, 아래 확인을 `server_main`에서 실행해 `[VRF]` 로그 확인.
1. `r:DebugSetCurrency(0, 0, 0, 0, 0)` → `[VRF] roster debug currency dia=0 books=0,0,0 common=0`.
2. 히어로 Lv1·서 소 1권만 있는 상태에서 `r:DoLevelTo("Hero", 7)` → `[VRF] roster reply ok_level level:Hero:1:1:7` (경험치만 오름, 레벨 미달).
3. 탑 맵에서 클라이언트가 `RequestPromote("Hero", false)` → `[VRF] roster reply err_lobby promote:Hero` (탑 이동이 번거로우면 정적 검사로 대체하고 보고).
