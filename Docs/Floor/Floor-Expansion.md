# 층 확장 — 원본 맵 조사·직접 넣기 시험·98~120층 추가 (2026-11-25)

## 0. 진행 상태 (이어 하기용)

| 단계 | 상태 |
|---|---|
| 1. 원본 맵 조사 | 끝 |
| 2. 직접 넣기 시험 (배경·풋홀드) | 끝 |
| 3. 98~111층 추가 (기존 MonsterTable 몬스터 + 원본 back 합성 배경) | 끝 |
| 4. 원본 몬스터 그림 직접 업로드 (탈라하트·기어드락 13종) | 끝 |
| 5. 112~120층 추가 (12구역 완성) | 끝 — **총 120층** |
| 6. `Docs/FloorPlan_Draft.csv`, `Docs/Boss/field_boss_map.csv` 반영 | 끝 |
| 7. 짧은 실행 확인 | 끝 (98·110·112·116·120층 전투, 97층 복귀, 지역 선택 11·12구역) |

- 커밋하지 않았다. 메이커는 편집 모드, 현재 맵 `Lobby`로 돌려놓았다.
- 다음 후보는 §7 참고.

## 1. 결론

- **층을 늘릴 수 있다. 97층 → 120층(23층 추가), 12구역(111~120층)까지 채웠다.**
- 98~111층(14층): 오디움·도원경·아르테리아·카르시온. 몬스터는 이미 `MonsterTable`에 있던 650~681번. 배경은 **원본 back 조각을 한 장으로 합성**해 올린 스프라이트.
- 112~120층(9층): 탈라하트·기어드락. MSW에 몬스터 팩이 없어서 **원본 몬스터 프레임을 메이커 `upload_sprites` + `upload_animationclips`로 직접 올려** 13종(713~725)을 새로 만들었다. 배경은 원본이 스파인 애니메이션이라 **아틀라스 그림으로 근사 합성**했다(§3.4).
- 풋홀드도 원본 좌표로 넣을 수 있다(사용자 발판 + 정렬 레이어). 탑 칸 전투에는 쓰지 않았다(§3.3).

## 2. 조사

### 2.1 근거 자료

- 원본 맵: `/Volumes/P7/MapleStoryWorld/최신메이플데이터_XML/Map/Map/Map/Map0~9.xml` (17,613개 맵). info·back·life·foothold를 스트림으로 읽어 색인을 만들었다 (`output/floor_bg/tools/index_maps.py`).
- 이름: `_Strings/Map.xml`, `_Strings/Mob.xml`. 몬스터 수치·프레임: `Packs/Mob/Mob/_root.xml`, 그림 `최신메이플데이터/_Resources/Mob/_Canvas/<id>.img/...`.
- 배경 조각: `최신메이플데이터/_Resources/Map/Back/_Canvas/<bS>.img/back|ani|spine...`, 기준점·스파인 아틀라스: `최신메이플데이터_XML/Map/Map/Back.xml`.
- 사냥터 판정: 마을 아님, back 있음, life에 몬스터 있음, 9억 번대 제외 → 2,114개(97층이 쓰던 맵 482개).

### 2.2 쓸 만한 미사용 사냥터

| 묶음 | 지역 | 몬스터 | MonsterTable | 메이커 맵 목록 | 처리 |
|---|---|---|---|---|---|
| A | 오디움·도원경·퀸스로드+아르테리아·카르시온 | 32종 | 이미 있음 (650~681) | 없음 | **98~111층으로 추가** |
| D | 탈라하트·기어드락 | 13종 | 없었음 → **713~725 새로 만듦** | 없음 | **112~120층으로 추가** (원본 프레임 직접 업로드) |
| B | 무릉도원 하늘 숲(14종, 원본 Lv40~70), 루디브리엄 시계탑 최하층·히든(19종, Lv61~108), 엘나스 폐광·미나르숲(6종) | 39종 | 있음 | 있음 | **나중에 끝에 붙일 후보** (§7) — 중간에 끼우면 층 번호가 밀려 저장 최고 층·별·보스 해금 층이 어긋나므로 끼우지 않는다 |
| C | 여우고개·청운골·리스토니아·툴렌시티·라라 지역·에리모스 | 43종 | 없음 (MSW 팩 39/49 있음) | 에리모스 빼고 있음 | 원본은 직업 이야기 초반 저레벨 지역 → **나중에 끝에 붙일 후보** |

