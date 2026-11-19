# 계정 정보 UI 리소스

스테이지 선택 화면과 같은 보라 패널·금색 테두리·파랑 선택 행을 사용합니다.

- 현재 연결: manifest.json v2
- 스크린샷: PREVIEW_stage_theme.png (엔진 실행 화면)
- 버튼: button_normal/hover/pressed/disabled/selected/selected_highlight/selected_pressed.png
- 배경: panel_profile_menu.png, panel_details.png
- 프로필 외곽 테두리: solid_white.png로 사각형 구성, 색상은 런타임 적용
- Legacy/: 현재 사용하지 않는 초기 크림색 배경·버튼·프레임
- 문자열은 이미지에 포함하지 않고 인게임 텍스트로 출력합니다.

현재 구성(v3): 계정 정보 탭 제거, 레벨 아래 EXP, 3×3 외형 격자와 상단 미리보기/하단 조건. 최신 화면 PREVIEW_appearance_grid.png, 칭호 PREVIEW_title_grid.png. NameTagComponent 칭호 모델 accounttitletag 사용.
