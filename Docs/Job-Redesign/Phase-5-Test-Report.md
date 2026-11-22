# 5-G 회귀 확인 보고서: 편성 칸 장비(주문서·큐브·금색 메소) 반영 후

> 작성: 테스트 에이전트, 2026-11-22
> 기준: `00-Master-Spec.md`(8절 메소 재화), `01-Agent-Rules.md`(테스트 범위·메이커 공유), `Phase-5-Equipment.md`(5-G = F33~F36, 모든 "메인 세션 결정" 절)
> 방법: 메이커 새로고침 → 빌드 로그 확인 → 플레이 → `maker_execute_script`(server_main / client)로 조작 → `[VRF]` 로그·저장소 읽기·스크린샷으로 판정. 버튼은 클라이언트에서 `SendEvent(ButtonClickEvent())`. 판 안 선택(돌파·포탈·미스터리 등)은 클라이언트 스크립트가 상태별로 UI와 같은 요청(`RequestPick(1)` / `RequestPortal(1)` / `RequestMystery(false)` / `RequestShopLeave()` / `RequestDeal("")` / `RequestBookClose()` / `RequestPickJob(1)`)을 보냈다.
> 코드·UI·데이터 파일은 고치지 않았다(이 문서만 새로 만듦). 디버그 판·무적·웨이브 강제 처치는 쓰지 않았다.

---

## 1. 요약

| 항목 | 결과 |
|---|---|
| 빌드 로그 | 오류 0 (Info `LIA` 1162, 경고 `LWA` 31 — 기존과 같은 인자 개수 경고) |
| 1. 재접속 시 이전·합산 재발 없음 | **통과** |
| 2. 옛 경로 호출 0 (F34) | **통과** |
| 3. 일반 판 1층 1회 (F35·F36) | **통과** |
| 4. 성장창 장비 탭 주문서 강화·레드 큐브 | **통과** |
| 5. 캐릭터 화면 편성 교체·되돌림, 지역 선택 툴팁 | **통과** |
| 콘솔 `LEA-` | 0건 |
| 저장 복원 | **미완료** — 시작 원문 13개 키 기록·검증은 끝났으나, 되쓰기 직전(13:13경) 다른 세션이 내 플레이를 멈추고 새 플레이(13:14~)를 시작해 지금까지 진행 중이다. 남이 연 플레이라 멈추지 않았고, 그 세션이 장비 탭을 쓰고 있어 저장소를 덮어쓰지 않았다(6절) |

---

## 2. 재접속 시 이전·주문서 합산 재발 없음 (F33 일부)

- 첫 플레이(12:59:31): `[VRF] gear loaded mig=true cube=F:30,R:20,B:5,P:1 pend= gold=0 us=1:139,2:13,3:3`.
- 이후 다른 세션(펫 작업)이 두 번 다시 플레이(13:14:00, 13:14:13): 두 번 모두 `gear loaded mig=true ...`.
- 세 번의 불러오기 동안 `gear migrated` 0건, `gear scroll merge` 0건. 저장 키에 `mig=1`·`usm=1`이 있어 다시 돌지 않는다.
- `DebugGearReset`(큐브 재지급)은 사용자 저장을 건드리므로 이번에는 쓰지 않았다(5-A·5-D 때 확인됨).

## 3. 옛 경로 호출 0 (F34)

- 정적 검사: `RootDesk/MyDesk` 전체 `.mlua`에서 `SlotRunData` / `FlushSlots` / `RequestSlotUpgrade` 호출이 `Engrave/PlayerEngrave.mlua` 밖에 0개. `AddScrolls` 호출은 `TowerRunManager.mlua:4432`의 `gear:AddScrolls`(PlayerGear) 하나뿐. `FlushSlots`는 주석 2줄(3468, 4313)에만 남아 있다.
- 판 로그: `slot upgrade`(PlayerEngrave 강화 로그) 0건. 판 뒤 저장소 `OrbisSlot` 원문이 시작과 글자 단위로 같음(`slotUnchanged=true`).

## 4. 일반 판 1층 1회 (F35·F36)

