# 3단계 요구사항: 다이아 캐릭터 뽑기 + 다이아 지급

> 작성: 자료 조사 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(최우선) → `01-Agent-Rules.md` → `Docs/UI-Design-System.md` → 이 문서
> 선행: 1단계(`Phase-1-Character-Core.md` + 끝의 "1단계 보완"), 2단계(`Phase-2-Character-UI.md` + 끝의 "2단계 보완") 완료
> 이 문서만 보고 구현할 수 있도록 값·저장 형식·엔티티·흐름·테스트를 모두 적는다. 문서에 없는 판단이 필요하면 마스터 스펙 의도대로 정하고 보고에 적는다.

---

## 0. 결정 사항 요약

| 번호 | 결정 | 근거 |
|---|---|---|
| G1 | 비용 **1회 100 / 10회 1,000 다이아**. 10회 할인 없음 | 마스터 스펙 5절 |
| G2 | 결과 5종 확률(합 100%): **경험의 서 소 45% / 중 30% / 대 12% / 직업 전용 전직 카드 12% / 공용 전직 카드 1%** | 로드맵 제안값 그대로. 기대 경험치 1회 435(밸런스 6절) |
| G3 | 전용 카드 12%의 직업 배분: 픽업 없음 = 표의 모든 직업 균등(5직업이면 각 2.4%). **픽업 중 = 12% 중 `PickupShare`%(기본 50 → 6%)를 픽업 직업에, 나머지를 다른 직업에 균등**(4직업 각 1.5%) | "기간 픽업 = 픽업 직업 전용 카드 확률 상승" |
| G4 | **한 화면·한 버튼 쌍**: 지금 시각에 진행 중인 픽업이 있으면 픽업 확률, 없으면 상시 통합 확률로 뽑는다. 픽업 기간은 데이터 표 `CharBannerTable`의 시작·끝(한국 시간) | 상시 통합 + 기간 픽업. 화면을 둘로 나누지 않아 단순 |
| G5 | 서버 시간은 **`DateTime.UtcNow.Elapsed`**(이미 `PlayerEngrave`·`LobbyManager`가 씀)와 `DateTime(y,m,d,h,mi,s).Elapsed` 비교. 단위는 하루 길이로 직접 계산해 단위 가정을 하지 않는다(3-2) | `Environment/NativeScripts/Misc/DateTime.d.mlua` 확인 |
| G6 | **10회 보장**: 10개 결과에 직업 카드(전용·공용)가 하나도 없으면 **마지막(10번째) 결과를 "카드만" 확률로 다시 뽑아 바꾼다**(전용 12 : 공용 1 비율, 전용의 직업 배분은 G3 그대로) | 마스터 스펙. 10회에 카드가 0장일 확률은 0.87¹⁰ ≈ 24.8% |
| G7 | 처음 얻는 전용 카드 = **그 카드 1장으로 해금**(Lv1·1차), 같은 뽑기에서 또 나오면 그때부터 적립. 1단계 `GiveJobCard` 규칙과 같다 | 1단계 2-7 (5) |
| G8 | **교환 포인트: 뽑기 1회당 1점(10회 = 10점), 30점 = 원하는 직업 전용 카드 1장**. 잠긴 직업이면 그 카드로 바로 해금. 포인트는 기간과 무관하게 계속 쌓이고 사라지지 않는다. 상한 없음 | 마스터 스펙 참고안. 블루 아카이브식 "이해하기 쉬운 천장" |
| G9 | **확률 천장(연속 실패 보정)은 넣지 않는다.** 교환 포인트가 30회마다 확정 카드 1장을 주는 천장 역할, 10회 보장이 하한 역할 | 같은 기능 두 개는 화면·설명만 복잡 |
| G10 | **5차를 마친 직업(해금 + 5차)의 전용 카드는 교환 포인트 10점으로 바뀐다**(뽑기 결과 표시 "+10 포인트"). 그런 직업은 교환 대상으로 고를 수 없다 | 쓸모없는 카드 방지. 6단계 각성은 카드를 쓰지 않음(마스터 스펙 2절) |
| G11 | 뽑기·교환 서버 코드는 **`PlayerRoster`에 추가**한다(새 컴포넌트 아님). 재화·카드·해금·저장이 모두 여기 있어 한 번의 저장으로 끝난다. 결과 알림은 **별도 Sync 값 `GachaSeq / GachaMsg / GachaResult`** (캐릭터 화면의 `RosterSeq` 처리와 섞이지 않게) | 1단계 구조(D15 `Do*`/`Request*` 쌍)와 일관 |
| G12 | 저장 키 `OrbisRosterV1`에 **`gt`(누적 뽑기 수), `rf`(첫 클리어 다이아를 받은 최고 층), `rs`(마지막 별 상자 다이아를 받은 구간 목록)** 추가. `pt`(교환 포인트)는 1단계 칸 재사용 | 1단계 저장 형식(모르는 키 보존 규칙과 충돌 없게 아는 키 목록에도 추가) |
| G13 | 로비 `BtnGacha`("스킬 뽑기") → **글자 "캐릭터 뽑기", 클릭 = 새 `_CharGachaUILogic:Open()`**. 옛 `_GachaUILogic`(스킬 뽑기)은 코드·UI를 지우지 않고 진입만 끊는다(삭제는 6단계) | 사용자 지시 |
| G14 | 다이아 지급: **층 첫 클리어 = `20 + 2 × 층`**(1층 22 ~ 97층 214, 합계 11,446), **별 구간 마지막 상자(3번째, 별 30개) = 300**(9구간, 합계 2,700). 지급 판정은 `PlayerRoster`가 자기 저장값(`rf`, `rs`)으로 하므로 **중복 지급이 구조적으로 불가능**하다 | 로드맵 제안값. 6절 밸런스 확인 |
| G15 | **`PlayerEngrave.mlua`는 한 줄도 고치지 않는다.** 첫 클리어는 `PlayerEngrave.ClearedFloor`(Sync), 별 상자는 `PlayerEngrave.SectionData`(Sync)를 **읽기만** 해서 PlayerRoster가 1초마다 확인·지급한다. 탑 결과 화면 표시를 위해 `TowerRunManager`에서 같은 지급 함수를 한 줄 더 부른다 | 다른 세션 작업 중인 파일 보호. 지급 함수는 몇 번 불러도 결과가 같다 |
| G16 | 기존 계정(이미 깬 층·받은 상자가 있는 계정)은 3단계 첫 접속 때 **소급 지급**한다(`rf`, `rs`가 없으면 0부터 계산) | 새 계정과 총량이 같아 공정. 별도 이전 코드 불필요 |
| G17 | 첫 지급(다이아 3,000·서 50/20/10·공용 2·카드 히어로 3·비숍 1)은 **3단계에서 바꾸지 않는다**(1·2단계 회귀 테스트 기대값 유지). 출시 권장값은 6절에 적고 6-D에서 적용 | 1단계 D13 |

---

## 1. 작업 단위

| 단위 | 내용 | 바꾸는 파일 | 완료 조건 |
|---|---|---|---|
| **3-A** | 2단계 보완(B1·B2·B3·띠 배경·바이퍼 5차 너클·미리보기 자세 시도·넘침 문구) | `Character/CharacterUILogic.mlua`, `ui/CharacterUI`, `Data/CharOutfitTable.csv` | `Phase-2-Character-UI.md` 끝 "2단계 보완"의 3-A 완료 조건 |
| **3-B** | 뽑기 데이터 표 2개 + `CharConfig` 행 추가 + `PlayerRoster` 뽑기·교환·배너·시뮬레이션·저장 키 | `Data/CharGachaTable.*`(신규), `Data/CharBannerTable.*`(신규), `Data/CharConfig.csv`, `Character/PlayerRoster.mlua` | 7절 T1~T11 |
| **3-C** | 뽑기 화면 `ui/CharGachaUI` + `Character/CharGachaUILogic.mlua` + 로비 버튼 전환 | `ui/CharGachaUI`(신규), `Character/CharGachaUILogic.mlua`(신규), `ui/LobbyUI`(Label 글자만), `Lobby/LobbyUILogic.mlua`(한 줄+주석) | T12~T21 |
| **3-D** | 다이아 보상: `PlayerRoster` 지급 함수·1초 확인, 탑 결과 표시, 별 상자 안내 문구 | `Character/PlayerRoster.mlua`, `Tower/TowerRunManager.mlua`(작은 추가), `Tower/ResultPopupLogic.mlua`(작은 추가), `Lobby/RegionSelectLogic.mlua`(작은 추가) | T22~T26 |

순서: 3-A → 3-B → 3-D → 3-C (3-C 테스트가 3-B의 강제 결과 함수를 씀). 단위마다 메이커 새로고침·빌드 로그 확인, 마지막에 7절 전체를 한 번의 플레이에서 확인한다.

**건드리지 않는 파일**: `Engrave/PlayerEngrave.mlua`, `Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md`(다른 세션 작업 중), `Skill/GachaUILogic.mlua`·`ui/GachaUI`(6단계에서 삭제), `ui/GrowthUI`·`Lobby/GrowthUILogic.mlua`.

---

## 2. 데이터 표

모두 `RootDesk/MyDesk/Data/`, **UTF-8 BOM** CSV + 같은 이름 `.userdataset`(`RelicTable.userdataset` 복사, 새 uuid, `name`·`id` = 파일 이름). 값은 문자열로 오므로 `tonumber`.

### 2-1. `CharGachaTable.csv` — 결과 종류와 확률

열: `Key,Kind,Rate,Grade,Name,ShortName,ColorHex,#Desc`

```csv
Key,Kind,Rate,Grade,Name,ShortName,ColorHex,#Desc
BookS,S,45,1,경험의 서 (소),소,#5C6675,경험치 100
BookM,M,30,2,경험의 서 (중),중,#2673BF,경험치 500
BookL,L,12,3,경험의 서 (대),대,#7347B3,경험치 2000
JobCard,J,12,4,직업 전용 전직 카드,전용,#D18014,직업별로 나눔. 픽업 중이면 PickupShare%를 픽업 직업에
CommonCard,C,1,5,공용 전직 카드,공용,#CC2E4D,가장 희귀. 전용 카드 대신 쓰거나 해금
```

- `Kind`는 코드가 쓰는 고정 문자: `S / M / L / J / C`. `Rate`는 %(소수 허용). 합계가 100이 아니어도 코드는 **합계로 나눠 정규화**하고 `log_warning("[VRF] gacha rate sum=..")`를 남긴다.
- `ColorHex`는 결과 칸의 등급색(`Color.FromHexCode`, 기존 `GachaUILogic`이 쓰는 함수). 등급색은 칸 위 가는 띠와 글자에만 쓴다(디자인 기준 3절 "등급색은 필요한 부분에 제한적").

