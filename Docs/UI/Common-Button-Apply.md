# 공용 버튼 적용 표

작성일: 2026-11-24

근거 자료: `/Volumes/P7/MapleStoryWorld/수정한리소스/common_buttons/IMPLEMENTATION.md`, CLAUDE.md의 "공용 버튼" 절.
RUID 목록: `output/ui/common_buttons/manifest.json`. 이 파일에는 27장의 RUID가 있고, 비활성 상태는 normal RUID를 다시 쓴다. SSD 원본 manifest에 RUID만 채운 사본은 `manifest_with_ruid.json`이다.

## 색 규칙
- **초록**: 일반 기능 버튼(강화·뽑기·받기·확인·입장 등 주 실행)
- **파랑**: 특수 기능 버튼. 같은 화면의 초록 버튼과 구분되는 특별 기능(초기화·교환·구매·합성·다시 뽑기 등)
- **회색**: 취소·뒤로·이전/다음 같은 보조 버튼
- 애매한 것은 사용자가 아래 표의 분류대로 확정했다(2026-11-24).

## 길이 규칙
- 버튼 폭이 220 이하면 short(160×80), 350 이하면 medium(280×80), 그보다 크면 long(426×80) 그림을 쓴다.
- 기존 버튼의 RectSize(폭·높이)와 위치는 **바꾸지 않았다**. 그림은 Sliced(경계 L16/B12/R16/T12)로 늘어난다.

## 글자 구조
- 버튼마다 `Labels`(CanvasGroup, BlocksRaycasts 끔) 아래에 `Glow`(하이라이트에서만 켬)와 `Front`를 UIBuilder로 추가했다.
- 글자 값: 폰트 Maple, 흰색. 외곽선 0.045, 그림자 (0,0,0,0.42) / 오프셋 (0.35, −0.55)이며 가이드 값 그대로다. 외곽선 색은 초록 (0.15,0.23,0.06,0.55), 파랑 (0.08,0.18,0.22,0.55), 회색 (0.18,0.18,0.18,0.65)이다.
- 글자 크기는 32×(버튼 높이/80)으로 정하고 16~36으로 제한한다. 폭이 모자라면 더 줄인다. BestFit을 켜서 실행 중에 바뀌는 긴 글자는 MinSize 12까지 줄어든다.
- 원래 글자(버튼 자체 또는 `Label` 같은 자식의 TextGUIRenderer)는 **컴포넌트만 꺼서 남겨 두었다**. 화면 스크립트는 지금처럼 원래 글자에 문자열을 쓰면 되고, `UIButtonLogic`이 매 프레임 Front·Glow로 옮긴다.
- 상태 처리는 `RootDesk/MyDesk/UIButtonLogic.mlua`가 한다. 공용 RUID로 공용 버튼을 알아보고, Transition은 None으로 둔다.
  - 기본: normal 그림, 글자 알파 1
  - 하이라이트: hover 그림, Glow 켬
  - 눌림: pressed 그림, 글자 Y−2
  - 비활성: normal 그림(같은 그림), Labels 알파 0.35, ButtonComponent.Enable=false, 버튼 CanvasGroup.Interactable=false
  - 비활성 여부는 `ButtonComponent.Enable` 하나로 정한다. 화면 스크립트가 Enable을 직접 바꾸거나 `_UIButtonLogic:SetDisabled`를 불러도 같게 반영된다.
  - 공용 버튼에는 화면별 `SetSkin` / `UseTheme` / `StyleButton` / `SetSelected`를 무시하고 공용 그림을 유지한다.
- 새 버튼을 공용 버튼으로 만들 때: 배경 ImageRUID를 공용 normal RUID로 하고 `Labels/Glow/Front` 구조를 붙인 뒤 원래 글자를 끈다. 적용 스크립트 방식을 따른다.

## 대상 아님
탭(파랑 선택 탭), 닫기 X, 카드·칸·목록 행(층 목록, 직업 카드, 각인 노드 등), 아이콘만 있는 버튼(<, >, 오프라인 아이콘), 토글·영구 선택 버튼(은총 프리셋, 성장 1칸/2칸/3칸, 잠재·주문서 모드, 보스 단계 1~5, 스킬 테스트 돌파 1~5·행 버튼), 정보 줄(펫 확률 표시).

## 보류
| 화면 | 버튼 | 이유 |
|---|---|---|
| RelicUI | Panel/Draw1 "1개 사용", Panel/Draw10 "10개 사용", Panel/Enhance "강화" (초록 예정) | **보류(다른 세션 파일)**: `ui/RelicUI.ui`, `RelicUILogic.mlua`는 유물 뽑기 연출 세션 소관 |