- 진입: 로비에서 `_TeleportService`로 `OrbisTower` 이동 → 페이드인 `RequestStart` → `run start stage=1 floor=1`, 결과 `debugRun=false`. 편성 `Bishop, NightLord, Hero`.
- 진행: 13:00:57 시작 → 13:09:23 결과(전투 시간 08:25). 스테이지 1~10(보스 포함) 모두 클리어, 체력 100%, 판 안 선택 29회 자동 응답.
- 금색 메소: 스테이지마다 `gold meso +23`(일반) / `+46`(정예) / `+115`(보스) 10회, 합계 **414**, 모두 `debugRun=false`.
- 큐브: 스테이지 클리어 무료 +1 ×5, 정예 레드 +1 ×3, 보스 `red+2 free+3` → 판 합계 레드 5·무료 8. 보스 스테이지에는 스테이지 무료 50%가 따로 붙지 않음(5-D 결정 1과 일치).
- 주문서: 판 끝 공용 주문서 `us=1:139,2:13,3:3` → `1:182,2:16,3:3`(결과 팝업 주문서 ×46). `scroll drop` 로그에 `Shoes` 없음.
- 판 끝 저장: `StageClear`에서 `gear:Flush()` 1회. 결과 직후 저장소 `OrbisGearV1` = 메모리 `Encode()`(`true`), `cube=F:38,R:25,B:5,P:1 gm=414 us=1:182,2:16,3:3`, `GearDirty=false`.
- 정산: `[VRF] account settlement exp=100 level=3` **1회**, `ResultAccountSaved=true`.
- `LEA-` 0건. 4단계 HUD(파티 3명 전투·결과 팝업)는 정상 동작.
- 결과 팝업 스크린샷: `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/p5g_result.png`
  - 전투 보상 줄: **금색 메소 ×414** · 깃털 ×59 · 주문서 ×46 · 레드 큐브 ×5 · 무료 큐브 ×8. **스킬북 없음**. 계정 EXP +100.

## 5. 성장창 장비 탭 · 캐릭터 화면 · 지역 선택

### 5-1. 주문서 강화 1회
- 1칸 · 무기 · 하급 주문서 선택 → 강화하기.
- `[VRF] gear scroll s1 Weapon tier=1 rate=30 cost=100 ok=false lv=0 bonus=3 gold=314 us=1:181,2:16,3:3` — 금색 메소 414 → 314, 하급 182 → 181, 실패 누적 +3%p.
- 화면: 최종 성공 확률 33% = 기본 25% + 주문서 5%p + 누적 3%p, 비용 100(보유 314), 하급 주문서 카드 파랑 선택 표시.
- 스크린샷: `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/p5g_scroll.png`

### 5-2. 잠재능력 레드 큐브 1회
- 잠재능력 모드 → 레드 큐브 → 큐브 사용.
- `[VRF] gear cube R s1 Weapon 1->1 up=false ceil=false cnt=2 lines=SkillDmg@1,CritDmg@1,EliteDmg@1 left=24` — 레드 25 → 24, 줄 갱신, 천장 2 / 30점.
- 스크린샷: `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/p5g_pot.png`

### 5-3. 캐릭터 화면 편성 칸 교체·되돌림
- 1칸 → 2칸 누름: `charui request party swap 1<->2 NightLord,Bishop,Hero` → `charui result ok_party`, 칸 아바타 바뀜.
- 다시 1칸 → 2칸: `party swap 1<->2 Bishop,NightLord,Hero` → `ok_party`. 원래 편성으로 돌아옴.

### 5-4. 지역 선택 툴팁
- `금색 메소` 보상 아이콘 클릭 → 툴팁 "장비 칸(슬롯) 강화 비용으로 쓰는 메소입니다…" 표시.
- 클리어 보상 줄: 금색 메소·깃털, 획득 가능 아이템: 주문서·깃털. **스킬북 아이콘 없음**.
- 스크린샷: `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/p5g_tooltip.png`

## 6. 저장 데이터 복원

### 6-1. 시작 전 기록 (완료)
- 12:59 저장소 원문 기록(`[VRF] BK` 로그) 후, 옮겨 적은 표를 저장소와 다시 비교해 **13개 키 모두 글자 단위 일치**(`[VRF] CMP n=13 diff=`) 확인.
  - `OrbisAccountV1`(105자, `total=633`), `OrbisEnergy`(34, `amount=25`), `OrbisEngrave`(57), `OrbisGrace`(127, `f=566`), `OrbisRelic`(102), `OrbisRelicV2`(244, 열쇠 4), `OrbisRosterV1`(221, `dia=1000`), `OrbisSkill`(915), `OrbisSlot`(180), `OrbisStar`(150), `OrbisSymbol`(44), `OrbisTower`(11, `best=20`), `OrbisGearV1`(758, `cube=F:30,R:20,B:5,P:1 gm=0 us=1:139,2:13,3:3`). `OrbisPetV1`은 없음(nil).
- 원문 표: `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/restore_5g_data.lua`