## 3. 직접 넣기 시험

### 3.1 배경 — 원본 back 합성 (98~111층)

1. 실행 중 `BackgroundComponent.Type`·`SpriteRUID` 대입은 `cannot set ..., no such field`(읽기 전용). 템플릿은 메이커 목록에 있는 맵만 있다.
2. 채택: 원본 맵의 front=0 back 조각을 1600x900 가상 카메라 화면으로 합성(`output/floor_bg/tools/render_back.py`) → 메이커 `upload_sprite` → 탑 맵 새 엔티티 `FloorBackdrop`(SpriteRenderer, `MapLayer0`, OrderInLayer −1000)이 카메라 화면을 덮도록 매 프레임 위치·크기를 맞춘다.
   - 합성 규칙: 조각 순서 = back 번호 순, `type` 1·3·4·6 가로 반복 / 2·3·5·7 세로 반복(간격 cx·cy), `rx·ry` 시차는 카메라 한 지점 기준 고정, `ani=1`은 0번 프레임, `f` 반전, `a` 투명도. 주 지면을 화면 아래 78%에. 아래가 비면 세로로 맞춰 확대.
   - 층 데이터: `FloorTable.BgRUID = sprite:<RUID>`. `TowerRunManager.ApplyFloorLook`은 이 값이면 템플릿을 바꾸지 않고 `FloorBgRUID`만 넘긴다. 클라이언트 `FloorBackdrop`이 `sprite:`면 표시, 아니면 숨김. `ScreenFadeLogic.LookReady`는 `sprite:`면 미리 불러오기만 기다린다.

### 3.2 풋홀드 — 원본 foothold 좌표

| 시도 | 결과 |
|---|---|
| 맵 루트 `FootholdComponent` 교체 (시험 맵) | 안 됨 — MapleTile 발판은 타일에서 다시 만들어짐 |
| 실행 중 `foothold` 모델 + `edgeLists:Add` / 렌더러 없는 사용자 발판 | 캐릭터가 통과 |
| 사용자 발판 + `SpriteRendererComponent`(SortingLayer `MapLayer0`, 투명) — 맵 파일 | **성공** (원본 오디움 발판 y=3.711에 섬) |
| `Models/Terrain/FloorFoothold.model`(IsDynamicFoothold=true + MapLayer0) 실행 중 소환 + edges | **성공** (y=2.401) |

- 사용자 발판은 SpriteRenderer 정렬 레이어가 캐릭터와 같아야 충돌한다. 실행 중 모양을 바꾸려면 `IsDynamicFoothold=true`를 모델에 미리 넣는다.
- 시험 맵: `map/FloorLookTest.map` (SectorConfig 미등록).

### 3.3 탑에 쓰는 방식

- **층마다 배경만 바꾸고 발판은 탑 맵 그대로.** 칸 전투는 y≈−0.04 한 줄·고정 칸·고정 카메라라 원본 발판을 넣으면 칸이 어긋나고 보이는 바닥 타일과도 안 맞는다. 발판이 필요해지면 `FloorFoothold` 모델을 쓴다.

### 3.4 원본 몬스터 직접 업로드 (탈라하트·기어드락)

- MSW 검색에 팩이 없다(`mob/8645460.img` 등 404, 이름 검색 0건).
- 절차(`output/floor_bg/tools/mobmeta.py`, `mob_batch.py`, `mob_clips.py`):
  1. `_root.xml`에서 동작별(stand·move·attack1·hit1·die1) 프레임의 origin·delay·`_outlink`를 읽는다.
  2. 메이커 `upload_sprites <json> -json` — 프레임 PNG, 피벗 = (origin.x/폭, 1 − origin.y/높이), subCategory `monster`. 같은 그림·같은 기준점은 한 번만.
  3. `upload_animationclips <json> -json` — 동작마다 클립(`Mob_<원본ID>_<동작>`), delay는 원본 그대로.
  4. MonsterTable 행: Stand/Attack/Hit/Die/MoveRUID = 클립, `PackRUID` = `orig/<원본ID>`.
