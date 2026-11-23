# 원본 메이플 맵 가져오기 시험 (2026-11-24)

## 결론

- **가능.** 메이커의 `maker_import_maplestory_map`(원본 9자리 맵 ID)으로 원본 필드 맵을 MSW 맵에 통째로 가져올 수 있다.
- 배경·바닥 타일·발판(풋홀드)·밧줄·장식 오브젝트·포털·NPC·**일반 몬스터 배치**까지 함께 온다. 가져온 맵에서 캐릭터가 발판 위에 선다(`IsOnGround = true`).
- 단, **필드보스(머쉬맘·스텀피 등)는 따라오지 않는다.** 최신 원본 데이터에서 필드보스는 맵의 `life` 목록에 없다(서버 쪽에서 따로 소환하는 방식). 보스는 우리 코드로 직접 소환해야 한다.

## 사용한 시험 맵

| 항목 | 값 |
|---|---|
| 시험 맵 파일 | `map/MapImportTest.map` (MapBuilder `fromTemplate("maple")`로 새로 만듦, 기존 맵은 손대지 않음) |
| 가져온 원본 맵 | `100020101` 머쉬맘의 오솔길 (버섯노래숲, 메이커 목록 이름 `Mushmom Forest Trail`) |
| SectorConfig 등록 | 하지 않음. 등록 없이도 플레이 중 `_TeleportService:TeleportToMapPosition(..., "MapImportTest")` 이동이 됐다 |
| 저장 | 시험 맵에만 `save` |
| 끝난 상태 | 편집 모드, 현재 맵 `Lobby` |
| 스크린샷 | `output/verification/map_import_test_100020101_play.png`, `output/verification/map_import_test_100020101_play2.png` |

## 가져온 결과 (저장된 시험 맵 분석)

| 항목 | 결과 |
|---|---|
| TileMapMode | `0` (MapleTile) — 우리 탑 맵 `OrbisTower`와 같다 |
| 배경 | `Background` 엔티티 1개, `BackgroundComponent.TemplateRUID = e9693e2f342347bfadab2b24f4fbc001` (원본 맵별 배경 템플릿). 화면에 하늘·산·마을 원경이 정상 표시 |
| 바닥 타일 | `TileMap` ~ `TileMap_7` 8개 (MapLayer0~7), 타일셋 RUID 예: `9dfea3808bbd49a5877d8624df21b1c7` |
| 발판 | 맵 루트 `FootholdComponent`에 풋홀드 50개. 범위 x −17.94 ~ −0.06, y −0.30 ~ 4.15 (가로 약 18 유닛) |
| 밧줄·사다리 | `ClimbableComponent` 엔티티 27개 |
| 장식 오브젝트 | `SpriteRendererComponent` 엔티티 36개 (버섯 집, 꽃, 나무 등) |
| 몬스터 | 20마리 (`monster-2416`…, 모델 `movemonster`, 원본 life의 `1110100` 초록버섯 20개와 일치). StateAnimation ActionSheet(stand/move/hit/die) RUID가 채워져 있다 |
| NPC / 포털 | NPC 1개(`staticnpc`), 포털 1개(`portal`) |
| 시작 위치 | `SpawnLocation` 이름의 엔티티가 4개 생김 (원래 템플릿 것 + 가져오기로 생긴 것, 같은 이름) |

### 플레이 확인 (짧게 1회)

- 로비에서 시험 맵으로 순간이동 → 서버 로그 `[VRF] map=MapImportTest pos=(-10.860, -2.149, 0.000) onGround=true mobs=20 npcs=1`.
- `LEA-` 오류 0건, `Error` 0건(런타임 콘솔).
- 빌드 로그에는 가져오는 순간 `LEA-4005 BoxSize HitComponent` 20건(몬스터마다), `LEA-4005 BoxSize PortalComponent` 1건이 찍혔다. 가져온 몬스터·포털의 충돌 상자 값 문제로 보인다.

## 한계 / 남은 문제

