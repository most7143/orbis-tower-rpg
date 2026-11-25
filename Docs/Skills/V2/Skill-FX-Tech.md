# 스킬 개편 V2 — 스킬 이펙트 기술 조사 (Skill-FX-Tech)

작성 2026-11-24 · 구상(기술 조사) 단계 · 게임 코드는 아직 바꾸지 않았다.
원본: `/Volumes/P7/MapleStoryWorld/최신메이플데이터_XML/Packs/Skill/Skill/_root.xml`(수치·노드), `/Volumes/P7/MapleStoryWorld/최신메이플데이터/_Resources/Skill/_Canvas/<img>/skill/<id>/...png`(그림), `/Volumes/P7/MapleStoryWorld/최신메이플데이터/Character/Character/_root.xml`(캐릭터 동작 프레임).

---

## 0. 결론 요약

1. **타격 시점은 원본 데이터에 이미 들어 있다.** 캐릭터 동작(action)의 **음수 delay 프레임 합계 = 타격 순간**이고, 시전 이펙트(effect)의 큰 베기 프레임도 정확히 그 시각에 시작한다.
   예: 레이징 블로우 `ragingBlow1~4` 모두 앞 4프레임이 −60ms → **240ms 타격**, effect 4번 프레임(큰 베기, 740×440)도 4×60 = **240ms에 시작**.
2. **MSW 저장소에 원본 스킬이 리소스 팩으로 이미 올라와 있다.** 팩 id 형식은 `skill/<img>/skill/<스킬id>`(예 `skill/112.img/skill/1121008`), 요소 `rel_path`가 원본 노드 이름(effect, effect0, hit/0, special/stand1 …)이다.
   레이징 블로우 effect(MSW clip `dee33ecf…`)는 18프레임 **크기·피벗이 원본과 완전히 같다**. 다만 MSW는 **예전 클라이언트 판**이라 hit/0(8프레임, 다른 그림), 엔젤레이(effect 10프레임 ↔ 지금 13프레임), 발할라 등은 다르고, 새 스킬(홀리 워터 2321015, 폭풍의 시 3121020, 홀리 매직쉘 2311009 등)은 팩이 없다.
3. **직접 등록도 된다(실측).** 메이커 콘솔 `upload_sprites`(프레임 PNG + 피벗) → `upload_animationclips`(프레임 순서 + delay)로 원본 delay·origin을 그대로 살린 animationclip이 만들어진다. 레이징 블로우 effect 18프레임 + hit/0 6프레임을 올려 성공했다(명령 2번, 각 수 초~20초).
4. 공용 구조: **스킬마다 "시전 시각 0 기준 타임라인"(이펙트 이벤트 + 타격 이벤트)을 데이터로 적는다.** 서버는 같은 표로 타격 시각에 데미지·hit 이펙트를 넣고, 클라이언트는 시전 순간 시전 이펙트를 낸다. 지연 값은 원본에서 자동 추출한다(§4).

---

## 1. 원본 이펙트 노드 의미

조사한 스킬: 히어로 1121008 레이징 블로우 · 1120003 어드밴스드 콤보 · 1121015 인사이징 · 1121054 발할라, 비숍 2321007 엔젤레이 · 2301002 힐 · 2311009 홀리 매직쉘 · 2301004 블레스 · 2311011 홀리 파운틴 · 2321015 홀리 워터 · 2321052 헤븐즈 도어 · 2321003 바하뮤트, 나이트로드 4111002/4211008 쉐도우 파트너 · 4121013 쿼드러플 스로우 · 4121015 퍼지 에어리어 · 4001344 럭키 세븐, 보우마스터 3121020 폭풍의 시 · 3111013 애로우 플래터 · 3121015 언카운터블 애로우 · 3101005 바람의 시, 바이퍼 5121013 전함 노틸러스 · 5121007 피스트 인레이지, 듀얼블레이드 4341006 더미 이펙트.
(참고: 쿼드러플 스로우는 지금 데이터에서 **4121013**이다. 표창 발사체 그림은 스킬이 아니라 소비 아이템(표창) 쪽에 있어 스킬 노드에 `ball`이 없다.)

### 1-1. 노드 표