- 결과 매핑(메이커 출력이 커서 대화 기록 jsonl에서 결과를 다시 읽는 `grab.py`를 썼다): `output/floor_bg/monsters_manifest.json`.
- 수정본: `8645466 die1/18.png`는 origin이 그림 아래 밖이라 투명 여백만 붙여 올렸다(`수정한리소스/_Resources/Mob/_Canvas/8645466.img/die1/18.png`, README 기록).
- 크기: 원본 픽셀 1:1. 탈라하트 몬스터는 원본 자체가 커서(키 약 290px) 화면에서 영웅보다 훨씬 크다. 체력바가 화면 위쪽에 붙는다.
- 소리: 원본은 mp3이고 메이커 `upload_audioclip`은 ogg만 받는다(이 PC에 변환 도구 ffmpeg 없음) → `HitSoundRUID`·`DieSoundRUID` 빈칸(코드는 빈칸이면 재생 안 함).
- 미리보기: `output/preview/floor_expansion_custom_monsters.png` (원본 stand 0번 13종 + 메이커 전투 화면 3장).

### 3.5 스파인 배경 근사 합성 (112~120층)

- 탈라하트·기어드락 맵의 back은 전부 `ani=2`(스파인)이다. 바이너리 `.skel`의 뼈대 변환을 해석하지 못해 원본 그대로 재현은 못 했다.
- 근사: 각 back 노드의 스파인 아틀라스에서 가장 큰 영역 그림을 잘라(회전 처리) back 좌표 기준으로 놓고, 첫 층(하늘)은 화면을 덮게 늘리고, 넓은 조각은 가로로 이어 붙인다(`output/floor_bg/tools/render_spine_approx.py`). 그림은 원본이지만 배치는 추정이다.
- 결과가 단색만 나온 맵(410013703)은 같은 지역 다른 맵(410013704)으로 바꿨다. 113·115층은 같은 원본 그림이 골라져 비슷하다.

## 4. 추가한 층 (98~120)

| 층 | 이름 | 지역 | 몬스터 (MonsterTable id) | 배경 원본 맵 |
|---|---|---|---|---|
| 98 | 성문으로 가는 길 | 오디움 | 강화형 파수꾼 산호·감람·금강 (651·658·654) | 410007003 |
| 99 | 점령당한 골목 | 오디움 | 앵글러 포격형·근접형·전자전 시험기 (659·655·652) | 410007008 |
| 100 | 볕 드는 실험실 | 오디움 | 경비병 강옥·마노 (656·653) | 410007013 |
| 101 | 잠긴 문 뒤 실험실 | 오디움 | 무너지는 실험체·실패한 피험체 (657·650) | 410007016 |
| 102 | 생기가 돌아오는 봄 | 도원경 | 시든 봄의 연구자·먹그늘 나비 (664·663) | 410007027 |
| 103 | 빛이 약한 여름 | 도원경 | 빛 없는 여름의 연구자·묵그림자 두루미 (665·661) | 410007030 |
| 104 | 색깔이 옅은 가을 | 도원경 | 색 잃은 가을의 연구자·먹자국 담비 (667·660) | 410007033 |
| 105 | 참혹한 흔적의 겨울 | 도원경 | 참혹한 겨울의 연구자·묵날개 올빼미 (666·662) | 410007036 |
| 106 | 외곽 전투지역 | 아르테리아(퀸스로드) | 하이레프 보병·법사 (670·671) | 410007546 |
| 107 | 최하층 통로 | 아르테리아 | 하이레프 기갑병·장검병 (672·673) | 410007528 |
| 108 | 최상층 통로 | 아르테리아 | 얼어붙은 레프 보병·법사 (669·668) | 410007539 |
| 109 | 잔잔한 해안가 | 카르시온 | 절망의 파수꾼·어둠이 파고든 괴조·심연의 파수꾼 (678·676·677) | 410007622 |
| 110 | 어둠이 내리는 나무줄기 | 카르시온 | 절망의 사냥꾼·어둠이 스며든 흑표·심연의 사냥꾼 (681·679·680) | 410007643 |
| 111 | 숨이 멎어드는 동굴 | 카르시온 | 근원에 다다른 기사·근원에 다가간 사제 (674·675) | 410007661 |
| 112 | 재와 침묵의 땅 | 탈라하트 | 재를 먹는 사념·침묵을 삼키는 사념 (713·714) | 410013622 |
| 113 | 섭리와 운명의 전쟁터 | 탈라하트 | 영혼 도살자·침묵을 삼키는 사념 (715·714) | 410013640 |
| 114 | 심판과 운명의 전쟁터 | 탈라하트 | 폭풍의 심판자·영혼 도살자 (716·715) | 410013645 |
| 115 | 영원과 운명의 전쟁터 | 탈라하트 | 영원의 집행자·폭풍의 심판자 (717·716) | 410013646 |
| 116 | 밤의 길 | 탈라하트 | 밤의 인도자·환영의 인도자 (718·719) | 410013660 |
| 117 | 지하 1층 | 기어드락 | 순찰형·공격형 경비 로봇 (720·721) | 410013704 |
| 118 | 로봇 창고 | 기어드락 | 일반형 병기 로봇·공격형 경비 로봇 (722·721) | 410013710 |
| 119 | 강화 병기 창고 | 기어드락 | 강화형·일반형 병기 로봇 (723·722) | 410013714 |
| 120 | 고브의 작업실 | 기어드락 | 크로노스의 잔재·원념 (724·725) | 410013718 |