1. **필드보스 없음**: 원본 `Map/Map/Map/Map*.xml` 전체의 `life`를 검색한 결과 머쉬맘(6130101)·좀비 머쉬맘·스텀피(3220000)·주니어 발록(8130100) 등은 한 번도 나오지 않는다. 필드보스 ID가 life에 남아 있는 맵은 `103030300` 보이지 않는 위험(다일) 정도뿐이다. → 보스 몬스터는 기존 몬스터 소환 방식(`SpawnByModelId`)으로 따로 넣어야 한다.
2. **가져온 몬스터는 움직이지 않는다**: `movemonster` 인스턴스에 AI 컴포넌트가 없다(두 장의 스크린샷에서 위치 동일). 우리 게임은 칸 전투라 어차피 쓰지 않으므로 보스 층에서는 지우거나 비활성으로 둬야 한다.
3. **캐릭터가 화면에 안 보였다**: 위치·바닥 판정은 정상인데 스크린샷에 캐릭터가 보이지 않았다. 캐릭터는 `MapLayer3`이고 가져온 타일·장식이 MapLayer0~7에 나뉘어 있어, 앞쪽 레이어 타일·장식에 가려졌을 가능성이 크다(확인 못 함). 따로 쓰는 맵이 되면 정렬 레이어를 맞춰야 한다.
4. **위치 기준이 다르다**: 가져온 맵은 원본 좌표를 그대로 써서 x가 −18 ~ 0 쪽에 치우쳐 있다. 우리 탑 맵은 칸 전투 위치(`MonsterPos = (-0.4, -0.04)`)와 `BattleCamera`가 원점 주변 기준이다.
5. **목록 이름은 영어**: `maker_list_maplestory_maps`는 영어 이름으로만 검색된다(`머쉬맘` 검색 0건, `Mushmom` 4건). 원본 ID로 검색하는 편이 정확하다.
6. 서버에서 플레이어 위치를 바꿔도 클라이언트 위치는 그대로였다(플레이어 이동은 클라이언트 기준). 시험용 이동은 `TeleportToMapPosition`을 써야 한다.

## 필드보스 관련 맵이 메이커 목록에 있는지

| 원본 ID | 원본 이름 | 메이커 목록 | 비고 |
|---|---|---|---|
| 100020101 | 머쉬맘의 오솔길 | 있음 (`Mushmom Forest Trail`) | 이번 시험 맵. life는 초록버섯 20 + NPC 1 |
| 100020301 | 블루머쉬맘의 숲 | 있음 (`Blue Mushmom Forest`) | |
| 105070002 | 머쉬맘의 무덤 | **없음** | |
| 910080020 / 910090210 | 좀비 머쉬맘 관련 히든맵 | 있음 | |
| 105100300 / 105100301 | 발록의 무덤 / 발록이 사라진 자리 | 있음 | |
| 103030300 | 보이지 않는 위험 | 있음 (`Unseen Danger`) | life에 다일(6220000)이 남아 있는 유일한 확인 사례 |
| 240040401 | 레비아탄의 협곡 | 있음 | |
| 200010303 | 엘리쟈의 정원 | **없음** | |
| (이름 검색) 스텀피 | — | 맵 이름에 스텀피가 들어간 필드 맵 없음 (`Stump` 검색은 다른 맵 2개) | |
| (이름 검색) 엘나스 | — | `El Nath` 검색은 마을·상점 4개만 나옴 | 엘나스 사냥터는 영어 지역명으로 따로 검색 필요 |

- 우리 층 계획(`Docs/FloorPlan_Draft.md` v6, 97층)은 2026-11-16에 이미 "메이커 목록에 있는 맵만" 기준으로 걸렀다. 그때 빠진 지역(오디움·도원경·아르테리아·숨이 멎어드는 동굴·잔잔한 해안가·어둠이 내리는 나무줄기·밤의 길·섭리와 운명의 전쟁터·재와 침묵의 땅·기어드락)은 이번에도 목록에 없다(`Odium` 검색 0건으로 재확인).

## 원본 XML을 직접 변환해 넣는 방식 평가 (시도하지 않음)