전체 스킬(약 9,700개) 바로 아래 노드 빈도: effect 4546 · hit 3463 · effect0 1721 · special 724 · summon 720 · affected 657 · mob 418 · screen 331 · ball 289 · repeat 280 · effect1 183 · prepare 180 · keydown 153 · keydownend 144 · tile 101 · end 134 · affected0 196 · repeat0 68 · special0 106.

| 노드 | 의미 | 어디에 그리나 | 예시 (프레임 / 합계) |
|---|---|---|---|
| `effect` | **시전 이펙트** (캐릭터 앞쪽 레이어) | 캐릭터 발밑 원점에 **붙여서**(따라다님). 원본 그림은 왼쪽 기준 → 오른쪽 볼 때 FlipX | 레이징 블로우 18f/930ms, 힐 14f/840ms |
| `effect0`, `effect1`, `effect2`… | 시전 이펙트 추가 층. 보통 `z = -1` → **캐릭터 뒤** 층. `fix = 1`이면 시전 위치에 **고정**(따라다니지 않음) | effect와 같은 원점, 정렬 순서만 캐릭터 뒤 | 인사이징 effect0 22f/1980ms (`fix=1`, z−1), 엔젤레이 effect0 6f |
| `hit/0`, `hit/1` … | **몬스터 타격 이펙트**. 번호는 변형(타수·무작위용). `hit/randomHit = 1`이면 무작위 고름 | 맞은 몬스터 위치(몸 중앙 부근)에 **고정** | 레이징 블로우 hit/0 6f/360ms, 바람의 시 hit/0~6 + randomHit |
| `hit2` (+`randomHitOrigin`) | 두 번째 타격 이펙트, `randomHitOrigin = 15`면 위치를 ±15px 흔듦 | 몬스터 | 피스트 인레이지 |
| `ball` / `flipBall` | **발사체**. `ball/width·height`는 판정 크기. flipBall은 반대 방향용 그림 | 발사 위치에서 대상까지 이동, 루프 | 엔젤레이 ball 8f/480ms, 애로우 플래터 ball(z=3) |
| `tile/0`, `tile/1`, `tile/2` | **바닥 설치형**. 0 = 등장(1회), 1·2 = 유지 루프(변형). 원점이 바닥 | 바닥(발판) 위 고정. 넓은 범위면 가로로 반복 배치 | 홀리 워터 tile/0 16f/1530ms + tile/1·2 8f 루프, 퍼지 에어리어 tile/0 37f/2970ms |
| `affected` / `affected0` | **대상(아군·자신)에게 붙는 버프 이펙트** (1회) | 대상 캐릭터에 붙임 (affected0은 뒤층) | 힐 affected 14f, 홀리 매직쉘 affected 10f |
| `repeat` / `repeat0` | **유지 중 반복 루프** (버프·보호막 동안) | 캐릭터에 붙여 루프, 끝나면 제거 | 홀리 매직쉘 repeat 15f (delay 없음) |
| `end` / `end0` | 유지형이 **끝날 때** 1회 | 캐릭터에 붙임 | 디바인 프로텍션, 피닉스 드라이브 |
| `special`, `special0~3` | 스킬별 특수 그림 (콤보 구슬, 파트너 몸, 지연 등장 이펙트) | 스킬마다 다름 | 어드밴스드 콤보 special 20f(`repeat=12`, `relMove(0,-40)`), 헤븐즈 도어 special 첫 프레임 1020ms 빈 대기 후 등장 |
| `special/<동작이름>` | **캐릭터 동작을 그대로 따라 하는 분신 그림** (쉐도우 파트너) — §5-3 | 캐릭터 옆에 붙어 캐릭터 동작·프레임 번호를 따라감 | 4111002 special/stand1·walk1·swingO1·stabO1·shoot1 … (delay 없음) |
| `summon/<상태>` | **소환수** 상태별 그림 (summoned 등장, stand/fly 대기, attack1 공격, die) + `attack1/info/attackAfter`(ms) | 소환 위치 고정 또는 캐릭터 따라감 | 바하뮤트 attack1 20f/2400ms, `attackAfter = 990` |
| `prepare` / `keydown` / `keydownend` | **충전·키다운형**: 준비(1회) → 누르는 동안 루프(`repeat`) → 끝 | 캐릭터에 붙임, 각 노드에 `action`(캐릭터 동작)·`time` | 폭풍의 시 prepare 9f/540ms → keydown 11f/660ms(repeat 5) → keydownend 4f |
| `mob` / `mob0` | **몬스터 머리 위 상태 이펙트** (기절 별 등). `pos`(위치 기준), `repeat = 1`(루프) | 몬스터에 붙여 루프 | 샤이닝 레이 mob 3f(stun), 퍼지 에어리어 mob 8f |
| `screen`, `screen2·3` | **화면 전체 연출** (원점이 화면 기준, 예 (751,333)) | 카메라 중앙에 고정, 맨 위 층 | 전함 노틸러스 screen 34f/3060ms |
| `state` (+`state/start`) | 유지 상태 표시 (콤보 구슬 등) | 캐릭터 주변 | 어드밴스드 콤보 state 6f |
| `trembleEffect` | 화면 흔들림 (`force`, `time`ms) | 카메라 흔들기로 대응 | 인사이징 force 7 / 720ms |