### 2-2. `CharBannerTable.csv` — 기간 픽업

열: `BannerId,PickupJob,Start,End,PickupShare,Title,#Desc`

```csv
BannerId,PickupJob,Start,End,PickupShare,Title,#Desc
B001,Bowmaster,2026-11-22 00:00,2026-12-06 00:00,50,보우마스터 픽업,한국 시간. 전용 카드 12% 중 50%(6%)를 픽업 직업에
B002,Viper,2026-12-06 00:00,2026-12-20 00:00,50,바이퍼 픽업,한국 시간
```

- `Start`·`End`는 **한국 시간** `YYYY-MM-DD HH:MM` 고정 형식. `Start ≤ 지금 < End`인 첫 행(표 순서)이 진행 중 픽업. 없으면 상시 통합.
- `PickupJob`은 `CharJobTable.JobId`. 표에 없는 직업이면 그 행은 무시하고 `log_warning`.
- 새 픽업은 행만 추가하면 된다(코드 수정 없음).

### 2-3. `CharConfig.csv`에 행 추가

```csv
GachaCost,100,캐릭터 뽑기 1회 다이아
GachaTenCost,1000,캐릭터 뽑기 10회 다이아
GachaPointPerDraw,1,뽑기 1회당 교환 포인트
ExchangeCost,30,교환 포인트로 직업 전용 카드 1장
MaxedCardPoints,10,5차를 마친 직업의 전용 카드가 바뀌는 교환 포인트
KstOffsetHours,9,픽업 기간 표기(한국 시간)와 UTC의 차이(시간)
FirstClearDiaBase,20,층 첫 클리어 다이아 기본
FirstClearDiaPerFloor,2,층 첫 클리어 다이아 층당 증가 (20 + 2 × 층)
SectionLastDia,300,별 구간 마지막(3번째) 상자 다이아
SectionLastTier,3,다이아를 주는 별 구간 상자 단계
```

---

## 3. 서버: `Character/PlayerRoster.mlua`에 추가

### 3-1. 속성

```
-- 캐릭터 뽑기 (3단계)
@Sync property integer GachaSeq = 0        -- 뽑기·교환 처리마다 +1 (뽑기 화면 갱신 신호)
@Sync property string GachaMsg = ""        -- 마지막 결과 코드 (3-6)
@Sync property string GachaResult = ""     -- 마지막 결과 상세 (3-5)
@Sync property string GachaBanner = ""     -- 진행 중 픽업 BannerId ("" = 상시 통합)
@Sync property integer GachaTotal = 0      -- 누적 뽑기 횟수 (저장 gt)
@HideFromInspector property number BannerCheckTimer = 0
```
- `_T`에 추가: `_T.gachaRows`(표 2-1), `_T.banners`(표 2-2, 파싱된 시각 포함), `_T.rewardFloor`(rf), `_T.rewardSections`(rs, `{ [구간]=true }`), `_T.forceBanner`(테스트, nil = 자동), `_T.forceNext`(테스트 강제 결과 목록).

### 3-2. 표 읽기 (`LoadTables` 끝에 추가, 실행 공간 지정 없음 — 클라이언트 확률 표시도 사용)

- `CharGachaTable` → `_T.gachaRows = { { key, kind, rate, grade, name, short, color }, ... }`(표 순서).
- `CharBannerTable` → `_T.banners = { { id, job, startText, endText, share, title }, ... }`. 시각 계산은 서버 전용 함수에서만 한다(아래).
- 로그를 `[VRF] roster tables jobs=5 outfits=143 gacha=5 banners=2`로 바꾼다.

시각 함수 (`@ExecSpace("ServerOnly")`):

```lua
method number KstToUtcElapsed(string text)
	-- "YYYY-MM-DD HH:MM"(한국 시간) → UtcNow.Elapsed와 같은 단위의 값. 형식이 틀리면 -1.
	local y = tonumber(string.sub(text, 1, 4)); local mo = tonumber(string.sub(text, 6, 7)); local d = tonumber(string.sub(text, 9, 10))
	local h = tonumber(string.sub(text, 12, 13)); local mi = tonumber(string.sub(text, 15, 16))
	if y == nil or mo == nil or d == nil or h == nil or mi == nil then return -1 end
	local day = DateTime(2000, 1, 2, 0, 0, 0).Elapsed - DateTime(2000, 1, 1, 0, 0, 0).Elapsed   -- 단위 가정 없이 하루 길이
	local hour = day / 24
	return DateTime(y, mo, d, h, mi, 0).Elapsed - self:Cfg("KstOffsetHours", 9) * hour
end

method string ActiveBannerId()
	-- 진행 중 픽업 Id ("" = 없음). 테스트 강제값이 있으면 그것("none"이면 "").
	if self._T.forceBanner ~= nil then if self._T.forceBanner == "none" then return "" end return self._T.forceBanner end
	local now = DateTime.UtcNow.Elapsed
	for _, b in ipairs(self._T.banners or {}) do
		local s = self:KstToUtcElapsed(b.startText); local e = self:KstToUtcElapsed(b.endText)
		if s >= 0 and e > s and now >= s and now < e and self._T.jobs[b.job] ~= nil then return b.id end
	end
	return ""
end
```
- `OnUpdate`: 1초 타이머(`PartyCheckTimer` 블록)에서 `local id = self:ActiveBannerId(); if id ~= self.GachaBanner then self.GachaBanner = id; log("[VRF] gacha banner " .. id) end`. `OnBeginPlay`의 `LoadRoster` 뒤에도 한 번.
- 시각이 시작 전이면 상시, 끝나면 다음 행 또는 상시로 자동 전환된다.

### 3-3. 확률 계산 (실행 공간 지정 없음)

```lua
method table GachaRates(string bannerId)
	-- 펼친 확률표: { { kind="S", job="", rate=45, grade=1, name=..., color=... }, ..., { kind="J", job="Hero", rate=2.4, ... }, ... }
	-- J 행은 직업별로 펼친다: 픽업이 없으면 표의 모든 직업 균등, 있으면 픽업 직업 = J × share/100, 나머지 = J × (1 - share/100) / (직업 수 - 1).
	-- rate 합계로 정규화해 합 100이 되게 한다.
method table BannerInfo(string bannerId)
	-- { id, job, title, startText, endText, share } 또는 nil
```
- 직업 목록은 `_T.jobList`(CharJobTable 순서, 잠긴 직업 포함). 직업 수가 1이면 픽업 = 전부.

### 3-4. 뽑기 한 번 굴리기 (서버 전용, 상태 변경 없음)

```lua
@ExecSpace("ServerOnly")
method string RollKind(table rates, boolean cardOnly)
	-- rates(GachaRates 결과)에서 하나를 고른다. cardOnly면 J·C 행만 대상. 반환 "S" / "M" / "L" / "C" / "J.Hero"
	-- r = _UtilLogic:RandomDouble() * 합계 → 누적 비교 (기존 PlayerEngrave 뽑기와 같은 난수 함수)
```

### 3-5. 뽑기 `DoGachaDraw(integer count)` (`ServerOnly`, 결과 코드 반환)

1. `RosterLoaded` 아니면 `err_notready`. `count ~= 1 and count ~= 10` → `err_count`.
2. `cost = (count == 10) and GachaTenCost or GachaCost`(CharConfig). `Diamonds < cost` → `err_dia`(아무것도 바꾸지 않음).
3. `rates = GachaRates(self.GachaBanner)`.
4. 결과 목록 만들기: `_T.forceNext`가 있으면 그 목록 앞에서 `count`개를 꺼내 쓰고(테스트), 없으면 `RollKind(rates, false)`를 `count`번.
5. **10회 보장**: `count == 10`이고 목록에 `C`나 `J.`로 시작하는 것이 없으면 `list[10] = RollKind(rates, true)`, 로그 `[VRF] gacha guarantee`. (강제 목록일 때도 같은 규칙을 적용한다)
6. `Diamonds -= cost`.
7. 목록을 순서대로 적용하고 **표시용 토큰**을 만든다:
   - `S/M/L` → `BookS/M/L += 1`, 토큰 그대로.
   - `C` → `CommonCards += 1`, 토큰 `C`.
   - `J.<job>` → `ApplyJobCard(job)` 결과에 따라 토큰 `U.<job>`(이번 카드로 해금) / `J.<job>`(적립) / `X.<job>`(5차 완료 직업 → `ExchangePoints += MaxedCardPoints`).
8. `ExchangePoints += count × GachaPointPerDraw`, `GachaTotal += count`.
9. `SaveRoster()` **한 번만**(기존 `Give*` 함수는 부를 때마다 저장하므로 쓰지 않는다).
10. `GachaReply("ok_draw", "draw:" .. count .. ":" .. table.concat(tokens, ","))`. 로그 `[VRF] gacha draw x10 banner=B001 dia=.. pt=.. tokens=S,M,J.Hero,...`.

`ApplyJobCard(string jobId) → string` (`ServerOnly`, 저장하지 않음):
- 잠김 → `self._T.chars[jobId] = self:NewChar(true)`, 반환 `"U"`.
- 해금 + `tier >= MaxTier()` → `ExchangePoints += MaxedCardPoints`, 반환 `"X"`.
- 그 외 → `self._T.cards[jobId] += 1`, 반환 `"J"`.
- 1단계 `GiveJobCard`는 그대로 둔다(다른 곳에서 쓸 수 있음).

결과 문자열 형식 (`GachaResult`):

| 동작 | 형식 | 예 |
|---|---|---|
| 뽑기 성공 | `draw:<횟수>:<토큰>,<토큰>,...` | `draw:10:S,M,S,L,J.Hero,S,U.Viper,M,S,C` |
| 교환 성공 | `exchange:<직업>:<U 또는 J>` | `exchange:Bowmaster:U` |
| 실패 | `<동작>:<인자>` | `draw:10`, `exchange:Hero` |

### 3-6. 결과 코드 (`GachaMsg`)