- **발판(foothold)**: 원본 XML의 선분(x1,y1,x2,y2, prev/next)을 1/100 해서 `FootholdComponent`에 넣으면 되므로 변환 자체는 쉽다. 다만 MapBuilder는 풋홀드 추가·연결 기능이 없다(빌더 비지원 영역) → `.map` 원문 직접 수정이 필요해 위험하다.
- **배경(back)**: 원본은 배경 조각 수십 개(bS·no·x·y·rx·ry·type)로 이뤄진다. MSW에서는 원본 맵 단위 배경 템플릿 RUID로 들어오므로, 조각을 하나하나 옮기려면 조각별 스프라이트 RUID 검색 + 시차 스크롤 재현이 필요하다. 현실적이지 않다.
- **타일·오브젝트**: 조각 수백 개를 RUID로 맞춰야 한다. 마찬가지로 비현실적.
- → **메이커 가져오기 도구가 있는 맵은 가져오기로, 목록에 없는 맵은 쓰지 않는 것**이 맞다. 목록에 없는 지역이 꼭 필요하면 "비슷한 배경 템플릿 + 우리 탑 맵 발판"으로 대체한다.

## 우리 탑 구조와 보스 층 연결 방법

### 현재 구조

- 탑 전체가 **맵 하나(`OrbisTower`)** 다. 층이 바뀌면 `TowerRunManager`가 `FloorTable.csv`의 `BgRUID`로 `Background.BackgroundComponent:ChangeBackgroundByTemplateRUID(bg)`를 부르고, `TileSetRUID`는 `FloorTileSet`(@Sync)로 넘겨 `BattleHUDLogic`(클라이언트)이 `TileMap.TileMapComponent.TileSetRUID`를 바꾼다. 타일 배치·발판은 그대로다.
- 전투는 칸 기반(`MonsterPos`, `CellWidth`, `SpawnByModelId(..., self.Entity)`), 카메라는 `BattleCamera`.
- 맵 이동은 로비 `LobbyManager`의 `TeleportToMapPosition(player, TowerPos, "OrbisTower")` 하나뿐.

### 권장 방식 (A안: 배경 템플릿만 빌려 쓰기)

1. 보스 층도 `OrbisTower` 한 맵에서 진행한다.
2. 가져오기로 얻은 **배경 템플릿 RUID / 타일셋 RUID만** 층 데이터(`FloorTable.csv`의 `BgRUID`, `TileSetRUID`)나 새 보스 표에 적는다. 예: 머쉬맘의 오솔길 배경 `e9693e2f342347bfadab2b24f4fbc001`, 타일셋 `9dfea3808bbd49a5877d8624df21b1c7`.
3. 보스 몬스터는 기존 칸 전투 소환 경로로 넣는다(`MonsterTable` 행 + `SpawnByModelId`).
- 장점: 맵 등록·카메라·스폰 위치·HUD·저장 흐름을 전혀 바꾸지 않는다. 이미 층마다 배경을 바꾸는 방식이라 코드 변경이 가장 적다.
- 시험 맵 `MapImportTest`는 RUID를 찾아내는 "작업대"로만 쓴다(다른 보스 맵도 같은 방법으로 가져와서 RUID만 기록).

### B안: 가져온 맵을 그대로 보스 전용 맵으로 쓰기 (권장하지 않음)

필요한 일:
- **맵 등록**: `map/<보스맵>.map`으로 저장, 필요하면 `Global/SectorConfig.config`에 `map://<이름>` 추가(이번 시험에서는 등록 없이도 이동은 됐다).
- **층 데이터 연결**: 보스 층에서 `TeleportToMapPosition`으로 그 맵으로 보내고, 끝나면 탑으로 되돌리는 흐름 추가. `TowerRunManager`는 탑 맵 루트에 붙은 컴포넌트라 보스 맵 루트에도 붙이거나 판 상태를 넘겨야 한다(층 진행·보상·저장이 맵 단위로 나뉘는 문제).
- **카메라·경계**: 가져온 맵은 x −18~0으로 치우쳐 있고 `BattleCamera`가 없다. 전투 카메라 엔티티와 칸 위치(`MonsterPos`)를 맵마다 따로 맞춰야 한다.
- **스폰 위치**: 이름이 같은 `SpawnLocation` 4개 정리, 칸 전투용 아군·보스 위치를 발판 높이에 맞춰 다시 잡기.
- **정리**: 가져온 일반 몬스터 20마리·NPC·포털 제거, `LEA-4005` 충돌 상자 오류 정리, 캐릭터가 가려지는 정렬 레이어 확인.
- 맵마다 이 작업이 반복되므로 보스 수가 늘수록 비용이 커진다.