### 1-2. 프레임 값 읽는 법

| 값 | 뜻 | MSW 변환 |
|---|---|---|
| `delay` (ms) | 그 프레임이 보이는 시간. **없으면** 코드가 따로 정하는 프레임(파트너·상태 표시) — 클립으로 만들 때는 100ms로 둔다(메이플 기본값으로 알려진 값, 추정) | 클립 `sprites[].delay` = **초** 단위 (60ms → 0.06) |
| `origin (x, y)` | 그림 왼쪽 위에서 잰 **기준점(캐릭터 발밑 / 몬스터 위치)** 픽셀 | 스프라이트 피벗 = (`x / 폭`, `1 − y / 높이`) — MSW 피벗은 **왼쪽 아래 기준**. 기존 MSW clip `dee33ecf`가 정확히 이 값(예 4번 프레임 740×440, origin(375,293) → 피벗 픽셀 (375,147)) |
| `z` | 프레임 단위 정렬 (대부분 0). 노드 단위 `z = -1`은 **캐릭터 뒤** | `PlayEffect` 옵션 `SortingLayer`·`OrderInLayer`로 캐릭터보다 낮게/높게 |
| `a0`, `a1` | 그 프레임 동안 투명도 255→128처럼 **서서히 변함**(2,143곳, 주로 mob0·사라지는 끝 프레임) | 클립에는 프레임별 페이드가 없다 → 그 프레임을 2~3장으로 쪼개 투명도를 구워 넣는다(원본 수정이므로 `수정한리소스/`에 보관) |
| `_outlink` | 다른 노드 프레임 그림을 **재사용** (예 홀리 워터 tile/1 0번 = tile/0 8번) | 같은 PNG는 한 번만 올리고 RUID를 재사용 |
| 폭·높이 1×1 | XML 상 자리 표시일 뿐, 실제 크기는 PNG | PNG 헤더에서 읽는다 |
| `noFlip = 1` | 좌우 반전 금지 (쉐도우 파트너 effect) | FlipX를 항상 false |
| `fix = 1` | 시전 위치에 고정 | `PlayEffectAttached` 대신 `PlayEffect` |
| `relMove (x, y)` | 이펙트 위치 보정 픽셀 | 위치 오프셋 (y는 부호 반대: 원본 y 아래가 +) |

---

## 2. 타격 시점 — 원본에서 뽑는 규칙

캐릭터 동작 데이터(`Character/_root.xml`의 `<node name="ragingBlow1">` 등)는 프레임마다 `action`·`frame`·`delay`. **앞쪽 음수 delay 프레임이 "공격 전", 처음 양수 프레임이 시작되는 순간이 타격**이다. 지금 코드(`TowerPlayer.HitTimeOf`)도 이 규칙을 쓴다.

