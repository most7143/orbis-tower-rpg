# 지역 선택 · 클리어 결과 UI 구현 전달

작성일: 2026-11-13  
기준 해상도: 1920×1080, 가로 화면  
리소스 작업본: `output/ui/region_select_result/`  
배포용 폴더: `/Users/seogeon-u/Documents/메이플월드_리소스/지역선택_결과/`

## 1. 이번 확정 사항

- 최근 메인화면·직업선택과 같은 따뜻한 금색·크림색 프레임, 보라 패널, 파랑 주요 버튼을 사용한다.
- **목록은 지역 이름 중심으로 표시한다.** 층 번호는 화면에서 빼되, 기존 Floor ID·저장 키·전투 진입 ID는 유지한다.
- 현재 데이터에 별도 광역 지역 그룹 표는 없다. 지금은 `FloorTable.FloorName`을 지역 표시명으로 쓴다. 10개씩 묶었다고 같은 지리적 지역으로 추정하지 않는다.
- 선택만으로 입장하거나 에너지를 사용하지 않는다. 별도의 입장 버튼이 있다.
- 결과의 **전투 보상은 전체 가로 폭을 사용**한다. 아이템 수가 많으면 가로 스크롤한다.
- 별도의 **신규 별 재화 보상 패널은 없다.** “신규 별 +1”은 이번에 추가된 별을 알리는 정보 배지다.
- 구간 보상은 **상자를 눌러 직접 수령**한다. 기존 인계서의 자동 신규 별 보상 지급 제안보다 이 결정이 우선한다.
- 상태 문구·지역명·숫자·아이템명·비용·제목을 포함한 모든 UI 글자는 엔진 텍스트다.
- PNG를 한 장짜리 화면으로 붙이지 말고, 프레임·아이콘·버튼·텍스트를 분리해서 조립한다.

이 패키지는 이미지와 구현 기준을 제공한다. 이번 작업에서는 `.ui` / `.mlua` / 저장 데이터 / 보상 로직을 변경하지 않았다.

## 2. 폴더와 파일

런타임용 PNG는 루트에 모았다. 하위 폴더는 두 개다.

- `Preview/`: 승인 참고 이미지, 실제 리소스 조립 화면, 텍스트 없는 조립 화면, 전체 리소스 및 버튼 상태표. **Maker에 통째로 등록하는 배경이 아니다.**
- `Docs/Implementation.md`: 이 문서.
- `resources.json`: 82개 PNG의 크기·역할·중앙 피벗·출처·체크섬.
- `layout_1920x1080.json`: 개별 요소의 좌상단 사각형, 텍스트 크기·색상·정렬. 예시 데이터가 포함된다.
- `text_styles.json`: 엔진 폰트·외곽선·그림자 및 역할별 크기.
- `button_states.json`: 버튼 이미지 상태 연결과 지역 선택 유지 규칙.
- `sources.json`: 기존 MSW 리소스와 재사용한 프로젝트 이미지의 출처.
- `verification.json`: PNG·상태·배치 파일 검증 결과.
- `generation.json`: 이미지 제작 방식과 프롬프트 기록.

`Preview/PREVIEW_approved_*.png`는 확정한 시각적 참고다.  
`Preview/PREVIEW_*_engine_text_1920x1080.png`는 납품 PNG를 직접 조립한 확인용 화면이다. 이 확인용 글자는 시스템 폰트로 근사했고, 실제 게임에서는 아래 Maple 설정을 사용한다.

## 3. 텍스트 설정

모든 글자: `TextGUIRendererComponent`.

기본값:

| 속성 | 값 |
|---|---|
| Font | 프로젝트에서 사용하는 `Maple` |
| FontStyle | `FontStyleType.Bold` = 1 |
| FontColor | `#FFF3D7` |
| OutlineColor | `#251C36` |
| OutlineWidth | 0.12 |
| FaceDilate / FaceSoftness | 0 / 0 |
| Underlay | true |
| UnderlayColor | `#080B1ABF` |
| UnderlayOffsetX / Y | 0.10 / -0.16 |
| UnderlayDilate / Softness | 0.04 / 0.06 |
| BestFit / SizeFit | false / false |
| IsRichText | false; 실제 필요할 때만 켠다 |

외곽선·그림자 수치는 픽셀 두께가 아니라 엔진 재질 파라미터다. 로컬 NativeScripts의 속성명을 확인한 시작값이며, Maker 화면에서 최종 조정한다. 기존 메인화면과 다른 폰트로 자동 대체하지 않는다.