| 코드 | 의미 | UI 문구 (3-C) |
|---|---|---|
| `ok_draw` | 뽑기 성공 | (결과 칸 연출) |
| `ok_exchange` | 교환 성공 | `{4차 이름} 전용 카드를 받았습니다` / `U`면 `{4차 이름} 해금!` |
| `err_notready` | 불러오기 전 | `캐릭터 정보를 불러오는 중입니다` |
| `err_lobby` | 로비가 아님 / 본인 요청 아님 | `로비에서만 할 수 있습니다` |
| `err_count` | 횟수가 1·10이 아님 | `잘못된 요청입니다` |
| `err_dia` | 다이아 부족 | `다이아가 부족합니다` |
| `err_points` | 교환 포인트 부족 | `교환 포인트가 부족합니다` |
| `err_job` | 없는 직업 | `알 수 없는 직업입니다` |
| `err_max` | 5차를 마친 직업은 교환 불가 | `이미 5차를 마친 직업입니다` |

`GachaReply(code, result)` (`ServerOnly`): `GachaSeq += 1`, `GachaMsg = code`, `GachaResult = result`, 로그 `[VRF] gacha reply <code> <result>`, 코드 반환. (`Reply`/`RosterSeq`는 쓰지 않는다 — 캐릭터 화면이 뽑기 결과를 자기 결과로 오해하지 않게. 대신 `CharData`·`JobCards`가 바뀌므로 캐릭터 화면은 `Signature`로 자동 갱신된다)

### 3-7. 교환 `DoExchange(string jobId)` (`ServerOnly`)

1. `err_notready` / 표에 없는 직업 `err_job`.
2. 해금 + `tier >= MaxTier()` → `err_max`.
3. `ExchangePoints < ExchangeCost` → `err_points`.
4. `ExchangePoints -= ExchangeCost`, `t = ApplyJobCard(jobId)`(여기서는 `U` 또는 `J`만 나온다), `SaveRoster()`.
5. `GachaReply("ok_exchange", "exchange:" .. jobId .. ":" .. t)`, 로그 `[VRF] gacha exchange job=.. result=.. pt=..`.

### 3-8. 클라이언트 요청 (`@ExecSpace("Server")`)

```lua
method void RequestGachaDraw(integer count)
	-- 클라이언트 요청: 캐릭터 뽑기 1회·10회.
	if not self:CanEditRoster(senderUserId) then self:GachaReply("err_lobby", "draw:" .. tostring(count)) return end
	self:DoGachaDraw(count)
end

method void RequestExchange(string jobId)
	-- 클라이언트 요청: 교환 포인트로 직업 전용 카드.
	if not self:CanEditRoster(senderUserId) then self:GachaReply("err_lobby", "exchange:" .. jobId) return end
	self:DoExchange(jobId)
end
```
- `CanEditRoster`(본인 + 불러오기 완료 + 로비)는 1단계 함수 그대로 = `err_lobby` 규칙 동일.

### 3-9. 저장 형식 확장 (`OrbisRosterV1`)

```
v=1;dia=..;book=..;cc=..;pt=12;awk=..;jc=..;ch=..;gift=1;gt=42;rf=17;rs=1,2
```

| 키 | 값 | 설명 |
|---|---|---|
| `pt` | 정수 | 교환 포인트 (1단계 칸 그대로) |
| `gt` | 정수 | 누적 뽑기 횟수 (`GachaTotal`) |
| `rf` | 정수 | 첫 클리어 다이아를 이미 받은 최고 층 (0 = 아직 없음) |
| `rs` | `구간,구간,...` (빈 값 허용) | 마지막 별 상자 다이아를 이미 받은 구간 번호 |

- **`Decode`의 아는 키 목록**(`k == "v" or ... or k == "gift"`)에 `gt`, `rf`, `rs`를 추가한다. 추가하지 않으면 `extra`로도 저장되고 `Encode`가 직접 쓴 값과 **두 번 기록**된다.
- `Encode`: `gift` 뒤, `extra` 앞에 `"gt=" .. GachaTotal`, `"rf=" .. rewardFloor`, `"rs=" .. 정렬된 구간 목록 concat(",")`.
- `ApplyFirstGift` / `LoadRoster`(데이터 없음): `GachaTotal = 0`, `_T.rewardFloor = 0`, `_T.rewardSections = {}`. **`DebugReset`은 `rf`·`rs`를 유지**한다(테스트 중 다이아가 다시 소급 지급되지 않게). `DebugReset` 안에서 `ApplyFirstGift` 전에 두 값을 보관했다가 되돌린다.
- 1단계 이전 저장(키 없음)은 0·빈 목록으로 읽는다 → G16 소급 지급.

### 3-10. 테스트 전용 함수 (`ServerOnly`)

| 함수 | 동작 |
|---|---|
| `DebugGachaSim(integer n, string bannerId)` | 상태를 바꾸지 않고 `RollKind`로 n회 굴려 종류·직업별 비율을 로그. 이어서 `n/10`번의 10회 묶음을 3-5의 4~5단계와 같은 규칙으로 만들어 **보장 위반 수(카드 0장인 묶음)**, 묶음당 평균 카드 수를 로그. 로그: `[VRF] gacha sim n=10000 banner=B001 S=45.1 M=29.8 L=12.2 J=11.9 C=1.0 pick=6.0 other=1.5,1.5,1.5,1.5 ten=1000 violations=0 cardsPerTen=1.55` |
| `DebugSetBanner(string id)` | `_T.forceBanner = id`(`"none"` = 강제 상시, `""` = 자동으로 되돌림 → nil), 즉시 `GachaBanner` 갱신 |
| `DebugForceNext(string kinds)` | 다음 뽑기 결과를 고정: `"J.Hero,J.Viper,S"`처럼 3-4 반환 형식을 쉼표로. 쓴 만큼 앞에서 소비 |
| `DebugSetPoints(integer n)` | 교환 포인트 설정 후 저장 |
| `DebugSetRewardMarks(integer floor, string sections)` | `rf = floor`, `rs = sections`("1,2" 또는 "") 설정 후 저장 |
| `DebugCheckSection(string sectionData)` | 3-11의 별 상자 확인을 `eng.SectionData` 대신 주어진 문자열("구간:비트,...")로 실행하고 지급액 반환 |

`DebugDump`은 그대로(`Encode()`에 새 키가 들어감).

### 3-11. 다이아 보상 (3-D)

```lua
@ExecSpace("ServerOnly")
method integer FirstClearDia(integer floor)
	-- 층 첫 클리어 다이아 = FirstClearDiaBase + FirstClearDiaPerFloor × 층.
	return math.floor(self:Cfg("FirstClearDiaBase", 20) + self:Cfg("FirstClearDiaPerFloor", 2) * floor)

@ExecSpace("ServerOnly")
method integer GrantFirstClearDiamonds(integer clearedFloor)
	-- rf < clearedFloor 이면 (rf+1 ~ clearedFloor) 층 몫을 한꺼번에 지급하고 rf = clearedFloor. 지급액 반환(없으면 0). 몇 번 불러도 같은 층은 한 번만.
	-- 지급 시 Diamonds 증가 + rf 갱신 → SaveRoster() 한 번 (같은 저장 문자열에 함께 기록되므로 중복 지급 불가)
	-- 로그: [VRF] dia first clear floors a-b +N total=..

@ExecSpace("ServerOnly")
method integer GrantSectionDiamonds(string sectionData)
	-- "구간:비트,..."에서 비트 (1 << (SectionLastTier-1)) = 4 가 켜진 구간 중 rs에 없는 구간마다 SectionLastDia 지급, rs에 추가. 지급액 반환.
	-- 로그: [VRF] dia section s=3 +300 total=..

@ExecSpace("ServerOnly")
method void CheckDiamondRewards()
	-- 1초마다(OnUpdate 타이머 블록, 맵 상관없이): PlayerEngrave를 읽기만 한다.
	local eng = self.Entity:GetComponent("script.PlayerEngrave")
	if eng == nil then return end
	if eng.Loaded then self:GrantFirstClearDiamonds(eng.ClearedFloor) end
	if eng.StarLoaded then self:GrantSectionDiamonds(eng.SectionData) end
```

- `OnUpdate`의 1초 블록에 `self:CheckDiamondRewards()` 추가(`FixParty`·`RefreshLobbyOutfit`처럼 로비 검사 없음 — 탑에서도 동작). `RosterLoaded` 전에는 `OnUpdate`가 일찍 끝나므로 지급하지 않는다.
- 첫 클리어 판정 근거: `PlayerEngrave.ReportCleared(floor)`는 `floor > ClearedFloor`일 때만 `ClearedFloor`를 올리고 저장한다(읽어서 확인). 즉 `ClearedFloor` = 지금까지 처음 깬 최고 층. 층은 순서대로만 열리므로(`TowerRunManager` 2849행 `unlocked = ClearedFloor/FloorsPerStage + 1`) "최고 층까지 모두 첫 클리어"로 봐도 된다.
- 별 상자 판정 근거: `PlayerEngrave.RequestClaimSection`이 상자를 주면 `SectionData`의 그 구간 비트(1·2·4 = 1·2·3번째 상자)를 켠다. 3번째 상자 = 별 30개(구간 10층 × 3별 전부).
- **주의**: 10번째 구간(91~97층, 7층 = 별 최대 21개)은 3번째 상자(30개)를 받을 수 없다. 기존 `PlayerEngrave` 규칙 문제라 3단계에서는 고치지 않고 6-D 밸런스 항목으로 넘긴다.

**탑 결과 표시 (`Tower/TowerRunManager.mlua`, 작은 추가)**

- 속성 추가(`RunBooks` 근처): `@Sync property integer RunDiamonds = 0` + 주석 `-- 이번 판에 받은 다이아 (층 첫 클리어, 결과 팝업 표시용)`.
- 판 시작 초기화(2842행 부근 `self.RunBooks = 0` 다음): `self.RunDiamonds = 0`.
- 층 클리어(5376행 부근) `if engrave ~= nil and engrave:ReportCleared(self.Floor - 1) then ... end` **블록 바로 뒤**에:
  ```lua
  local roster = isvalid(self._T.player) and self._T.player:GetComponent("script.PlayerRoster") or nil
  if roster ~= nil and engrave ~= nil then
  	local dia = roster:GrantFirstClearDiamonds(engrave.ClearedFloor)
  	if dia > 0 then self.RunDiamonds = self.RunDiamonds + dia end
  end
  ```
  (같은 프레임이라 1초 확인보다 먼저 지급되고, 1초 확인은 0을 돌려준다)
- `Tower/ResultPopupLogic.mlua`: 속성 `property string IconDiamond = "81462aeb825b4660991cef05651271cf"` 추가, 전투 보상 목록 맨 앞에 `if mgr.RunDiamonds > 0 then table.insert(items, { icon = self.IconDiamond, name = "다이아", n = mgr.RunDiamonds }) end`.