| 동작 | 프레임 수 | 음수 합(=타격 ms) | 동작 전체 ms | 비고 |
|---|---|---|---|---|
| ragingBlow1~4 | 12~14 | **240** | 660 | effect 큰 프레임 시작 240ms와 일치 |
| quadrupleThrow | 8 | 180 | 660 | |
| uncountableArrow | 6 | 420 | 660 | |
| HY232heavensDoor | 2 | 180 | 360 | 실제 큰 연출은 special 1020ms 대기 뒤 |
| alert5 (샤이닝 레이) | 3 | 300 | 900 | |
| holyshield | 2 | 300 | 720 | |
| incising / angelRay / luckySeven / bombExplosion / fistEnrageNew | — | 0 (음수 없음) | 660~720 | 아래 우선순위 2~4로 정한다 |

**타격 시각 결정 우선순위** (자동 추출 스크립트로 한 번에 계산해 표에 적는다):
1. 동작의 음수 delay 합계 (> 0이면 그 값).
2. `common/attackDelay` (예: 전함 노틸러스 1950ms).
3. 이펙트 노드 첫 프레임이 **빈 그림 + 긴 delay**면 그만큼 뒤 (헤븐즈 도어 special 1020ms, 블레스 effect 0번 420ms).
4. 그래도 없으면 이펙트의 "처음으로 큰 프레임(최대 면적의 50% 이상)" 시작 시각 (레이징 블로우 240, 인사이징 effect 180 / effect0 720).
5. 발사체: 위 시각에 발사 → `거리 / 속도` 뒤 닿을 때 타격.
6. 소환수: `summon/attackN/info/attackAfter` (바하뮤트 990ms).

다단히트(`attackCount`)는 첫 타 시각 + 타 간격(지금 표 `HitGap` 0.06초)으로 이어 넣고, hit 이펙트는 타마다 1회(긴 이펙트면 첫 타만 — 지금 `HitPolicy` 규칙 유지).

---

## 3. MSW에 애니메이션 등록하는 방법

### 3-1. 경로 비교

| 경로 | animationclip 등록 | 비고 |
|---|---|---|
| `mcp__msw-mcp__asset_create_account_resource_storage_item` | **안 됨** (category가 sprite/audioclip/avataritem뿐) | 스프라이트 1장씩만 |
| 메이커 콘솔 `upload_sprites` → `upload_animationclips` | **됨 (실측)** | `mcp__msw-maker-mcp__maker_run_command`로 실행. 메이커가 켜져 있어야 함 |
| 메이커 콘솔 `modify_animationclips` | 등록된 클립 수정 | 형식은 미확인(필요해지면 오류 메시지로 필드 확인) |
| 메이커 AnimationClip Editor (수동) | 됨 | 사람이 직접, 느림 |

### 3-2. 단계 (실측한 형식)

1) **스프라이트 묶음 JSON** (`upload_sprites <경로> -json`):
```json
[
  { "path": "/Volumes/P7/.../112.img/skill/1121008/effect/4.png",
    "subCategory": "skill", "filter": 0, "wrapMode": 1,
    "border": { "x": 0, "y": 0, "z": 0, "w": 0 },
    "pivot": { "x": 0.50676, "y": 0.33409 } }
]
```
- `pivot`은 0~1 비율, **왼쪽 아래 기준**: `x = originX / 폭`, `y = 1 − originY / 높이`.
- 결과: `[{"Code":0,"FileName":"4","RUID":"79f94bfd…"}, …]` (입력 순서대로).
- 경로에 한글이 있어도 된다. `-out` 옵션은 파일 경로를 받지 않는다("-out argument must be stream") → `-json`만 쓰고 출력을 읽는다.

2) **클립 JSON** (`upload_animationclips <경로> -json`):
```json
[
  { "clipName": "FxV2_1121008_effect",
    "clipDesc": "레이징 블로우 1121008 effect 원본 18프레임",
    "subCategory": "skill",
    "sprites": [ { "spriteRUID": "46b1b926…", "delay": 0.06, "customOffset": { "x": 0, "y": 0 } } ] }
]
```
- `clipName` 필수·50자 이하, `clipDesc` 1000자 이하, `delay`는 초.
- 결과: `[{"Code":0,"Message":"AnimationClips had been uploaded successfully.","Name":…,"RUID":…}]`.

### 3-3. 이번에 올린 것