- 순서: 원본 그란디스 진행·레벨 순(호텔 아르크스 → 오디움 → 도원경 → 아르테리아 → 카르시온 → 탈라하트 Lv290~294 → 기어드락 Lv295~299). MonsterTable 레벨은 96 → 112로 이어진다.
- 112~120층은 원본 몬스터가 13종뿐이라, 층마다 그 원본 세부 지역의 새 몬스터 1종 + 바로 앞 지역 몬스터 1종으로 묶었다(113·114·115·118·119층). 119층 이름 "강화 병기 창고"는 원본 "로봇 창고 5~8"(강화형 병기 로봇 구역)을 구분하려고 붙였다.
- 새 몬스터 713~725: `Type` 밸런스형, `AtkType` = 원본 PADamage ≥ MADamage면 P(13종 모두 P), 사거리 1, 근접. 능력치는 층 번호 기준 공식(`StageMonsterLevel`)이라 원본 HP(2.8백억 등)는 쓰지 않는다. 원본 수치는 `monsters_manifest.json`이 아니라 `_root.xml`에 있다(레벨 290~299, PADamage 18,360~19,170).
- 필드보스(`FieldBossId`): 그란디스 후반은 원본 필드보스가 없어 기존 규칙(앞뒤 층 보스)대로 **712 리버스 다크소울**. `BossBgRUID`·`BossTileSetRUID` 빈칸.
- `TileSetRUID` 빈칸(탑 기본 바닥 타일).
- 배경·파노라마 RUID: `output/floor_bg/manifest.json`. 그림: `output/floor_bg/floorbg_*.png`·`floorpano_*.png`, 미리보기 `output/preview/floor_expansion_backgrounds.png`(98~111). SSD 사본: `수정한리소스/Map/Back/_Composed/`.
- `Docs/FloorPlan_Draft.csv`(층·원본 맵·몬스터 50줄)와 `Docs/Boss/field_boss_map.csv`(98~120층 23줄)에 반영했다.

## 5. 확인한 것 / 못 한 것

### 실제 확인 (메이커 플레이, 2026-11-25)

- 빌드 오류 0, 콘솔 `LEA-` 0건.
- 98층(오디움)·110층(카르시온): 합성 배경이 타일·영웅·몬스터 뒤, 전투 정상 → `output/verification/floor_exp_04_tower_floor98.png`, `floor_exp_05_tower_floor110.png`. 97층 재시작 시 `FloorBackdrop show=-`.
- 112층: 직접 올린 "재를 먹는 사념"이 원본 모습으로 서고 공격·피격·사망하며 웨이브가 10→1로 진행 → `floor_exp_07_custom_mob_8645460.png`, `floor_exp_08_custom_mob_8645460_b.png`.
- 116층(밤의 인도자, 공격 이펙트 확인)·120층(크로노스의 원념) 전투 → `floor_exp_09_tower_floor116.png`, `floor_exp_10_tower_floor120.png`.
- 로비 지역 선택: `count=120 sections=12 sec12=111-120`, 12구역 목록·파노라마·몬스터 그림 표시 → `floor_exp_11_region_select_118.png` (11구역은 `floor_exp_06_region_select_105.png`).
- 시험 맵 발판: `floor_exp_03_odium_backdrop_foothold.png`.

### 확인 못 함