**별 상자 안내 (작은 추가, `PlayerEngrave` 응답 문구는 그대로)**

- `Lobby/RegionSelectLogic.mlua` `Claim(tier)`: `self._T.claimTier = tier` 저장. 응답 처리(`eng.ClaimSeq ~= self._T.claimSeq`)에서 `msg = eng.ClaimMsg`; `self._T.claimTier == 3`이고 `string.sub(msg, 1, string.len("보상 획득")) == "보상 획득"`이면 `msg = msg .. " · 다이아 +300"`(300은 `roster:Cfg("SectionLastDia", 300)`) 후 `Hint(msg)`.
- `Tower/ResultPopupLogic.mlua`의 같은 수령 흐름(265행 요청, 367행 응답)에도 같은 처리.
- 실제 지급은 1초 안에 `CheckDiamondRewards`가 한다.

---

## 4. 뽑기 화면 `ui/CharGachaUI` + `Character/CharGachaUILogic.mlua` (3-C)

### 4-1. 공통 규칙

- 1920×1080, `pos`는 부모 중심 기준 anchoredPosition(앵커·피벗 0.5), 따로 적은 것만 예외. 크기 px.
- 빌더: `.claude/skills/msw-ui-system/scripts/msw_ui_builder.cjs`, 경로는 node 안에서 `'ui/'+'CharGachaUI'+'.'+'ui'`. 만들기 전 `Docs/UI-Design-System.md`, `builder-protocol.md` 핵심 + `builder-protocol-ui.md` §3를 읽는다.
- 그룹: `new UIBuilder("CharGachaUI", 10, true)`, `Root` **enable false**.
- 골조(`Bg`·`Bg/HeadBg`·`Frame`·`Title`·`Close`·`HeadLine`)는 **`ui/CharacterUI`에서 UIBuilder로 읽어 컴포넌트 값을 그대로 복사**한다(특히 `Frame` 렌더러 LocalPosition (0,-85.5)·LocalScale (0.9916667, 0.8287037)). `Bg/HeadBg`는 2단계 보완 (d)와 같은 값.
- 버튼: 빌더로 만든 뒤 `ButtonComponent`를 `Transition: 2` + `ImageRUIDs`(Highlighted/Pressed/Selected/Disabled) 패치. 상태 이미지는 `Phase-2-Character-UI.md` 2-2 표(card/button/action/close) 그대로. 배경 `PreserveSprite 0`, 패널·버튼 Sliced(경계 12), close Simple. 버튼 글자는 버튼 자체 텍스트, 가로 Center·세로 Middle, Padding 0.
- 런타임: `_UIButtonLogic:UseTheme(entity, role)`(역할 `button / card / action / close`), 비활성 `_UIButtonLogic:SetDisabled`, 선택 `_UIButtonLogic:SetSelected`. 클릭 핸들러는 `CharacterUILogic.Bind`처럼 `entity.EnabledInHierarchy`와 `ButtonComponent.Enable`을 검사해 **비활성 버튼 입력을 실제로 막는다**. 클릭음 `_UISoundLogic:PlayClick()`.
- 글자색: 일반 `(0.14,0.17,0.21)`, 보조 `(0.36,0.40,0.46)`, 유리 `(0.20,0.48,0.12)`, 부족 `(0.78,0.20,0.20)`, 어두운 면 위 흰색. 등급색은 `CharGachaTable.ColorHex`.
- 아이콘: 직업 아이콘 = `CharSlotTable`의 `Main` 칸 `T4` Id → `HeroSkillTable.IconRUID`(2단계 `CharacterUILogic.LoadJobs`와 같은 방법). 경험의 서·전직 카드 전용 아이콘은 프로젝트에 없으므로 **글자**로 표시한다(빠진 리소스, 2단계와 같음). 다이아 아이콘 `81462aeb825b4660991cef05651271cf`. 아이콘 렌더러는 `PreserveSprite 0`(None), 피벗 (0.5,0.5), RaycastTarget false.

### 4-2. 엔티티 표

`Root` 아래

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Root` | panel(빈) | 0,0 | 1920×1080 | — | enable false |
| `Root/Bg` | sprite | 0,0 | 1920×1080 | panel_outer `ad1157dddb334c91b1b58c1073e060e1`, Sliced | CharacterUI 복사 |
| `Root/Bg/HeadBg` | sprite | 0,451 | 1920×178 | panel_outer, Sliced, RaycastTarget false | 띠 비침 방지 |
| `Root/Frame` | sprite | 0,0 | 1920×1080 | panel_light `951c44d6b73c4d46929c47a479cfb4aa`, Sliced | CharacterUI 복사(렌더러 위치·배율 포함) |
| `Root/Title` | text | 앵커·피벗 (0,1), 28,-24 | 360×60 | `캐릭터 뽑기`, 40, 흰색, 왼쪽 | |
| `Root/Close` | button(close) | 앵커·피벗 (1,1), -20,-20 | 64×64 | close 4상태, Simple, 글자 없음 | |
| `Root/HeadLine` | sprite | 0,360.5 | 1904×3 | `e64a2ce6121a4dbf9d377aa8204cf00c`, Color (0.15,0.54,0.81,1), Simple | |
| `Root/DiaIcon` | sprite | 520,400 | 56×56 | 다이아 `81462aeb825b4660991cef05651271cf` | |
| `Root/DiaText` | text | 680,400 | 260×48 | `3,000`, 30, 흰색, 왼쪽 | 보유 다이아 |
| `Root/Banner` | sprite | -600,-90 | 640×860 | panel_inner `bea05f1ee4ea484eb10471a620c69046`, Sliced | 픽업·교환 영역 |
| `Root/Result` | sprite | 330,-90 | 1180×860 | panel_inner, Sliced | 결과·뽑기 버튼 영역 |
| `Root/Modal` | panel(빈) | 0,0 | 1920×1080 | — | enable false. 확률·교환 창 |

`Root/Banner` 아래 (가로 -320~320, 세로 -430~430)

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Tag` | text | 0,380 | 600×36 | `기간 픽업`, 24, 보조색, 가운데 | 상시: `상시 뽑기` |
| `Title` | text | 0,330 | 600×56 | `보우마스터 픽업`, 40, 일반색, 가운데 | 표 `Title`. 상시: `통합 뽑기` |
| `Period` | text | 0,280 | 600×32 | `11월 22일 00:00 ~ 12월 6일 00:00 (한국 시간)`, 22, 보조색, 가운데, BestFit 최소 16 | 상시: `언제나 뽑을 수 있습니다` |
| `IconBox` | sprite | 0,130 | 220×220 | card_normal `a88cfe62e8e64c39bf14d682a77ef4a0`, Sliced | 상시: enable false |
| `IconBox/Icon` | sprite | 0,0 | 160×160 | 픽업 직업 아이콘 | |
| `JobName` | text | 0,-5 | 600×44 | `보우마스터 · 궁수`, 30, 일반색, 가운데 | `Name4 · JobGroup`. 상시: 빈칸 |
| `Desc` | text | 0,-95 | 580×110 | 3줄, 22, 보조색, 가운데 (4-4) | |
| `PointBox` | sprite | 0,-245 | 580×110 | card_normal, Sliced | |
| `PointBox/Text` | text | 0,22 | 540×36 | `교환 포인트 12 / 30`, 26, 일반색, 가운데 | 30 이상이면 유리색 |
| `PointBox/Sub` | text | 0,-22 | 540×30 | `30점으로 원하는 직업 전용 카드 1장`, 20, 보조색, 가운데 | |
| `BtnRates` | button(button) | -150,-365 | 260×88 | `확률 보기`, 26 | |
| `BtnExchange` | button(action) | 150,-365 | 260×88 | `카드 교환`, 26, 흰 글자 | 포인트 < 30이면 비활성 |