| 클립 | RUID | 내용 |
|---|---|---|
| `FxV2_1121008_effect` | `ea354f925e2a42638dfc848a2ccfe343` | 레이징 블로우 effect 18프레임, delay 원본(60·30·90ms, 합 930ms), 피벗 = 원본 origin |
| `FxV2_1121008_hit0` | `c04bd755e6334f8eb47a3c5037795cf7` | 최신 hit/0 6프레임(합 360ms) — MSW 기존 hit/0(`7d4fd3b7…`, 8프레임)과 다른 최신 그림 |

프레임 스프라이트 RUID 목록: 스크래치패드 `fx/rb_meta.json`(구현 때 다시 만들 수 있음).
소요: 스프라이트 24장 업로드 명령 1번 + 클립 2개 명령 1번, 합쳐 1분 이내. 원본 파일은 고치지 않았다.

### 3-4. 제약·확인 못 한 것
- **origin이 그림 밖**인 프레임(예 홀리 매직쉘 effect 마지막 o(−56,170))은 피벗이 0~1 밖이 된다. 받아 주는지 미확인 → 안 되면 투명 여백을 붙여 피벗을 안으로 넣는다(수정본은 `수정한리소스/`).
- 아주 큰 그림(screen 노드 1500px 안팎)의 최대 크기 제한 미확인.
- a0/a1 페이드는 클립에서 표현 불가 → 구워 넣기.
- 프로젝트 기존 메모: "직접 올린 스프라이트가 메이플 리소스보다 약 2배 크게 그려져 0.5로 맞춤"(`HeroCombat.AuraEnhScale`). 이번 원본 PNG는 MSW 기존 clip과 픽셀 크기가 같으므로 같은 크기로 보여야 정상 — **메이커에서 나란히 띄워 크기 확인 필요(이번엔 메이커가 다른 작업으로 플레이 중이라 못 함)**.

### 3-5. 권장 흐름 (스킬 하나당)
1. 원본 노드 목록·프레임(크기·origin·delay) 추출.
2. MSW 팩 `skill/<img>/skill/<id>` 조회(`node .claude/skills/msw-search/scripts/msw_resource_api.cjs get "skill/112.img/skill/1121008"`) → 노드별 클립의 프레임 수·크기·피벗이 원본과 **모두 같으면 그 RUID 사용**(등록 불필요).
3. 다르거나 없으면 §3-2로 등록, 클립 이름 `FxV2_<스킬id>_<노드>`(슬래시는 밑줄).
4. 원본 vs MSW 미리보기 이미지 `output/preview/`.
→ 1~3은 스크립트 하나로 자동화 가능(이번 시험 스크립트: `extract.py`, `summ.py`, `prev.py`).

---

## 4. 게임 안 재생과 타이밍 — 공용 구조 설계

### 4-1. 지금 코드 흐름 (조사 결과)
- `TowerRunManager` → `TowerPlayer:DoAttack/DoCast(target, procs)` → `QueueHit` → 클라이언트 `PlayAttackMotion`:
  시전음, **시전 이펙트 `PlayEffectAttached(casterFx, 캐릭터, …, FlipX = Scale.x > 0)`**, 동작 프레임 `PlayAction(actionFrames)`(아바타 기본 동작 프레임을 원본 순서대로 이어 붙임), 그리고 `HitTimeOf(actionFrames)`(음수 합) 또는 `HitSyncDelay` 뒤 `RequestHit` → 서버 `ApplyCast` → `ApplyStrikes` → 타마다 `ApplyHit`(hit 이펙트 `PlayEffect`(몬스터 `HitPos`, FlipX = true) + 소리 + 숫자 + HP).
- 데이터: `Data/SkillActionTable.csv` (`Action, ActionFrames, ActionTime, CasterFx, HitFx, FxScale, MobFx, HitPolicy, UseSound, HitSound, HitGap, HitY, Targets`), 직업별 코드(`HeroCombat`·`JobCombat`·`BishopCombat`)가 특수 연출을 각자 RUID 속성으로 들고 있다.
- 문제점
  1. 레이징 블로우처럼 `ActionFrames`에 음수 프레임이 없으면(H08은 `swingT3:0:100|…` 전부 양수) `HitSyncDelay` 고정값 → 이펙트 큰 베기(240ms)와 숫자가 어긋날 수 있다.
  2. 타격 요청이 **클라이언트 → 서버 → 다시 클라이언트**로 돌아온다 → 실제 서버에서는 hit 이펙트가 시전 이펙트보다 **왕복 지연(2×지연)만큼 늦게** 보인다(메이커에선 지연 0이라 안 보임).
  3. 설치형·붙는 루프·파트너가 직업 코드마다 제각각(소환검 `SwordStandRUID` 루프 등).