| 용도 | 글자 크기 | 색상 | 정렬 |
|---|---:|---|---|
| 화면 제목 | 56 | `#FFF3D7` | 가운데 |
| 결과 제목 | 64 | `#FFE5A6` | 가운데 |
| 지역 선택 제목 바 | 38 | `#482813` | 가운데 |
| 우측 지역명 | 42 | `#FFF3D7` | 왼쪽 |
| 지역 목록 이름 | 28 | `#FFF3D7` | 왼쪽 |
| 패널 소제목 | 28 | `#FFF3D7` | 왼쪽 |
| 미션 | 선택 25 / 결과 28 | `#FFF3D7` | 왼쪽 |
| 달성 | 23~25 | `#A4EBAF` | 오른쪽 |
| 새로 달성 | 25 | `#FFDD80` | 오른쪽 |
| 미달성·기존 달성 | 23~25 | `#B7AFCA` | 오른쪽 |
| 입장 버튼 | 54 | `#FFF3D7` | 가운데 |
| 결과 하단 버튼 | 36~40 | `#FFF3D7` | 가운데 |
| 보조 버튼 | 28~30 | `#FFF3D7` | 가운데 |
| 비용 | 22~26 | `#FFF3D7` | 가운데 |
| 아이템명 | 22 | `#FFF3D7` | 가운데 |
| 안내·회복 시간 | 21~23 | `#9F94B2` 또는 `#C5C3D2` | 상황별 |
| 크림색 바탕의 제목·수령 버튼 | 22~28 | `#482813` | 가운데 |

크림색 바탕 위 글자는 외곽선·그림자를 끈다. 결과 제목은 외곽선 `#432912`, 두께 0.15를 시작값으로 쓴다.

버튼 글자 색상:

| 상태 | 색상 |
|---|---|
| 기본 | `#FFF3D7` |
| 하이라이트 | `#FFFFFF` |
| 클릭 | `#FFE2A0` |
| 비활성 | `#A6A3AE` |
| 지역 선택 유지 | `#FFF3D7` |

지역명은 한 줄 영역을 충분히 확보한다. 임의 줄바꿈이나 축소로 행 높이를 바꾸지 말고, 긴 이름은 엔진 말줄임 정책을 적용한다. 원문은 상세 영역에서 볼 수 있게 한다. 별·자물쇠·체크·번개는 텍스트 글리프가 아니라 제공된 PNG다.

## 4. 버튼 이미지

기본 4상태의 접미사는 `normal / highlight / pressed / disabled`다.

| family | PNG 크기 | 용도 |
|---|---|---|
| primary | 512×160 | 입장, 다시 도전 |
| secondary | 448×128 | 메인으로, 현재 도전 지역으로, 지역 선택으로 |
| arrow | 96×96 | 지역 이전/다음, 필요한 목록 이동 |
| claim | 176×64 | 보상 받기의 금색 글자 바탕 |
| milestone | 176×128 | 결과 화면 구간 보상 카드 |
| milestone_compact | 128×112 | 지역 선택 하단의 상자와 숫자 표시 |
| region_row | 544×80 | 지역 목록 행 |
| scrollbar_thumb | 16×80 | 스크롤 손잡이 색상 상태 |

파일 예: `button_primary_normal_512x160.png`.

기본 4상태는 같은 캔버스·알파 경계·피벗을 유지한다. 비활성은 회색으로 확실히 구별한다. 이미 회색인 PNG 위에 전체 알파를 한 번 더 낮추지 않는다. 클릭 상태를 만들기 위해 부모 위치를 움직이지 않는다.

지역 행에만 다음 유지 상태가 추가된다.

- `button_region_row_selected_544x80.png`
- `button_region_row_selected_highlight_544x80.png`
- `button_region_row_selected_pressed_544x80.png`

즉 **35장 = 8종 × 4상태 + 지역 선택 유지 3장**이다.

### 기본 버튼의 MSW 연결