`Root/Result` 아래 (가로 -590~590, 세로 -430~430)

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Title` | text | 0,385 | 1120×40 | `뽑기 결과`, 26, 일반색, 왼쪽 | |
| `Empty` | text | 0,60 | 1000×44 | `다이아로 캐릭터를 뽑아 보세요`, 26, 보조색, 가운데 | 결과가 있으면 숨김 |
| `Cell1`~`Cell10` | sprite | x = -440, -220, 0, 220, 440 / y = 200(1~5), -80(6~10) | 200×250 | card_normal, Sliced | 처음 enable false |
| `Cell{n}/Bar` | sprite | 0,113 | 176×8 | `e64a2ce6121a4dbf9d377aa8204cf00c`, Simple, Color = 등급색 | |
| `Cell{n}/Icon` | sprite | 0,40 | 112×112 | 직업 아이콘 | 전용 카드만 표시 |
| `Cell{n}/Label` | text | 0,40 | 180×112 | `경험의 서\n(소)`, 30, 등급색, 가운데, 2줄 | 아이콘이 없는 결과(서·공용)만 표시 |
| `Cell{n}/Name` | text | 0,-55 | 184×56 | 22, 일반색, 가운데, 2줄, BestFit 최소 16 | 4-4 |
| `Cell{n}/Tag` | text | 0,-100 | 184×30 | 22, 가운데 | `해금!` 유리색 / `+10 포인트` 파랑 (0.15,0.45,0.75) / 빈칸 |
| `Msg` | text | 0,-280 | 1120×32 | 결과 문구, 22, 가운데 | 2.5초 뒤 숨김 |
| `BtnOne` | button(action) | -230,-365 | 400×96 | `1회 뽑기 · 다이아 100`, 28, 흰 글자 | 다이아 < 100 비활성 |
| `BtnTen` | button(action) | 230,-365 | 400×96 | `10회 뽑기 · 다이아 1,000`, 28, 흰 글자 | 다이아 < 1,000 비활성 |

- 1회 결과는 `Cell1`만 켜고 `anchoredPosition`을 (0, 60)으로 옮긴다. 10회 결과를 그리기 전 `Cell1`을 원래 (-440, 200)으로 되돌린다.

`Root/Modal` 아래 (모달 창, 마지막 자식이라 맨 위에 그려짐)

| 경로 | 종류 | pos | 크기 | 리소스·글자 | 비고 |
|---|---|---|---|---|---|
| `Dim` | sprite | 0,0 | 1920×1080 | panel_outer, Sliced, Color (1,1,1,0.6), **RaycastTarget true** | 뒤 버튼 입력 차단 |
| `Rate` | sprite | 0,0 | 960×760 | panel_outer, Sliced | 확률 창, enable false |
| `Rate/Title` | text | 앵커·피벗 (0,1), 28,-20 | 600×56 | `확률 정보`, 34, 흰색, 왼쪽 | |
| `Rate/Close` | button(close) | 앵커·피벗 (1,1), -16,-16 | 64×64 | close 4상태 | |
| `Rate/Body` | sprite | 0,-40 | 912×632 | panel_light, Sliced | |
| `Rate/Body/Text` | text | 0,0 | 852×580 | 4-4 확률 문구, 22, 일반색, 왼쪽·위(Left 1 / Top 256), BestFit 최소 16 | |
| `Exchange` | sprite | 0,0 | 1200×680 | panel_outer, Sliced | 교환 창, enable false |
| `Exchange/Title` | text | 앵커·피벗 (0,1), 28,-20 | 700×56 | `직업 카드 교환`, 34, 흰색, 왼쪽 | |
| `Exchange/Close` | button(close) | 앵커·피벗 (1,1), -16,-16 | 64×64 | close 4상태 | |
| `Exchange/Body` | sprite | 0,-36 | 1152×560 | panel_light, Sliced | |
| `Exchange/Body/Job1`~`Job5` | button(card) | x = -460, -230, 0, 230, 460 / y 80 | 200×240 | card 7상태, 글자 없음 | 선택 = Selected 유지 |
| `Exchange/Body/Job{n}/Icon` | sprite | 0,30 | 112×112 | 직업 아이콘 | |
| `Exchange/Body/Job{n}/Name` | text | 0,-62 | 190×36 | `보우마스터`, 24, 일반색, 가운데 | `Name4` |
| `Exchange/Body/Job{n}/Sub` | text | 0,-96 | 190×28 | `잠김 → 해금` / `전용 카드 2장` / `5차 완료`, 20, 보조색, 가운데 | |
| `Exchange/Body/Info` | text | 0,-110 | 1080×60 | 22, 일반색, 가운데, 2줄 | 4-4 |
| `Exchange/Body/BtnOk` | button(action) | 0,-210 | 360×88 | `교환 (포인트 30)`, 26, 흰 글자 | 4-3 활성 조건 |

모든 텍스트는 해당 배경 안에 들어가는 크기다(테두리 12px 안쪽). 긴 문구(`Period`, `Name`, `Rate/Body/Text`)는 BestFit.

### 4-3. 동작 (`CharGachaUILogic`, `@Logic`, 모든 메서드 `@ExecSpace("ClientOnly")`)

- 속성: `RootPath = "/ui/CharGachaUI/Root"`, `LobbyMapName = "Lobby"`, `RevealInterval = 0.12`, `SingleDelay = 0.3`, `BusyTimeout = 5`, `MsgDuration = 2.5`, 색 4종(2단계와 같은 값).
- `OnBeginPlay`: 버튼 연결·테마(`BtnOne/BtnTen/BtnExchange/Exchange/Body/BtnOk` = action, `BtnRates` = button, `Exchange/Body/Job{n}` = card, 닫기 3개 = close), `Root` 닫기, 직업 목록·아이콘 읽기, 로그 `[VRF] CharGachaUILogic begin`.
- `Open()`: `Root` 켜기, `Modal` 끄기, 결과 칸 초기화(`Empty` 표시), `lastSeq = roster.GachaSeq`, `Refresh()`, 로그 `[VRF] chargacha open banner=<GachaBanner>`.
- `Close()`: 끄기, 로그 `[VRF] chargacha close`. 로비를 벗어나면 자동 닫기(`CharacterUILogic.OnUpdate`와 같은 방식).
- `OnUpdate`(열려 있을 때): ① 로비 검사 ② `roster.GachaSeq ~= lastSeq` → `HandleResult()` ③ 공개 연출 진행 ④ 대기 시간 초과(5초) → `응답이 없습니다. 다시 시도해 주세요`, 버튼 복구 ⑤ `Diamonds / ExchangePoints / GachaBanner / CharData / JobCards`를 이은 문자열이 바뀌면 `Refresh()`.
- `Refresh()`: `DiaText`(쉼표), 배너 영역(4-4), `PointBox/Text`, 버튼 활성:
  - `BtnOne`: `Diamonds ≥ GachaCost` and 대기·연출 중 아님. 글자 `1회 뽑기 · 다이아 100`(비용은 `roster:Cfg`).
  - `BtnTen`: `Diamonds ≥ GachaTenCost` and 대기·연출 중 아님.
  - `BtnExchange`: `ExchangePoints ≥ ExchangeCost` and 대기·연출 중 아님.
  - `BtnRates`: 항상 활성.
- 뽑기: `OnDraw(count)` → `roster:RequestGachaDraw(count)`, `pendingAt` 기록, 실행 버튼 전부 비활성, 로그 `[VRF] chargacha request draw <count>`.
- `HandleResult()`: `GachaMsg`/`GachaResult`.
  - `ok_draw`: `f = Split(result, ":")`, 토큰 목록 = `Split(f[3], ",")` → 공개 연출 시작(`Empty` 숨김, 칸 모두 끄기 → 1회는 `SingleDelay` 뒤 `Cell1`, 10회는 `RevealInterval`마다 `Cell1`~`Cell10` 순서대로 켜고 내용 채움, 칸마다 로그 `[VRF] chargacha reveal i=<n> <토큰>`). 공개가 끝나면 `pendingAt = nil`, `Refresh()`, `Msg`에 요약(해금이 있으면 `바이퍼 해금!` 유리색, 없으면 빈칸).
  - `ok_exchange`: 교환 창 닫기, `Msg` = 3-6 문구(유리색).
  - 그 외 `err_*`: `Msg` = 3-6 문구(부족색), 버튼 복구.
  - 로그 `[VRF] chargacha result <코드> <결과>`.
- 확률 보기: `Modal`·`Rate` 켜기, `Exchange` 끄기, 본문 = 4-4. 닫기는 `Rate/Close`(교환 창은 `Exchange/Close`)로만 하고, `Dim` 클릭은 입력만 막고 아무 동작도 하지 않는다. 창을 닫으면 `Modal`도 끈다.
- 교환: `BtnExchange` → `Modal`·`Exchange` 켜기, 선택 = 픽업 직업(없으면 첫 번째 잠긴 직업, 없으면 1번). `Job{n}` 클릭 → 선택(SetSelected), `Info` 갱신. `BtnOk` 활성 조건: `ExchangePoints ≥ 30` and 선택 직업이 5차 완료가 아님 and 대기 중 아님. 클릭 → `roster:RequestExchange(job)`, 로그 `[VRF] chargacha request exchange <job>`.
- 연출 중 닫기(`Close`)는 화면만 닫는다. 서버 지급은 이미 끝났으므로 다시 열면 `Empty` 상태.

### 4-4. 문구

토큰 → 칸 표시

| 토큰 | 칸 | `Label`(아이콘 없음) | `Icon` | `Name` | `Tag` | 등급(색) |
|---|---|---|---|---|---|---|
| `S` / `M` / `L` | 서 | `경험의 서\n(소)` / `(중)` / `(대)` | 끔 | `경험의 서 (소)` 등 | 빈칸 | 1 / 2 / 3 |
| `C` | 공용 카드 | `공용\n카드` | 끔 | `공용 전직 카드` | 빈칸 | 5 |
| `J.<job>` | 전용 카드 | 끔 | 직업 아이콘 | `{Name4}\n전용 카드` | 빈칸 | 4 |
| `U.<job>` | 전용 카드(해금) | 끔 | 직업 아이콘 | `{Name4}\n전용 카드` | `해금!` | 4 |
| `X.<job>` | 5차 완료 전환 | 끔 | 직업 아이콘 | `{Name4}\n전용 카드` | `+10 포인트` | 4 |

배너 영역

| 상태 | `Tag` | `Title` | `Period` | `IconBox` | `JobName` | `Desc` |
|---|---|---|---|---|---|---|
| 픽업 중 | `기간 픽업` | 표 `Title` | `M월 D일 HH:MM ~ M월 D일 HH:MM (한국 시간)` (표 문자열을 잘라 만든다, 시각 계산 없음) | 켬, 픽업 직업 아이콘 | `{Name4} · {JobGroup}` | `픽업 직업 전용 카드 6% (다른 직업 각 1.5%)\n10회 뽑기는 직업 카드 1장 이상 보장\n뽑기 1회마다 교환 포인트 1점` |
| 상시 | `상시 뽑기` | `통합 뽑기` | `언제나 뽑을 수 있습니다` | 끔 | 빈칸 | `직업 전용 카드는 모든 직업이 같은 확률 (각 2.4%)\n10회 뽑기는 직업 카드 1장 이상 보장\n뽑기 1회마다 교환 포인트 1점` |

확률 숫자는 `roster:GachaRates(GachaBanner)`에서 계산한다(하드코딩 금지). 숫자 표기: 소수 첫째 자리까지, `.0`은 뺀다(`6%`, `1.5%`, `2.4%`).

확률 창 본문 (픽업 중 예시, 줄마다 계산값)

```
경험의 서 (소)  45%
경험의 서 (중)  30%
경험의 서 (대)  12%
직업 전용 전직 카드  12%
   · 보우마스터 (픽업)  6%
   · 히어로 · 비숍 · 나이트로드 · 바이퍼  각 1.5%
공용 전직 카드  1%