### 4-2. 데이터: `SkillFxTable` (스킬당 여러 줄 = 타임라인)

시전 순간을 0ms로 두고 이벤트를 나열한다. 같은 표를 서버(데미지·hit)와 클라이언트(시전·설치 연출)가 함께 읽는다.

| 열 | 뜻 | 예 (레이징 블로우) |
|---|---|---|
| `FxId` | 연출 묶음 id (스킬 차수별로 따로, 예 `H08_1`, 상위 스킬 교체 시 `H08_4`) | `H08_1` |
| `Seq` | 줄 순서 | 1 |
| `AtMs` | 시전 기준 시작 시각(ms) | 0 |
| `Kind` | `cast`(캐릭터에 붙음) · `castFix`(시전 위치 고정) · `hit`(맞은 몬스터) · `ball`(발사체) · `tile`(바닥 설치) · `attachLoop`(붙는 루프) · `end` · `mob`(몬스터 머리 위) · `screen` · `partner` · `shake` | `cast` |
| `Clip` | RUID (`|`로 변형 여러 개) | `ea354f92…` |
| `Src` | 원본 경로 (검수용) | `1121008/effect` |
| `Layer` | `front`(캐릭터 앞) / `back`(z −1) / `top`(screen) | `front` |
| `Flip` | `face`(바라보는 방향, 기본) / `none`(noFlip) / `toTarget` | `face` |
| `Offset` | 픽셀 x,y (relMove 등, x는 앞쪽+) | `0,0` |
| `Scale` | 크기 | 1 |
| `DurMs` | 루프·설치 유지 시간 (0 = 클립 한 번) | 0 |
| `LoopClip` | tile/1 · repeat 같은 이어지는 루프 클립 | |
| `Pick` | 변형 고르기: `cycle`/`random`/`first` | `cycle` |
| `Jitter` | hit 위치 흔들기 px (randomHitOrigin) | 0 |
| `Width` | 설치형 가로 범위 px (lt/rb에서) | |

타격 자체는 스킬 수치 표(새 스킬 표)에 `HitMs`(첫 타), `HitCount`, `HitGapMs`, `HitClip`, `BallSpeed`를 둔다. **`HitMs`는 §2 규칙으로 자동 추출한 값**을 기본으로 적는다.

레이징 블로우 예시:
```
FxId,Seq,AtMs,Kind,Clip,Src,Layer,Flip,Offset,Scale,DurMs,LoopClip,Pick,Jitter,Width
H08_1,1,0,cast,ea354f925e2a42638dfc848a2ccfe343,1121008/effect,front,face,"0,0",1,0,,first,0,
(타격: HitMs=240, HitCount=4, HitGapMs=60, HitClip=c04bd755e6334f8eb47a3c5037795cf7, 동작 ragingBlow1~4 중 무작위)
```

### 4-3. 실행 구조 (`SkillFxPlayer` 공용 모듈 제안)
- **서버**가 시전을 확정하면 `castId`·`FxId`·대상 목록을 클라이언트에 한 번 보낸다(Multicast). 클라이언트는 받은 순간을 0ms로 `cast/castFix/back/screen/partner` 이벤트를 타이머로 예약해 재생 + 동작 프레임 재생.
- **같은 순간 서버도** `HitMs + (h−1)·HitGapMs`에 `ApplyHit`(데미지·숫자·hit 이펙트)를 예약한다. 서버가 보낸 시각 기준으로 양쪽이 같은 지연을 겪으므로 **시전 이펙트와 hit가 어긋나지 않는다**(지금의 왕복 지연 문제 해소). `RequestHit` 왕복은 쓰지 않거나, 동작 재생 확인용으로만.
- 발사체: `HitMs`에 발사(`ball` 클립을 Body 없는 엔티티로 `Translate`, 지금 `heroshot` 풀링 재사용) → 닿는 순간 타격(지금 `TickShots → ApplyStrikes` 그대로).
- 다단히트 중 대상이 죽으면 다음 몬스터로 넘기는 지금 규칙(`ApplyHit` carry)은 유지.
- 공격 속도 버프(`StarTempo`)는 `AtMs`·`HitMs`·동작 프레임·클립 `PlayRate` 옵션에 **같은 배율**을 곱한다(이펙트만 느리고 타격만 빨라지는 어긋남 방지).
- 시작 시 `PreloadFx`로 표의 모든 Clip을 미리 불러 첫 재생 끊김을 막는다(지금 함수 재사용).