- `ButtonComponent.Transition = TransitionType.SpriteSwap` (2).
- 기본 이미지는 `SpriteGUIRendererComponent.ImageRUID`에 넣는다.
- `ImageRUIDs.HighlightedSprite` → highlight.
- `ImageRUIDs.PressedSprite` → pressed.
- `ImageRUIDs.DisabledSprite` → disabled.
- 일반 동작 버튼은 `Selectable = false`. `SelectedSprite`가 남아 클릭 상태를 유지하는 것처럼 보이지 않게 한다.
- SpriteGUI 색은 흰색, 알파 1. 추가 ColorTint로 이미지를 재착색하지 않는다.
- 아이콘·글자·장식은 같은 버튼의 자식이며, 장식 SpriteGUI는 `RaycastTarget = false`.

### 지역의 논리 선택

현재 선택 지역은 마우스 호버나 키보드 포커스와 다른 상태다. 기존 UI 컨트롤러가 논리 선택을 별도 보관하게 한다.

이 행을 직접 관리하는 컨트롤러에서는 `Transition = None`, `Selectable = false`로 두고, 입력 상태와 선택 여부를 `button_states.json`의 조합표에 대입해 이미지를 바꾼다. 엔진 SpriteSwap과 수동 ImageRUID 설정을 동시에 돌리지 않는다.

선택된 행의 기본/호버/누름은 각각 selected / selected_highlight / selected_pressed다. `icon_check_selected_40x40.png`도 선택 상태를 보조한다.

기존 컨트롤러 구조상 별도 강조 레이어가 필요할 때만 `overlay_region_selected_544x80.png`를 사용한다. 이미 선택 프레임이 들어 있는 selected PNG 위에 다시 쌓지 않는다.

잠긴 지역도 미리보기를 허용한다. 이때 행 전체를 비활성 처리하지 말고 잠금 아이콘을 표시하며, **입장 버튼만 비활성**으로 둔다. row disabled는 실제 입력 차단·로딩 등에 쓴다.

### 상자 수령 버튼

결과의 상자 카드 전체를 하나의 클릭 영역으로 사용한다. 금색 claim 바탕과 문구는 카드의 자식이며 별도의 중복 클릭 이벤트를 만들지 않는다.

카드 입력 상태가 바뀌면 부모 milestone 이미지와 자식 claim 바탕의 4상태를 함께 연결할 수 있다. “수령 완료”는 disabled 이미지와 체크, “잠김”은 disabled 이미지와 잠금으로 구분한다. 받을 수 있을 때만 `overlay_reward_ready_176x128.png`를 올린다.

화살표·보조 버튼 등 시각 크기가 작아져도 실제 터치 영역은 최소 88×88을 확보한다. 스크롤 손잡이 그림의 폭은 16이지만 드래그 입력 영역은 더 넓게 만든다.

## 5. 이미지 배치·피벗·9-slice

- 모든 PNG 피벗은 **(0.5, 0.5)**다.
- `layout_1920x1080.json`의 rect는 화면 좌상단 기준 `[x,y,width,height]`다.
- 루트 중앙 앵커를 쓸 때: `anchoredPosition = (x+w/2-960, 540-y-h/2)`.
- 이 공식은 루트 좌표 기준이다. 자식은 부모 중심을 빼서 변환한다.
- `UITransformComponent.Position`을 직접 쓰지 않는다.
- PNG 원본 크기와 표시 크기가 다른 경우 UI 프레임은 사각형 전체를 표시한다. 원본 비율을 유지하면서 **잘라내는 cover 방식은 금지**한다.
- 아이템·스티지·상자 아이콘은 비율을 유지하고 가운데 배치한다. PNG의 투명 여백을 임의로 잘라 피벗을 바꾸지 않는다.
- 기본 납품은 목표 크기로 만든 PNG다. 우선 `SpriteGUIRendererComponent.Type = ImageType.Simple`로 정확히 표시한다.
- `resources.json.sliceBorder`는 추후 등록용 권장 픽셀 여백이다. **MSW 리소스에 9-slice 메타데이터가 등록됐다는 뜻이 아니다.** 메타데이터를 등록하지 않고 Sliced로 바꾸지 않는다.
- 전투 보상·미션·지역 행 등 가변 요소의 글자와 아이콘은 패널에 합성하지 않는다.
- 외곽 배경은 실제 게임/로비 배경을 표시하고 엔진의 dim을 적용한다. `overlay_dim_1x1.png`는 필요할 때 화면 크기로 늘려 쓴다.
- 지역 미리보기는 실제 배경을 넣는다. PNG로 그린 가짜 지역을 새 데이터로 사용하지 않는다. 패키지 Preview의 숲 이미지는 시안 확인용이다.
- `panel_panorama_frame_992x240.png`를 바탕으로 두고 안쪽에 배경 이미지를 배치한다. 모서리는 `mask_panorama_972x220.png` 또는 기존 엔진 마스크 구조로 처리한다.