· 10회 뽑기에 직업 카드(전용·공용)가 하나도 없으면 마지막 칸을 직업 카드로 바꿔 줍니다.
· 처음 얻는 전용 카드는 그 직업을 해금합니다.
· 뽑기 1회마다 교환 포인트 1점. 30점으로 원하는 직업 전용 카드 1장을 받습니다.
· 5차를 마친 직업의 전용 카드는 교환 포인트 10점으로 바뀝니다.
```
상시일 때는 직업 줄을 `   · 모든 직업  각 2.4%` 한 줄로.

교환 창 `Info`: `{Name4} 전용 전직 카드 1장 — 교환 포인트 30 사용` + 둘째 줄 `잠긴 직업은 이 카드로 바로 해금됩니다`(잠긴 직업) / `보유 전용 카드 {n}장`(해금 직업) / `5차를 마친 직업은 교환할 수 없습니다`(부족색).

### 4-5. 로비 버튼 전환 (바꾸는 기존 파일)

- `ui/LobbyUI`: `Root/BtnGacha/Label`의 `Text`만 `캐릭터 뽑기`로(UIBuilder `patchComponent`). 엔티티·UUID·아이콘(`2db220ec552a4416b44aba16503247d9`)·크기(288×96)·글자 영역(176×44) 그대로. `캐릭터 뽑기`(6글자)가 한 줄에 들어가는지 스크린샷 확인, 넘치면 Label에 BestFit(최소 22).
- `Lobby/LobbyUILogic.mlua`: `self:Bind(self.btnGacha, function() _GachaUILogic:Open() end)` → `self:Bind(self.btnGacha, function() _CharGachaUILogic:Open() end)`. `btnGacha` 속성 위 주석을 `-- 캐릭터 뽑기 화면 (3단계, 옛 스킬 뽑기 진입은 끊음 — 스킬 뽑기 삭제는 6단계)`로.
- `Skill/GachaUILogic.mlua`, `ui/GachaUI`는 고치지 않는다. 진입점이 없어 열리지 않을 뿐이다(`OnBeginPlay`에서 닫힌 상태로 시작).

---

## 5. 구현 순서와 주의

1. **3-A**: 2단계 보완(Phase-2 문서 끝) → 새로고침 → 확인.
2. **3-B**: CSV 2개 + userdataset 2개 + `CharConfig` 10행 → `PlayerRoster` 속성·표 읽기·`GachaRates`·`RollKind`·`ApplyJobCard`·`DoGachaDraw`·`DoExchange`·`Request*`·저장 키·배너 갱신·테스트 함수 → 새로고침 → T1~T11.
3. **3-D**: `FirstClearDia`·`GrantFirstClearDiamonds`·`GrantSectionDiamonds`·`CheckDiamondRewards` → `TowerRunManager`·`ResultPopupLogic`·`RegionSelectLogic` 작은 추가 → T22~T26.
4. **3-C**: `ui/CharGachaUI`(빌더) + 버튼 상태 패치 → `CharGachaUILogic.mlua` → 로비 버튼 → T12~T21.

주의:
- 메서드 설명 주석은 메서드 본문 첫 줄. `member`는 예약어. 선언 안 한 필드 대입은 조용히 실패하므로 내부 상태는 `self._T`.
- 문자열 치환에 `string.gsub` 금지(LEA-1103) — `string.find` + `sub`.
- `JSONEncode` 쓰지 않는다(빈 표 실패). 결과·저장 모두 문자열 형식.
- 뽑기 한 번에 `SaveRoster()`는 한 번. `GiveBooks/GiveJobCard/GiveCommonCard/GiveExchangePoints`를 반복 호출하지 않는다(호출마다 저장).
- `PlayerEngrave`는 읽기만(`Loaded`, `ClearedFloor`, `StarLoaded`, `SectionData`). 쓰기·함수 호출 없음.
- `Signature`(캐릭터 화면)는 이미 `Diamonds`·`CharData`·`JobCards`를 포함하므로 뽑기 후 캐릭터 화면이 자동으로 맞는다. 고칠 필요 없음.

---

## 6. 밸런스 메모

### 6-1. 뽑기 기대값 (G2·G3·G6·G8 기준)

| 항목 | 1회 | 10회 1번 | 100회(10회 × 10) |
|---|---|---|---|
| 경험치(서) | 435 | 약 4,226 (보장으로 서 1권이 카드로 바뀌는 몫 −124) | 약 42,260 |
| 직업 전용 카드(전체) | 0.12 | 1.429 | 14.3 |
| └ 픽업 직업 | 0.06 | 0.715 | 7.1 |
| └ 다른 직업 각각(픽업 중) | 0.015 | 0.179 | 1.8 |
| 공용 카드 | 0.01 | 0.119 | 1.2 |
| 교환 포인트 | 1 | 10 | 100 (= 카드 3.3장) |

- 10회에 카드 0장 확률 0.87¹⁰ = 24.8% → 보장이 그만큼 카드를 더한다(전용 0.229 + 공용 0.019).
- 1직업 1→5차 완성에 필요한 것: 해금 1장 + 전직 1+2+3+5 = **카드 12장**, 경험치 **46,520**(1단계 D8).
- **픽업 직업을 처음부터 완성**: 100회에 픽업 카드 7.1 + 교환 3.3 + 공용 1.2 ≈ 11.7장 → 카드 기준 약 **103회**, 경험치 기준 약 **110회** → **약 110회 = 11,000 다이아에 새 직업 1명 5차 Lv30**. 1단계 D4의 "픽업 기준 약 150~180회"보다 빠르다(교환 포인트와 공용 카드를 더했기 때문).
- 픽업이 아닌 직업을 상시에서 완성: 100회에 카드 2.9 + 교환 3.3 + 공용 1.2 ≈ 7.4장 → 약 **163회**.

### 6-2. 다이아 수급 (G14)

| 진행 | 첫 클리어 누적 | 별 마지막 상자 누적(모든 별) | 합계 | 뽑기 횟수 |
|---|---|---|---|---|
| 10층 | 310 | 300 | 610 | 6회 |
| 20층 | 820 | 600 | 1,420 | 14회 |
| 30층 | 1,530 | 900 | 2,430 | 24회 |
| 50층 | 3,550 | 1,500 | 5,050 | 50회 |
| 70층 | 6,370 | 2,100 | 8,470 | 84회 |
| 97층(끝) | 11,446 | 2,700 | 14,146 | 141회 |

- 공식: 첫 클리어 누적(F층) = `20F + F(F+1)`. 별 상자는 구간(10층)마다 별 30개 전부일 때만 300(10구간은 불가, 3-11 주의).
- **결론**: 탑을 끝까지(97층, 별 전부) 오르면 약 141회 = **새 직업 약 1.3명 완성**, 또는 시작 3직업을 카드 기준 대부분 5차까지(시작 3직업 5차까지 카드 33장 필요, 141회 + 교환으로 약 26.5장). 30층까지는 24회뿐이라 **초중반 성장이 느리다**.
- 경험의 서는 6단계에서 스테이지 클리어 스킬북 보상이 경험의 서로 바뀌면 크게 늘어나므로, 뽑기의 실질 병목은 **전직 카드**다.
- 3단계 값은 로드맵 제안 그대로 확정한다. 6-D 밸런스에서 검토할 것: ① 반복 수급원(예: 매일 첫 클리어 소량, 별 1·2번째 상자에도 소량 다이아) ② 10구간(91~97층) 3번째 상자 기준 ③ 초반 보정(예: 1~30층 첫 클리어 2배).

### 6-3. 첫 지급량

- 현재(테스트용): 다이아 3,000(30회), 서 50/20/10(35,000 경험치 = 1명 거의 Lv27), 공용 2, 카드 히어로 3·비숍 1.
- 정식으로는 **과하다**: 서만으로 한 직업이 4차 관문 근처까지 가고, 30회 뽑기는 30층 진행분보다 많다.
- **출시 권장값(6-D에서 적용)**: 다이아 1,000(10회 1번 체험), 서 10/4/1(5,000 경험치 ≈ Lv9), 공용 0, 시작 카드 히어로 1·비숍 1·나이트로드 1(첫 전직 체험). 3~5단계 동안은 회귀 테스트 기대값(1단계 T1, 2단계 T2~T4)을 위해 지금 값을 유지한다.

---

## 7. 테스트 케이스

준비 (server_main):
```lua
local p = nil
for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end
local r = p:GetComponent("script.PlayerRoster")
local eng = p:GetComponent("script.PlayerEngrave")
r:DebugReset()
r:DebugSetBanner("B001")
```
클라이언트 버튼: `_EntityService:GetEntityByPath("/ui/CharGachaUI/Root/Result/BtnTen"):SendEvent(ButtonClickEvent())`. 결과는 `[VRF]` 로그·Sync 값(`r:DebugDump()`)으로 확인. 클라이언트 첫 스크립트는 UI 준비 전일 수 있으니 한 번 더 실행.

### 3-B 서버

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T1 표 | 플레이 직후 로그 | `[VRF] roster tables jobs=5 outfits=143 gacha=5 banners=2`, `r:GachaRates("B001")` 합계 100(±0.001)·`J` 행 5개(보우마스터 6, 나머지 1.5), `r:GachaRates("")` `J` 행 각 2.4 — 결과를 `log("[VRF] rates ...")`로 출력 |
| T2 시각·배너 | `log(DateTime.UtcNow:ToFormattedString("yyyy-MM-dd HH:mm"))`, `r:DebugSetBanner("")` 후 2초 뒤 `r.GachaBanner` | 지금이 2026-11-22~12-06(한국 시간)이면 `B001`. `r:KstToUtcElapsed("2026-11-22 09:00")`과 `DateTime(2026,11,22,0,0,0).Elapsed`가 같아야 함(KST 09:00 = UTC 00:00). 이어서 `r:DebugSetBanner("none")` → `""`, `r:DebugSetBanner("B001")` → `B001` |
| T3 확률(픽업) | `r:DebugGachaSim(10000, "B001")` | S 45±1.5, M 30±1.5, L 12±1, J 12±1, C 1±0.3, pick 6±0.8, other 각 1.5±0.4, `violations=0`, `cardsPerTen` 1.55±0.08 |
| T4 확률(상시) | `r:DebugGachaSim(10000, "")` | J 직업별 각 2.4±0.5, 나머지는 T3과 같음, `violations=0` |
| T5 1회 | `r:DebugSetCurrency(1000,0,0,0,0)`, `r:DebugSetPoints(0)` → `r:DoGachaDraw(1)` | `ok_draw`, `GachaResult` = `draw:1:<토큰 1개>`, 다이아 900, 포인트 1, `GachaTotal` +1, 토큰 종류만큼 재화 +1 |
| T6 10회 부족·성공 | 이어서 `r:DoGachaDraw(10)` → `r:DebugSetCurrency(1000,0,0,0,0)` → `r:DoGachaDraw(10)` | 첫 번째 `err_dia`·다이아 900 그대로, 두 번째 `ok_draw`·토큰 10개·다이아 0·포인트 +10, 토큰 중 `C`/`J.`/`U.`/`X.` 하나 이상 |
| T7 보장 | `r:DebugSetCurrency(1000,0,0,0,0)`, `r:DebugForceNext("S,S,S,S,S,S,S,S,S,S")` → `r:DoGachaDraw(10)` | `[VRF] gacha guarantee`, 10번째 토큰이 `C` 또는 `J.`/`U.`/`X.` |
| T8 해금 | 바이퍼 잠긴 상태(`r:DebugSetChar("Viper",0,1,1)`), `r:DebugSetCurrency(1000,0,0,0,0)`, `r:DebugForceNext("J.Viper,J.Viper")`, 1회 2번 | 첫 토큰 `U.Viper`(해금 Lv1 1차, 카드 0), 두 번째 `J.Viper`(카드 1) |
| T9 5차 전환 | `r:DebugSetChar("Hero",1,30,5)`, `r:DebugSetPoints(0)`, `r:DebugForceNext("J.Hero")` → 1회 | 토큰 `X.Hero`, 히어로 카드 변화 없음, 포인트 11(1 + 10) |
| T10 교환 | `r:DebugSetChar("Bowmaster",0,1,1)`, `r:DebugSetPoints(29)` → `r:DoExchange("Bowmaster")` → `r:DebugSetPoints(30)` → `r:DoExchange("Bowmaster")` → `r:DoExchange("Hero")`(5차) → `r:DoExchange("Nope")` | `err_points` → `ok_exchange exchange:Bowmaster:U`(해금, 포인트 0) → `err_max` → `err_job` |
| T11 저장·로비 | `r:DebugDump()` 기록 → stop → play → `r:DebugDump()` | 두 덤프 같음, `pt=`·`gt=`·`rf=`·`rs=` 포함. 탑 맵에서 클라이언트 `RequestGachaDraw(1)` → `[VRF] gacha reply err_lobby draw:1`(어려우면 정적 검사로 대체하고 보고) |

### 3-C 화면

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T12 진입 | 로비 버튼 글자 확인 → 클라이언트 `/ui/LobbyUI/Root/BtnGacha` 클릭 이벤트 | 글자 `캐릭터 뽑기`(한 줄), `CharGachaUI/Root` enable true, `[VRF] chargacha open banner=B001`, `/ui/GachaUI` 쪽 패널은 계속 꺼짐 |
| T13 표시 | 서버 `r:DebugSetCurrency(1000,0,0,0,0)`, `r:DebugSetPoints(12)` | `DiaText` `1,000`, `Banner/Title` `보우마스터 픽업`, `Period` `11월 22일 00:00 ~ 12월 6일 00:00 (한국 시간)`, `PointBox/Text` `교환 포인트 12 / 30`, `Empty` 보임, 스크린샷으로 정렬·글자 영역 확인 |
| T14 다이아 부족 차단 | 서버 `r:DebugSetCurrency(50,0,0,0,0)` → `BtnOne`·`BtnTen` 클릭 이벤트 | 두 버튼 Disabled 이미지, `GachaSeq` 변화 없음(요청 로그 없음) |
| T15 1회만 가능 | 서버 `r:DebugSetCurrency(100,0,0,0,0)` | `BtnOne` 활성, `BtnTen` 비활성 |
| T16 1회 해금 연출 | 바이퍼 잠김, `r:DebugForceNext("J.Viper")` → `BtnOne` 클릭 | `[VRF] chargacha request draw 1` → `chargacha reveal i=1 U.Viper`, `Cell1`만 (0,60)에 표시, 아이콘·`바이퍼\n전용 카드`·`해금!`, `Msg` `바이퍼 해금!`, 다이아 0, 캐릭터 화면을 열면 바이퍼 해금 |
| T17 10회 연출 | `r:DebugSetCurrency(1000,0,0,0,0)`, `r:DebugForceNext("S,M,L,C,J.Hero,S,S,M,S,S")` → `BtnTen` | `reveal i=1`~`i=10` 순서대로(약 0.12초 간격), `Cell1`이 (-440,200)으로 돌아옴, 칸 1 `경험의 서 (소)` 회색 띠·글자, 칸 3 보라, 칸 4 `공용 카드` 진홍, 칸 5 히어로 아이콘. 연출 동안 뽑기 버튼 비활성, 끝나면 복구(다이아 0이라 비활성 유지) |
| T18 확률 창 | `BtnRates` → `Rate/Close` | `Modal`·`Rate` 켜짐, 본문에 `보우마스터 (픽업)  6%`, `각 1.5%`; 닫기 후 `Modal` 꺼짐. 서버 `r:DebugSetBanner("none")` 후 다시 열면 `모든 직업  각 2.4%`, 배너 영역 `상시 뽑기` / `통합 뽑기`, `IconBox` 꺼짐 |
| T19 교환 | 서버 `r:DebugSetBanner("B001")`, 보우마스터 잠김, `r:DebugSetPoints(30)` → `BtnExchange` → `Exchange/Body/Job4` → `BtnOk` | 교환 창에서 Job4 Selected, `Info`에 `잠긴 직업은 이 카드로 바로 해금됩니다`, 결과 `ok_exchange exchange:Bowmaster:U`, 창 닫힘, `Msg` `보우마스터 해금!`, 포인트 0, `BtnExchange` 비활성 |
| T20 교환 차단 | `r:DebugSetPoints(30)`, 히어로 5차 Lv30 → 교환 창에서 `Job1` 선택 → `BtnOk` 클릭 이벤트 | `BtnOk` 비활성, 요청 로그 없음, `Info` `5차를 마친 직업은 교환할 수 없습니다` |
| T21 닫기·로비 이탈 | `Root/Close` / 탑 이동 | `[VRF] chargacha close`, Root 꺼짐 |

### 3-D 다이아

| 번호 | 절차 | 기대 결과 |
|---|---|---|
| T22 소급 첫 클리어 | `F = eng.ClearedFloor`(F ≥ 1이어야 함, 0이면 탑 1층을 한 번 깬다) → `r:DebugSetRewardMarks(0, "")` → 2초 대기 | 다이아 `+ (20F + F(F+1))`, `[VRF] dia first clear floors 1-F +N`, 덤프 `rf=F`. 2초 더 기다려도 추가 지급 없음 |
| T23 함수 중복 방지 | `r:DebugSetRewardMarks(F - 1, "")` → (2초 안에) `r:GrantFirstClearDiamonds(F)` 반환값 → 다시 `r:GrantFirstClearDiamonds(F)` | 첫 반환 `20 + 2F`(또는 1초 확인이 먼저 돌았다면 0이고 다이아는 이미 증가 — 둘 중 하나만), 두 번째 0. 다이아 증가 총량은 `20 + 2F` 한 번 |
| T24 별 상자 | `r:DebugSetRewardMarks(F, "")` → `r:DebugCheckSection("1:7,2:3")` → 다시 같은 호출 | 첫 반환 300(구간 1만, 구간 2는 비트 4 없음), `rs=1`, 두 번째 0 |
| T25 탑 결과 표시 | 아직 안 깬 층(F+1)을 실제로 클리어(가능하면) | 결과 팝업 전투 보상에 다이아 아이콘 `x{20+2(F+1)}`, `[VRF] dia first clear floors (F+1)-(F+1)`. 같은 층 재클리어 시 다이아 항목 없음. 시간이 많이 들면 정적 검사로 대체하고 보고 |
| T26 별 상자 안내 | 정적 검사(가능하면 3번째 상자를 받을 수 있는 계정에서 실측) | `RegionSelectLogic`·`ResultPopupLogic`이 3번째 상자 수령 성공 문구 뒤에 ` · 다이아 +300`을 붙임 |

완료 조건: 메이커 빌드 오류 0, LSP 진단 오류 0, 콘솔 `LEA-` 없음, T1~T26 통과(T11 탑 확인·T25·T26 실측은 정적 검사 대체 가능, 사유 보고), `git diff --stat`으로 1절 표의 파일만 바뀌었는지 확인(`PlayerEngrave.mlua`에 이 단계 변경 없음). 테스트 끝에 `r:DebugReset()`, `r:DebugSetBanner("")`, 편성 `Hero,Bishop,NightLord`로 되돌린다(`rf`·`rs`는 실제 진행과 맞게 둔다).

---

## 8. 이후 단계로 넘기는 것

- 4단계: (02-Roadmap 4단계 보류 항목) 잠긴 직업 판 전달 방어, 메인 바이퍼 HUD 문제.
- 6단계: 스킬 뽑기(`Skill/GachaUILogic.mlua`, `ui/GachaUI`, `PlayerEngrave.RequestGacha`) 삭제, 스킬 탭 편성 실패 안내(2단계 보완 관찰 6), 첫 지급 출시값(6-3), 다이아 반복 수급·10구간 별 상자 기준·초반 보정(6-2), 테스트 함수 정리.
- 빠진 리소스(사용자 확인 후 준비): 경험의 서 소·중·대 아이콘, 직업 전용·공용 전직 카드 아이콘, 캐릭터 뽑기 전용 메뉴 아이콘(지금은 옛 스킬 뽑기 아이콘 재사용), 결과 연출용 이펙트(지금은 칸 순서 공개만).

---

## 3단계 보완 (4단계 조사에서 결정)

> 작성: 자료 조사 에이전트(4단계), 2026-11-22
> 3단계 테스트 보고서(`Phase-3-Test-Report.md`)의 버그 B1과 질문 Q1~Q6에 대한 결정. 구현은 **4단계 작업 단위 4-A와 같은 구현 에이전트가 4-A를 시작하기 전에** 한다(`Phase-4-Combat.md` 1절 순서 참고).
> 건드리지 않는 파일: `Engrave/PlayerEngrave.mlua`, `Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md`.

| 번호 | 질문 | 결정 | 바꾸는 파일 |
|---|---|---|---|
| B1·Q1 | 결과 팝업 다이아 아이콘이 칸 위로 밀림 | **다이아만 `PreserveSprite None`**. 다른 보상 아이콘은 `AspectOnly` 그대로 | `Tower/ResultPopupLogic.mlua` |
| Q2 | 결과 칸 `Name` 글자가 BestFit으로 16까지 줄어듦 | **`Name` 높이를 66으로 늘려 글자 22 유지.** 아이콘·글자 칸을 4px 올린다 | `ui/CharGachaUI` |
| Q3 | 교환 창 5차 안내가 두 줄 모두 빨강 | **둘째 줄만 빨강.** `Info`를 두 텍스트(`Info`, `Info2`)로 나눈다 | `ui/CharGachaUI`, `Character/CharGachaUILogic.mlua` |
| Q4 | 상시 뽑기 때 왼쪽 가운데가 비어 보임 | **상시일 때만 직업 이름 목록 3줄**(새 텍스트 `Banner/AllJobs`)을 아이콘 자리에 표시 | `ui/CharGachaUI`, `Character/CharGachaUILogic.mlua` |
| Q4-2 | 잠긴 직업 미리보기 자물쇠가 아바타 얼굴과 겹침 | **자물쇠를 발밑 아래로**(덮개 기준 y 40 → -150) | `ui/CharacterUI` |
| Q5 | 결과 팝업 "기존 달성" 순서 문제 | **판 시작 때 그 층의 별을 저장해 두고 결과에 그 값을 쓴다** + 결과 화면이 값이 바뀌면 다시 채운다. PlayerEngrave 수정 없음 | `Tower/TowerRunManager.mlua`, `Tower/ResultPopupLogic.mlua` |
| Q6 | 테스트 계정 20층 진행 | **그대로 둔다** (`ClearedFloor=20`, `rf=20`). 이후 테스트 기대값은 20층 진행 기준 | 없음 |

### (a) B1 — 다이아 아이콘 (`Tower/ResultPopupLogic.mlua` `Fill`, 186행 부근 보상 칸 반복문)

보상 칸은 다시 쓰이므로 다이아일 때 `None`, 아닐 때 `AspectOnly`로 **매번 되돌린다**.

```lua
if it ~= nil then
	self:SetImage(p .. "/Loot" .. tostring(k) .. "Icon", it.icon)
	-- 다이아 스프라이트는 피벗이 가운데가 아니라 AspectOnly에서 위로 밀린다 → 다이아만 None
	local iconEnt = self:E(p .. "/Loot" .. tostring(k) .. "Icon")
	if isvalid(iconEnt) then
		local spr = iconEnt.SpriteGUIRendererComponent
		local want = PreserveSpriteType.AspectOnly
		if it.icon == self.IconDiamond then want = PreserveSpriteType.None end
		if spr.PreserveSprite ~= want then spr.PreserveSprite = want end
	end
	-- (기존 Count·Name 줄 그대로)