- 새 층 끝까지 클리어(보스 스테이지 712, 별·결과·첫 클리어 보상), 113~115·117~119층 각각의 화면, 직접 올린 몬스터 13종 전부의 사망 동작 화면.
- 오프라인 보상 120층 계정, 모바일 화면 비율, 보스 탭.

## 6. 막힌 것

- **스파인 배경 재현**: 바이너리 `.skel` 뼈대 변환을 못 읽어 112~120층 배경은 근사다.
- **몬스터 소리**: mp3 → ogg 변환 도구가 없어 빈칸.

## 7. 남은 문제·나중에 끝에 붙일 후보

1. **나중에 끝에 붙일 후보 (121층~)** — 중간에 끼우지 않는다(저장 최고 층·별·`BossTable.UnlockFloor`·`AccountRules` 층 기준이 어긋남).
   - 묶음 B: 무릉도원 하늘 숲(`25001xxxx`, 14종), 루디브리엄 시계탑 최하층·히든(`2200600xx`·`2200700xx`, 19종), 엘나스 폐광(`2110415xx~`, 4종)·미나르숲(2종). 몬스터·배경 템플릿이 다 있어 데이터만 넣으면 된다. 원본 레벨이 낮아 "회상" 성격.
   - 묶음 C: 여우고개·청운골·리스토니아·툴렌시티·라라 지역·에리모스(43종). MSW 팩 → MonsterTable 새 행(stand·attack1·hit1·die1·move, Damage·Die 소리)이 필요. 배경은 메이커 목록(에리모스는 합성).
2. 112~120층 몬스터가 크다(원본 크기). 필요하면 몬스터 쪽 크기 배율로 줄인다(전투 코드 담당과 상의).
3. 새 층 바닥 타일은 탑 기본 풀밭이다.
4. 12구역부터 구역 누적 보상 기준(10·20·30별)은 기존과 같다(이제 10층이 다 차 있어 채울 수 있음).

## 8. 바꾼 파일

| 파일 | 내용 |
|---|---|
| `RootDesk/MyDesk/Data/FloorTable.csv` | 98~120층 23줄 추가 |
| `RootDesk/MyDesk/Data/MonsterTable.csv` | 713~725 탈라하트·기어드락 13줄 추가 |
| `RootDesk/MyDesk/Tower/FloorBackdrop.mlua` (+ `.codeblock`) | 새 컴포넌트: `sprite:` 배경을 카메라 화면에 덮어 표시 |
| `RootDesk/MyDesk/Tower/TowerRunManager.mlua` | `ApplyFloorLook`에서 `sprite:`면 템플릿 교체를 건너뜀(4줄). 같은 파일의 스킬 V2 변경은 스킬 담당 작업 |
| `RootDesk/MyDesk/Game/ScreenFadeLogic.mlua` | `LookReady`에서 `sprite:` 배경은 미리 불러오기만 확인(3줄) |
| `map/OrbisTower.map` | `FloorBackdrop` 엔티티 1개 추가 |
| `map/FloorLookTest.map`, `RootDesk/MyDesk/Models/Terrain/FloorFoothold.model` | 시험 전용 |
| `Docs/FloorPlan_Draft.csv`, `Docs/Boss/field_boss_map.csv` | 98~120층 반영 |
| `output/floor_bg/` | 배경·파노라마 PNG 46장, `manifest.json`, `monsters_manifest.json`, `tools/`(색인·합성·몬스터 업로드 스크립트) |
| `output/preview/floor_expansion_backgrounds.png`, `floor_expansion_custom_monsters.png`, `output/verification/floor_exp_*.png` | 미리보기·확인 스크린샷 |
| SSD `수정한리소스/` | 합성 배경 23장, 몬스터 프레임 수정본 1장, README 기록 |

### 다시 만들 때

- 배경(일반 back): `python3 output/floor_bg/tools/render_back.py <원본맵ID>` / 스파인 back: `render_spine_approx.py <원본맵ID>` → 메이커 `upload_sprite "<png>" -subcategory background -json` → `FloorTable.BgRUID = sprite:<RUID>`, `PanoRUID`(1296x293 띠).
- 몬스터: `mob_batch.py <묶음이름> <원본ID...>` → `upload_sprites mobup/sprites_<묶음>.json -json` → `grab.py` 로 결과 추출 → `mob_clips.py <묶음>` → `upload_animationclips mobup/clips_<묶음>.json -json` → MonsterTable 행.