## 적용 표 (총 91개: 초록 60 / 파랑 15 / 회색 16)
| 화면 | 버튼 경로 | 글자 | 색 | 그림 | 버튼 크기 | 글자 크기 | 글자 원본 | 근거 |
|---|---|---|---|---|---|---|---|---|
| AccountUI | `Root/Window/Right/Appearance/Footer/Apply` | 적용 | 초록 | medium | 256x48 | 19 | (자체) | 꾸미기 적용 = 일반 실행 |
| CharGachaUI | `Root/Result/BtnOne` | 1회 뽑기 · 다이아 100 | 초록 | long | 400x96 | 36 | (자체) | 1회 뽑기 = 주 실행 |
| CharGachaUI | `Root/Result/BtnTen` | 10회 뽑기 · 다이아 1,000 | 초록 | long | 400x96 | 32 | (자체) | 10회 뽑기 = 주 실행 |
| CharGachaUI | `Root/Banner/BtnExchange` | 카드 교환 | 파랑 | medium | 260x88 | 35 | (자체) | 카드 교환 = 뽑기와 구분되는 특별 기능 |
| CharGachaUI | `Root/Banner/BtnRates` | 확률 보기 | 회색 | medium | 260x88 | 35 | (자체) | 확률 보기 = 보조 정보 **(애매·확정)** |
| CharGachaUI | `Root/Modal/Exchange/Body/BtnOk` | 교환 (포인트 30) | 초록 | long | 360x88 | 35 | (자체) | 교환 확정 = 일반 실행 |
| CharacterUI | `Root/Detail/BtnLv1` | 1레벨업 | 초록 | short | 170x60 | 24 | (자체) | 1레벨업 = 성장 실행 |
| CharacterUI | `Root/Detail/BtnLvMax` | 최대 레벨업 | 초록 | short | 220x60 | 24 | (자체) | 최대 레벨업 = 성장 실행(보수적으로 초록) **(애매·확정)** |
| CharacterUI | `Root/Detail/BtnPromote` | 전직 | 초록 | long | 432x64 | 26 | (자체) | 전직 = 성장 실행 |
| CharacterUI | `Root/Detail/BtnPromoteCommon` | 공용 카드로 전직 | 파랑 | long | 432x64 | 26 | (자체) | 공용 카드로 전직 = 전직과 구분되는 대체 수단 |
| CharacterUI | `Root/Detail/BtnUnlock` | 공용 카드로 해금 (1장) | 초록 | long | 432x64 | 26 | (자체) | 공용 카드로 해금 = 잠긴 캐릭터의 주 실행 |
| ConfirmUI | `Root/Box/Yes` | 중단하기 | 초록 | short | 168x70 | 28 | (자체) | 중단하기 = 확인 실행 |
| ConfirmUI | `Root/Box/No` | 계속하기 | 회색 | short | 168x70 | 28 | (자체) | 계속하기 = 취소(창 닫기) |
| DealUI | `Deal/Col_Angel/Btn` | 축복 받기 | 초록 | long | 512x88 | 35 | (자체) | 축복 받기 = 선택 실행 |
| DealUI | `Deal/Col_Devil/Btn` | 계약 맺기 | 초록 | long | 512x88 | 35 | (자체) | 계약 맺기 = 선택 실행 |
| DealUI | `Deal/Skip` | 지나가기 | 회색 | medium | 256x88 | 35 | (자체) | 지나가기 = 보조 |
| EngraveUI | `Panel/Right/BtnUp` | 레벨업 | 초록 | long | 420x72 | 29 | (자체) | 레벨업 = 성장 실행 |
| EngraveUI | `Panel/Right/BtnResetAll` | 전체 초기화 | 파랑 | medium | 300x64 | 26 | (자체) | 전체 초기화 = 특별 기능 |
| EngraveUI | `Panel/Right/BtnResetTab` | 탭 초기화 | 파랑 | medium | 300x64 | 26 | (자체) | 탭 초기화 = 특별 기능 |
| GachaUI | `Panel/BtnOne` | 1회 뽑기 | 초록 | medium | 300x66 | 26 | (자체) | 1회 뽑기 = 주 실행 |
| GachaUI | `Panel/BtnTen` | 10회 뽑기 | 초록 | medium | 300x66 | 26 | (자체) | 10회 뽑기 = 주 실행 |
| GachaUI | `Panel/BtnSkip` | 전체 공개 | 파랑 | medium | 300x66 | 26 | (자체) | 전체 공개 = 연출 건너뛰기 특별 기능 **(애매·확정)** |
| GachaUI | `Panel/BtnDone` | 확인 | 초록 | medium | 300x66 | 26 | (자체) | 확인 = 일반 실행 |
| GachaUI | `Panel/Select/BtnSelect` | 선택 | 초록 | medium | 300x66 | 26 | (자체) | 선택 = 일반 실행 |
| GachaUI | `Panel/Unlock/BtnOk` | 확인 | 초록 | medium | 260x62 | 25 | (자체) | 확인 = 일반 실행 |
| GraceUI | `Panel/Roll` | 은총 받기 | 초록 | long | 376.875x56.25 | 23 | (자체) | 은총 받기 = 주 실행 |
| GrowthUI | `Root/Slot/Upgrade` | 강화하기 | 초록 | medium | 320x70 | 28 | (자체) | 강화하기 = 성장 실행 |
| GrowthUI | `Root/Slot/Pot/Take` | 새 옵션 적용 | 초록 | medium | 240x70 | 28 | (자체) | 새 옵션 적용 = 실행 |
| GrowthUI | `Root/Slot/Pot/Keep` | 이전 유지 | 회색 | short | 200x70 | 28 | (자체) | 이전 유지 = 취소 성격 보조 |
| GrowthUI | `Root/Slot/Pot/Buy` | 다이아 30 | 파랑 | short | 160x54 | 22 | (자체) | 다이아 구매 = 강화와 구분되는 특별 기능 **(애매·확정)** |
| LobbyUI | `Root/BtnPlay` | 플레이 | 파랑 | long | 448x128 | 36 | Label | 플레이 = 메뉴(초록)와 구분되는 대표 기능 **(애매·확정)** |
| LobbyUI | `Root/BtnGrowth` | 성장 | 초록 | medium | 288x96 | 35 | Label | 성장 메뉴 = 일반 기능 |
| LobbyUI | `Root/BtnGacha` | 캐릭터 뽑기 | 초록 | medium | 288x96 | 33 | Label | 캐릭터 뽑기 메뉴 = 일반 기능 |
| LobbyUI | `Root/BtnAchieve` | 캐릭터 | 초록 | medium | 288x96 | 35 | Label | 캐릭터 메뉴 = 일반 기능 |
| LobbyUI | `Root/BtnSkillTest` | 스킬 테스트 | 회색 | medium | 256x72 | 29 | Label | 스킬 테스트 = 개발용 보조 **(애매·확정)** |
| LobbyUI | `Root/RegionSelect/enter` | 입장 | 초록 | long | 420x112 | 36 | enterText | 지역 입장 = 주 실행 |
| LobbyUI | `Root/RegionSelect/currentRegion` | 현재 도전 지역으로 | 파랑 | long | 580x60 | 24 | currentRegionText | 현재 도전 지역으로 = 탐색 특별 기능 |
| LobbyUI | `Root/RegionSelect/BossView/Challenge` | 도전 | 초록 | long | 420x112 | 36 | (자체) | 보스 도전 = 주 실행 |
| LobbyUI | `Root/StageSelect/Enter` | 입장 · 에너지 3 | 초록 | long | 680x112 | 36 | (자체) | 층 입장 = 주 실행 |
| LobbyUI | `Root/StageSelect/Prev` | 이전 | 회색 | short | 220x104 | 36 | (자체) | 이전 = 보조 이동 |
| LobbyUI | `Root/StageSelect/Next` | 다음 | 회색 | short | 220x104 | 36 | (자체) | 다음 = 보조 이동 |
| MagicBookUI | `Win/Use` | 사용하기 | 초록 | medium | 340x72 | 29 | (자체) | 사용하기 = 실행 |
| MainPanel | `GiveUp` | 중단 | 회색 | short | 208x88 | 35 | (자체) | 중단 = 보조(확인창을 엶) **(애매·확정)** |
| MainPanel | `Panel/Content/Skill/List/Row_DoubleStrike/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Execute/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_IronSkin/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Mastery/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_MesoHunter/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Might/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Precision/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Skill12/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Swift/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Undying/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Vampire/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Skill/List/Row_Vitality/Btn` | 강화 | 초록 | short | 156x65 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Cap/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Cape/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Coat/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Glove/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Shoes/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MainPanel | `Panel/Content/Stat/Row_Weapon/Btn` | 강화 | 초록 | short | 158.4x66 | 26 | (자체) | 강화 = 성장 실행(행별 18개) |
| MysteryUI | `Win/Accept` | 받는다 | 초록 | short | 220x72 | 29 | (자체) | 받는다 = 실행 |
| MysteryUI | `Win/Skip` | 지나친다 | 회색 | short | 220x72 | 29 | (자체) | 지나친다 = 보조 |
| OfflineUI | `Root/Panel/BtnClaim` | 받기 | 초록 | medium | 240x72 | 29 | (자체) | 받기 = 실행 |
| PetUI | `Panel/GachaBg/Draw1` | 1회 뽑기 | 초록 | long | 400x84 | 34 | (자체) | 1회 뽑기 = 주 실행 |
| PetUI | `Panel/GachaBg/Draw10` | 10회 뽑기 | 초록 | long | 400x84 | 34 | (자체) | 10회 뽑기 = 주 실행 |
| PetUI | `Panel/GachaBg/Buy1` | 1장 구매 · 50 | 파랑 | medium | 250x60 | 24 | (자체) | 뽑기권 구매 = 뽑기와 구분되는 특별 기능 |
| PetUI | `Panel/GachaBg/Buy10` | 10장 구매 · 450 | 파랑 | medium | 250x60 | 24 | (자체) | 뽑기권 구매 = 뽑기와 구분되는 특별 기능 |
| PetUI | `Panel/DetailBg/Equip` | 장착 | 초록 | long | 360x80 | 32 | (자체) | 장착 = 실행 |
| PetUI | `Panel/Result/Box/Ok` | 확인 | 초록 | medium | 280x72 | 29 | (자체) | 확인 = 일반 실행 |
| PopupGroup | `PopupBack/PopupPanel/PopupBtnOK` | 확인 | 초록 | medium | 278.4x116 | 36 | (자체) | 확인 = 실행 |
| PopupGroup | `PopupBack/PopupPanel/PopupBtnCancel` | 취소 | 회색 | medium | 278.4x116 | 36 | (자체) | 취소 = 보조 |
| ResultPopup | `Root/Panel/BtnRetry` | 다시 도전 | 초록 | short | 210x87.5 | 35 | (자체) | 다시 도전 = 주 실행 |
| ResultPopup | `Root/Panel/BtnLobby` | 로비로 | 회색 | short | 210x87.5 | 35 | (자체) | 로비로 = 뒤로 |
| ResultPopup | `Root/Result2/retry` | 다시 도전 | 초록 | long | 480x112 | 32 | retryText | 다시 도전 = 주 실행 |
| ResultPopup | `Root/Result2/resultBack` | 지역 선택으로 | 회색 | long | 480x112 | 36 | resultBackText | 지역 선택으로 = 뒤로 |
| SkillTestUI | `Root/Replay` | 다시 재생 | 초록 | short | 192x80 | 32 | (자체) | 다시 재생 = 실행 |
| SkillTestUI | `Root/Monster` | 몬스터 변경 | 파랑 | short | 192x80 | 32 | (자체) | 몬스터 변경 = 특별 기능 **(애매·확정)** |
| SkillTestUI | `Root/Lobby` | 로비로 | 회색 | short | 192x80 | 32 | (자체) | 로비로 = 뒤로 |
| SkillUI | `Panel/BtnUpgrade` | 레벨업 | 초록 | medium | 280x70 | 28 | (자체) | 레벨업 = 성장 실행 |
| SkillUI | `Panel/BtnSynth` | 합성 | 파랑 | medium | 280x70 | 28 | (자체) | 합성 = 레벨업과 구분되는 특별 기능 |
| SkillUI | `Panel/Formation/Btn1` | 1번으로 | 초록 | short | 110x48 | 19 | (자체) | 1번으로 = 배치 실행 |
| SkillUI | `Panel/Formation/Btn2` | 2번으로 | 초록 | short | 110x48 | 19 | (자체) | 2번으로 = 배치 실행 |
| SkillUI | `Panel/Formation/Btn3` | 3번으로 | 초록 | short | 110x48 | 19 | (자체) | 3번으로 = 배치 실행 |
| SymbolUI | `Panel/Prev` | 이전 | 회색 | short | 132x52 | 21 | (자체) | 이전 = 보조 이동 |
| SymbolUI | `Panel/Next` | 다음 | 회색 | short | 132x52 | 21 | (자체) | 다음 = 보조 이동 |
| TarotUI | `Draft/Confirm` | 운명을 받아들인다 | 초록 | short | 211.2x88 | 22 | (자체) | 운명을 받아들인다 = 확정 실행 |
| TarotUI | `Choice/Reroll` | 다시 뽑기 (1) | 파랑 | short | 150x40 | 16 | (자체) | 다시 뽑기 = 특별 기능 |
| WorkshopUI | `Win/Shop/Reroll` | 다시 뽑기 | 파랑 | long | 360x64 | 26 | (자체) | 다시 뽑기 = 구매와 구분되는 특별 기능 |
| WorkshopUI | `Win/Detail/Sell` | 판매 | 파랑 | short | 200x64 | 26 | (자체) | 판매 = 특별 기능 **(애매·확정)** |
| WorkshopUI | `Win/Detail/Leave` | 나가기 | 회색 | short | 200x64 | 26 | (자체) | 나가기 = 뒤로 |