별 게이지는 track 위에 fill을 별도 배치한다. fill은 `Type = Filled`, `FillMethod = Horizontal`, `FillOrigin = 0`, `FillAmount = clamp(stars / target, 0, 1)`. 레이아웃의 fill 노드에는 확인용으로 잘린 rect와 엔진에서 쓸 **fullRect**가 함께 있다. 실제 fill 이미지 크기는 fullRect로 고정하고 FillAmount만 바꾼다.

## 6. 지역 선택 화면

권장 계층:

    RegionSelectRoot
      Header
        BackButton / Title / Energy
      RegionListPanel
        TitlePlaque / RegionNavigation
        ScrollViewport
          Content
            RegionRow_{id}
              Name / Star1..3 / StatusIcon
        Scrollbar / CurrentRegionButton
      RegionDetailPanel
        Panorama / RegionTitle / CumulativeStars
        Monsters
        Missions
        ClearRewards / PossibleDrops
      SectionRewards
        Counter / Gauge / MilestoneButtons
      EnterButton

- 목록 항목 수와 정렬은 현재 데이터에서 가져온다. 5개는 배치 예시일 뿐 고정 개수가 아니다.
- 선택된 지역·최고 도전 가능 지역·클리어 여부는 별도 데이터다.
- 선택 시 우측 상세 정보만 바뀐다. 잠금 지역의 미리보기에서도 에너지가 차감되지 않는다.
- “현재 도전 지역으로”는 대상 항목을 선택하고 스크롤 위치를 맞춘다.
- 우측 몬스터는 `FloorTable.MonsterId`로 `MonsterTable`에서 읽는다. 예시 스티지는 ID 21이다. 임의 몬스터를 추가하지 않는다.
- 미션 3개: 클리어 / 제한 시간 내 클리어 / 체력 50% 이상으로 클리어. 시간 기준은 미정이므로 임의 숫자를 확정하지 않는다.
- 클리어 보상과 획득 가능 아이템을 구분한다. 아이템명·수량·확률은 실제 서버 데이터에서 가져온다.
- 입장 버튼은 에너지 3을 별도 문구로 표시한다. 부족/잠김/요청 중은 disabled 상태다.
- 30 최대 에너지, 10분마다 1 회복, 오프라인 회복은 기존 시스템 기준을 따른다. UI 제작이 에너지 로직을 새로 만드는 작업은 아니다.

## 7. 결과 화면

- 큰 별은 **누적 달성**, 미션 행은 **이번 판 결과와 기존 달성**을 구분한다.
- 기존에 얻은 별은 이번 판에서 조건을 놓쳐도 사라지지 않는다.
- 새로 얻은 별만 new 아이콘/글로우를 쓴다. 기존 별의 피벗과 표시 중심은 같다.
- 신규 별 0개일 때는 “신규 별 +0” 배지를 숨기는 것을 기본으로 한다.
- 전투 보상 영역은 화면 전체 폭을 사용한다. 확인용 화면의 6칸 중 빈 칸은 수용량을 보여주는 예시다. 런타임에서는 **실제로 받은 보상만** 만든다.
- 아이템이 많으면 가로 스크롤하고 부모 viewport에서 마스킹한다. 고정 3종으로 제한하거나 제목을 줄여 공간을 만들지 않는다.
- 수량·아이템명은 엔진 텍스트로 붙인다. 결과 재오픈으로 보상을 다시 지급하지 않는다.
- 별 개별 보상 섹션은 만들지 않는다. 구간 보상은 아래의 별도 수령 영역에서만 처리한다.
- “지역 선택으로”는 단순 로비 복귀와 다르다. 복귀 후 선택 화면을 열고 방금 지역의 상세·선택·스크롤 위치를 복원해야 한다.
- 재도전은 기존 로직을 재사용하고 에너지 3을 표시한다.
- 실패/중단 결과도 같은 패널을 재사용하되 결과 제목과 항목을 바꾸고 새 별/보상을 임의로 지급하지 않는다.

## 8. 구간 보상 직접 수령

10 / 20 / 30은 시안의 표시 예시다. 실제 구간 ID·필요 별·보상 내용은 서버 설정으로 공급한다. 표시 지역명과 구간 ID가 같다고 추정하지 않는다.