### 4-4. 좌우 반전·정렬
- 원본 그림은 **왼쪽 기준** → 오른쪽을 보면 `FlipX = true`(지금 코드와 같음). `noFlip = 1` 노드는 반전 안 함.
- 붙는 이펙트는 시전 순간 방향으로 고정. 시전 중 방향이 바뀌어도 따라 돌게 하려면 `SyncFlip = true`.
- hit 이펙트는 몬스터 쪽에서 캐릭터를 향하게(`FlipX = true`, 몬스터가 오른쪽에 있으므로 지금 값 유지).
- 층: `front` = 캐릭터보다 높은 `OrderInLayer`, `back`(z −1) = 낮게, `screen` = 맨 위. 플레이어의 실제 SortingLayer/OrderInLayer를 구현 때 한 번 재서 상수로 둔다(지금 코드는 옵션 없이 기본값).

---

## 5. 설치형 · 반복 · 붙는 이펙트

### 5-1. 설치형 (`tile`)
- 위치: 시전 시점 대상(또는 캐릭터 앞) **바닥 y**(발판 위)에 `PlayEffect`로 고정.
- 순서: `tile/0`(등장, 1회) → 클립 길이(delay 합) 뒤 `tile/1`(루프, `isLoop = true`, 변형 tile/2는 번갈아/무작위) → `DurMs`(원본 `common/time` 초) 뒤 `RemoveEffect` (+ 있으면 사라짐 클립).
- MSW 이펙트에는 "끝났을 때" 신호가 없으므로 **원본 delay 합으로 타이머를 건다.**
- 범위가 그림보다 넓으면(`lt/rb` 폭 > tile 폭) 가로로 `ceil(범위 / 그림폭)`개를 그림폭 간격으로 깐다.
- 지속 피해는 서버 타이머(`common/dot`·`subTime` 등)로, 이펙트와 별개.

### 5-2. 반복 (`repeat`, `keydown`, `mob`)
- `PlayEffectAttached(clip, 대상, …, isLoop = true)` → 받은 serial 보관 → 효과가 끝나면 `RemoveEffect(serial)` 후 `end` 1회.
- delay 없는 repeat 프레임은 등록 시 100ms로 둔다.
- 충전형(prepare→keydown→keydownend)은 로그라이크 자동 전투에서 "준비 + 고정 반복 횟수(`keydown/repeat`) + 끝"으로 펼쳐 일반 타임라인으로 만든다.

### 5-3. 캐릭터에 붙어 따라 하는 분신 (쉐도우 파트너)
- 원본 구조: `special/<캐릭터 동작 이름>/<프레임 번호>` — **캐릭터 동작 목록과 같은 이름·같은 프레임 수**(stand1 3장, walk1 4장, swingO1 3장, stabO1 2장, shoot1 3장 …), delay가 없다 = **캐릭터의 현재 동작·프레임 번호를 그대로 따라 그리는 방식**. `special/summoned`(12f) 등장, `special/dead` 사라짐, `effect`는 시전(`noFlip = 1`).
- MSW 팩에도 있음: `skill/411.img/skill/4111002` 클립 42개(special/stand1, swingO1, … create2~4 포함).
- 구현안: 플레이어 자식 엔티티 하나(SpriteRenderer, 캐릭터 뒤 층, 뒤쪽으로 약간 오프셋)를 두고, 지금 `TowerPlayer:ShowActionFrame(action, frame)`이 호출될 때마다 **분신도 같은 `special/<action>`의 같은 프레임**으로 바꾼다(팩 클립의 프레임 스프라이트 RUID 사용, 또는 클립 + `StartFrameIndex = EndFrameIndex = frame`). 대기 중엔 `special/stand1` 루프.
- 공격 복제: 분신이 같은 스킬 시전 이펙트·hit를 약간 늦게(예 0.1초, 원본 감각) 한 번 더 — 데미지 배율은 스킬 표에서.
- 같은 방식이 미러 이미징(4331002 special), 소환수(summon/stand·attack1 + attackAfter)에도 쓰인다.