```

확인: 미클리어 층 첫 클리어(또는 아래 Q5 테스트처럼 별 기록을 비운 층) 결과 팝업 → `Loot0Icon` `PreserveSprite = None`, 스크린샷에서 다이아가 칸 가운데. `Loot1Icon`(스킬북)은 `AspectOnly`.

### (b) Q2 — 결과 칸 이름 (`ui/CharGachaUI`, UIBuilder `patchComponent`로 10칸 모두)

| 경로 | 바꾸기 전 | 바꾼 뒤 |
|---|---|---|
| `Root/Result/Cell{n}/Icon` | pos 0,40 | pos **0,44** (크기 112×112 그대로) |
| `Root/Result/Cell{n}/Label` | pos 0,40 | pos **0,44** (180×112 그대로) |
| `Root/Result/Cell{n}/Name` | pos 0,-55 / 184×56 | pos **0,-51 / 184×66**, 글자 22, BestFit 유지하되 최소 크기 **20** |
| `Root/Result/Cell{n}/Tag` | pos 0,-100 / 184×30 | 그대로 |

- 세로 배치(칸 250, 테두리 12): 띠 109~117, 아이콘 -12~100, 이름 -84~-18, 태그 -115~-85. 서로 겹치지 않는다.
- 확인: `U.Viper`·`J.Bowmaster` 결과에서 `Name`의 실제 글자 크기 22 유지(BestFit이 줄이지 않음), 스크린샷.

### (c) Q3 — 교환 창 안내 두 줄 (`ui/CharGachaUI` + `CharGachaUILogic.mlua` 646행 부근)

| 경로 | 종류 | pos | 크기 | 값 |
|---|---|---|---|---|
| `Root/Modal/Exchange/Body/Info` (기존, 수정) | text | **0,-95** | **1080×32** | 22, 일반색, 가운데, 1줄 |
| `Root/Modal/Exchange/Body/Info2` (신규) | text | 0,-127 | 1080×32 | 22, 가운데, 1줄, RaycastTarget false |

```lua
self:SetText("Modal/Exchange/Body/Info", sj.name4 .. " 전용 전직 카드 1장 — 교환 포인트 " .. tostring(exCost) .. " 사용", self.TextColor)
self:SetText("Modal/Exchange/Body/Info2", second, color)   -- color: 5차 완료면 BadColor, 아니면 TextColor
-- 선택 직업이 없을 때는 두 줄 모두 ""
```

확인: T20 절차(히어로 5차 선택) → `Info` 일반색, `Info2` `5차를 마친 직업은 교환할 수 없습니다` 빨강(0.78,0.20,0.20). 잠긴 직업 선택 → `Info2` `잠긴 직업은 이 카드로 바로 해금됩니다` 일반색.

### (d) Q4 — 상시 뽑기 빈 영역 (`ui/CharGachaUI` + `CharGachaUILogic.mlua` 357~369행)

가장 단순한 처리: 아이콘 자리에 **직업 이름 목록**을 글자로 보여 준다(새 리소스 없음).

| 경로 | 종류 | pos | 크기 | 값 |
|---|---|---|---|---|
| `Root/Banner/AllJobs` (신규) | text | 0,130 | 600×150 | 26, 일반색, 가운데·가운데, 3줄, RaycastTarget false, **enable false** |

- 상시(`GachaBanner == ""`): `AllJobs` 켜기, 글자 = `모든 직업이 같은 확률로 나옵니다\n` + 직업 `Name4`를 표 순서로 앞 3개 ` · `로 잇고 줄바꿈 + 나머지(예: `히어로 · 비숍 · 나이트로드\n보우마스터 · 바이퍼`). 직업 수가 늘어도 3개씩 끊어 줄을 만든다(최대 4줄까지, 넘치면 BestFit 최소 18).
- 픽업 중: `AllJobs` 끄기(지금처럼 `IconBox`·`JobName` 표시).
- 확인: `r:DebugSetBanner("none")` → `AllJobs` 켜짐, `IconBox` 꺼짐, 스크린샷에서 제목과 `Desc` 사이에 글자 3줄이 영역 안. `DebugSetBanner("B001")` → `AllJobs` 꺼짐.

### (e) Q4-2 — 미리보기 자물쇠 위치 (`ui/CharacterUI`)

| 경로 | 바꾸기 전 | 바꾼 뒤 |
|---|---|---|
| `Root/Detail/PreviewDim/Icon` | pos 0,40 | pos **0,-150** (64×64 그대로) |

- 근거(덮개 `PreviewDim` 기준, 380×460 → 세로 -230~230): 아바타 영역(`Avatar` 230×300)은 -115~185, 자물쇠는 -182~-118(발밑 아래 3px), `PreviewName`(덮개 기준 -223~-187, 덮개보다 뒤 형제라 위에 그려짐)과 5px 떨어짐.
- 확인: 잠긴 보우마스터 선택 스크린샷에서 자물쇠가 아바타 발 아래, 이름 글자 위에 겹치지 않음.

### (f) Q5 — 결과 팝업 "기존 달성" 순서 문제

**원인 조사 결과**
- `ResultPopupLogic.Fill`(146행)은 `PlayerEngrave.StarData`를 직접 읽지 않고 **서버가 정한 `mgr.ResultStarsPrev`/`ResultStarsNow`(Sync)** 를 읽는다.
- 서버 `TowerRunManager.StageClear`(3019~3021행)는 `ResultStarsPrev = eng:GetStars(층)` → `eng:RecordStars(...)` → `ResultStarsNow = Prev | bits` 순서라 **코드상 순서는 맞다.** `Defeat` 경로(3846행)도 기록 전 값을 읽는다.
- 따라서 `prev=7`이 나올 수 있는 경우는 ① 같은 층 결과가 두 번 계산되어 두 번째 계산이 이미 기록된 별(7)을 이전 값으로 읽는 경우(테스트 중 `ApplyMonsterDamage`를 반복 주입해 스테이지 클리어 처리가 같은 초에 2~3번 찍힌 기록이 있음), ② 클라이언트가 `RunState = RESULT`를 먼저 받고 별 Sync 값을 늦게 받아 **이전 판 값으로 한 번만 채운 경우**(Fill은 팝업이 켜지는 순간 한 번만 실행)다. 테스트 보고서의 "클라이언트가 갱신된 StarData를 읽는다"는 추정은 코드와 맞지 않는다. 어느 쪽이든 아래 두 줄 수정으로 막힌다.

**수정안 (PlayerEngrave 수정 없음)**

1. `TowerRunManager.StartRun`: `self.StageEndFloor = ...`를 정한 직후
   ```lua
   -- 결과 화면 '기존/새로 달성' 판정용: 판 시작 시점의 이 층 별 (판 중 기록·중복 정산과 무관하게 고정)
   self._T.startStars = (engrave ~= nil) and engrave:GetStars(self.StageEndFloor) or 0
   ```
2. `StageClear`(3019행)와 패배 결과(3846행)의 `eng:GetStars(...)` 대신 `self._T.startStars or (eng ~= nil and eng:GetStars(...) or 0)`을 쓴다. 로그 `[VRF] star result ... prev=`는 그대로 남긴다(판 시작 로그에도 ` stars=` 추가).
3. `Defeat()` 첫 줄에 `if self.RunState == "RESULT" then return end` (이미 결과가 정해진 판에 늦게 온 패배가 결과 값을 덮지 않게).
4. `ResultPopupLogic.OnUpdate`: 팝업이 보이는 동안 `floor..":"..prev..":"..now..":"..clear` 키가 마지막으로 채운 키와 다르면 `Fill(mgr)`을 다시 부른다(늦게 도착한 Sync 값 반영). `Fill` 끝에서 `self._T.fillKey`를 기록한다. 연출 중 다시 채우면 별 연출이 처음부터 다시 도는 것은 허용한다.

**확인 (테스트 계정 진행을 바꾸지 않는 방법)**: server_main에서 `eng._T.stars[20] = nil; eng:SyncStars()`(데이터만 비움, 코드 수정 없음) → 20층 판 시작 → `[VRF] run start ... stars=0` → 보스 처치(4-A 디버그 함수 `mgr:DebugFinishFloor()`가 생긴 뒤라면 그것으로, 그 전이면 3단계 테스트 방식 `mgr.Stage = mgr.StagesPerFloor` + `ApplyMonsterDamage(999999)` 반복) → `[VRF] star result floor=20 ... prev=0`, 결과 팝업 미션 3줄이 `새로 달성`(초록), `신규 별 +3` 배지. `RecordStars`가 별 7을 다시 기록하므로 계정 진행은 원래대로 돌아온다(`StarData`에 `20:7`). 같은 층을 한 번 더 깨면 세 줄 모두 `기존 달성`.

### 3단계 보완 완료 조건

- 메이커 빌드 오류 0, 콘솔 `LEA-` 0, 위 (a)~(f) 확인 항목 통과(스크린샷 포함).
- `git diff --stat`에 `Engrave/PlayerEngrave.mlua`의 이 작업 변경이 없음.