1. 필요 별 미달 → 잠금 상자, lock, disabled.
2. 필요 별 달성 + 미수령 → 금색 상자, 준비 강조, “보상 받기”, 입력 가능.
3. 수령 요청 중 → 입력 잠금, 기존 화면 유지, 중복 요청 차단.
4. 서버 성공 응답 → 저장된 수령 상태로 갱신, 회색 상자+체크, “수령 완료”.
5. 실패 응답 → 실제 상태를 다시 읽고 재시도 가능 상태 복원.

판정·지급·수령 플래그 저장은 서버가 담당한다. 최소한 플레이어/구간/보상 단계로 중복 수령을 막는다. 클라이언트의 별 표시나 PNG 상태만 보고 지급하지 않는다. 결과를 다시 열거나 접속을 다시 해도 수령 완료가 유지돼야 한다.

## 9. 기존 RUID와 신규 PNG

`sources.json`에 기존 RUID를 기록했다.

| 역할 | 기존 RUID |
|---|---|
| 스티지 애니메이션 | `25152ef859744489b7e34a993b3a3c6d` |
| 책 모양 스킬북 예시 | `00c119f6f437479d9b0f95edfb29e870` |
| 뻣뻣한 깃털 예시 | `8a668406667d48e89c0b29755bd3ebed` |
| 매직박스 | `20724aa0526d4f6cba6a9695a07e1126` |
| 무기 슬롯 주문서 2단계 | `583f9e860ca446febffdb2032877cf1c` |

- 스티지를 정지 아이콘으로 보여줄 때는 기존 clip RUID의 정적 썸네일(`thumbnail://` 접두사) 또는 제공한 정지 PNG를 쓴다.
- 책/깃털/주문서는 배치용 예시다. 실제 보상 아이템 ID와 아이콘 매핑이 있으면 그 데이터를 우선한다.
- 패딩·크기·색을 변경한 납품 PNG에 원본 RUID가 자동으로 부여되는 것은 아니다.
- 원본을 그대로 표시할 수 있는 경우 기존 RUID를 재사용하고, 새 PNG를 쓰려면 Maker에 등록해 새 RUID를 연결한다.
- `resources.json.runtimeRUID`는 아직 null이다. 가짜 RUID를 만들지 않았고 기존 리소스의 바이너리를 덮어쓰지 않았다.
- 신규 PNG는 정적 sprite다. 반복 애니메이션 클립으로 등록할 필요가 없다.
- 픽셀 아이템은 원본 픽셀과 명암을 유지하고 정수배 확대했다. 임의 smoothing을 더하지 않는다.

## 10. 구현·검증 인계

기존 참고 경로:

- `ui/LobbyUI.ui`, `RootDesk/MyDesk/Lobby/LobbyUILogic.mlua`
- `ui/ResultPopup.ui`, `RootDesk/MyDesk/Tower/ResultPopupLogic.mlua`
- `RootDesk/MyDesk/Tower/TowerRunManager.mlua`
- `RootDesk/MyDesk/Data/FloorTable.csv`, `MonsterTable.csv`, `SlotScrollTable.csv`

실제 구현할 에이전트는 현재 프로젝트 지침과 MSW UIBuilder를 사용한다. `.ui`의 JSON을 직접 수정하지 않는다. 이번 파일 패키지를 만들었다고 런타임 구현이 끝난 것은 아니다.

Maker 검증 항목:

- 목록 실제 휠·드래그, 긴 목록, 현재 도전 지역 이동.
- 선택/호버/누름/비활성/선택 유지의 상태 구분과 피벗 안정성.
- 잠긴 지역 미리보기와 입장 차단, 에너지 부족·연타·로딩.
- 보상 0개/1개/많은 경우의 가로 스크롤과 마스크.
- 신규 별 없음/있음, 기존 별 보존, 실패·중단 결과.
- 구간 보상 잠김/받기 가능/수령 완료/요청 실패, 재접속·중복 요청.
- 작은 화면 글자, 실제 Maple 폰트, 버튼 안쪽 여백.
- 상단 오른쪽 MSW 내장 UI와 안전 영역 간섭.
- UI 코드 변경 후 Refresh 및 실제 실행·로그로 확인.

이 패키지의 파일·알파·상태·배치는 로컬에서 검사했다. Maker 실행·입력·서버 지급·저장은 아직 검증하지 않았다.