---

## 6. 구현 단계 (제안)

1. **추출 스크립트**: 스킬 id 목록 → 노드별 프레임(크기·origin·delay·z·a0/a1·outlink) JSON + §2 타격 시각 자동 계산 + MSW 팩 대조(같음/다름/없음).
2. **등록 스크립트**: "다름/없음" 노드만 `upload_sprites` → `upload_animationclips`, 결과 RUID를 `SkillFxTable.csv`에 기록. a0/a1·그림 밖 origin은 가공본을 `수정한리소스/`에 저장 후 올림.
3. **미리보기**: 원본 vs MSW 나란히(`output/preview/`).
4. **`SkillFxTable` + `SkillFxPlayer`**(새 폴더 `RootDesk/MyDesk/Character/` 또는 `Tower/`): 타임라인 재생(클라이언트) + 타격 예약(서버) + 설치/루프/분신 관리(serial 정리 포함, 판 종료·맵 이동 시 전부 RemoveEffect).
5. 히어로 1개 스킬(레이징 블로우)로 메이커 확인 → 나머지 직업 확장. 기존 직업 코드의 특수 RUID 속성들은 표로 옮긴다.

---

## 7. 위험

| 위험 | 대응 |
|---|---|
| MSW 팩은 예전 판 → 일부 노드 그림·프레임 수가 최신과 다름 | 노드마다 프레임 서명(수·크기·피벗) 대조, 다르면 등록 |
| 업로드 스프라이트 크기 차이(과거 메모: 2배) | 첫 시험에서 MSW 기존 clip과 나란히 재생해 확인 — **이번엔 미확인** |
| origin이 그림 밖 / 거대 그림 / a0·a1 | 여백 추가·분할·구워 넣기 (원본은 `수정한리소스/`에) |
| 서버 왕복 때문에 실서버에서 hit가 늦게 보임 | §4-3 서버 기준 예약으로 전환 |
| 공격 속도 배율이 연출·타격에 다르게 적용 | 한 배율을 모든 시각에 곱함 |
| 루프·설치 이펙트 serial 누수 (판 종료·사망·맵 이동) | `SkillFxPlayer`가 serial을 모아 일괄 제거 |
| 업로드 명령은 메이커가 켜져 있어야 하고 한 번에 하나(`busy`) | 스크립트로 묶어 몇 번에 나눠 실행 |
| 이펙트 수가 많은 스킬(헤븐즈 도어 30f×다층, 노틸러스 screen)이 모바일에서 무거움 | 층 수 줄이기·크기 옵션은 사용자 확인 후 |

## 8. 이번에 확인하지 못한 것
- 올린 클립(`ea354f92…`)을 메이커에서 실제 재생한 화면 확인: 메이커가 **다른 작업의 플레이 중**(다른 세션의 자동 점검 로그가 계속 찍힘)이라 멈추지 않고 건너뜀. 다음 단계에서 디버그 판 `PlayEffectAttached` + 스크린샷(`output/verification/fx_test_*.png`)으로 확인 필요. 이때 기존 `dee33ecf…`와 나란히 재생해 크기·위치·delay를 비교한다(기존 clip의 실제 delay는 API에 안 나옴 — 썸네일 GIF는 150ms 고정).
- `modify_animationclips` 형식, 피벗 0~1 밖 허용 여부, 최대 그림 크기.
- mob 노드 `pos` 값의 정확한 기준(0/1/2 = 머리·중앙·발로 추정).

## 9. 결과물 위치
- 이 문서: `Docs/Skills/V2/Skill-FX-Tech.md`
- 미리보기: `output/preview/fx_raging_blow.png` (① 원본 effect = 새 클립, ② MSW 기존 clip `dee33ecf`, ③ 원본 hit/0 = 새 클립, ④ MSW 기존 hit/0 `7d4fd3b7`)