### 6-2. 되쓰기 (미완료) — 이유
- 13:13경 내가 시작한 플레이가 **내가 아닌 쪽에 의해 멈췄고**, 13:14:00·13:14:13에 새 플레이가 시작됐다(펫 화면 `PetUILogic begin`, `pet ui open` 로그 → 펫 작업 세션 또는 사용자). 이 플레이는 13:26 현재도 진행 중이다.
- 규칙(내가 시작하지 않은 플레이는 멈추지 않음)에 따라 약 13분 기다렸으나 끝나지 않아 되쓰기를 하지 않았다.
- 그 세션은 내 테스트 값(금색 메소 314 등)으로 시작했고, 13:26에 저장소를 읽어 보니 **그 세션에서 장비 탭 강화가 더 일어났다**(금색 메소 314 → 14, 1칸 갑옷 추가 확률 8.28). 지금 되쓰면 그 세션의 조작도 지워지고, 그 세션이 끝날 때 다시 덮어쓸 수 있어 손대지 않았다.

### 6-3. 지금 저장소에서 시작 원문과 다른 키 (13:26 읽기 전용 확인)
| 키 | 시작 원문 | 지금 | 원인 |
|---|---|---|---|
| `OrbisAccountV1` | `total=633` | `total=733`, `last` 바뀜 | 내 일반 판 계정 EXP +100 |
| `OrbisGrace` | `f=566` | `f=625` | 내 일반 판 깃털 +59 |
| `OrbisRelicV2` | `k=4` | `k=5` | 내 일반 판 열쇠 +1 |
| `OrbisGearV1` | `cube=F:30,R:20 gm=0 us=1:139,2:13` | `cube=F:38,R:24 gm=14 us=1:181,2:16`, 1칸 무기 줄·추가확률 3·천장 2, 1칸 갑옷 추가확률 8.28 | 내 판 보상·주문서 강화 1회·레드 큐브 1회 + **다른 세션의 강화(금색 메소 300 사용)** |
- 나머지 9개 키(`OrbisRosterV1`·`OrbisRelic`·`OrbisTower`·`OrbisEngrave`·`OrbisSymbol`·`OrbisStar`·`OrbisSlot`·`OrbisEnergy`·`OrbisSkill`(길이 915 같음)): 시작 원문과 같음. `OrbisPetV1` 여전히 없음.

### 6-4. 되돌리는 방법 (메인 세션·사용자용)
- 지금 플레이가 끝난 뒤, **새로 연 플레이**의 server_main에서 아래 스크립트를 한 번 실행 → 곧바로 `stop` → 다시 `play` 후 13개 키를 읽어 비교한다.
  - `/private/tmp/claude-501/-Users-seogeon-u-orbis-tower-rpg/77a25ded-ea2f-4460-abd0-a710532973f7/scratchpad/restore_5g.lua`
  - 스크립트는 `PlayerGear:DebugGearRestore(원문)`로 메모리까지 맞추고, 각 컴포넌트의 더티 표시를 끈 뒤(종료 저장이 덮어쓰지 않게) 13개 키를 `SetAndWait`하고 다시 읽어 `[VRF] RESTORE5G written n=13 bad=` 로 결과를 남긴다.
- 주의: 이 스크립트는 13:14 이후 세션에서 한 장비 강화(금색 메소 300 사용분)도 함께 되돌린다. 그 세션이 펫 작업 에이전트라면 그쪽 복원 원문에 **내 테스트 값이 섞여 있을 수 있다**(13:14 시작 시점 저장소 = 내 테스트 뒤 값).

## 7. 발견 사항 (수정은 하지 않음)

| 번호 | 내용 | 위치 | 심각도 |
|---|---|---|---|
| N1 | 판 시작 로그 `gear run s1 lv=We0,Ca0,Co0,Gl0,Ca0` — 투구(Cap)와 망토(Cape)가 둘 다 `Ca`로 찍혀 구분이 안 된다. 로그 표기만의 문제(값 계산과 무관) | `Tower/TowerRunManager.mlua` `StartRun`의 `gear run` 로그(부위 앞 2글자) | 낮음 |
| N2 | 결과 팝업 금색 메소 칸의 `x414` 숫자가 아이콘 아래쪽과 살짝 겹친다(세 자리 수). 다른 칸은 정상 | `Tower/ResultPopupLogic.mlua` 보상 칸(`center = true` 항목) | 낮음(표시) |
| N3 | 이번 판은 지역 선택 `입장` 버튼이 아니라 텔레포트로 들어가 에너지가 줄지 않았다(테스트 방법 차이, 버그 아님) | — | 참고 |
