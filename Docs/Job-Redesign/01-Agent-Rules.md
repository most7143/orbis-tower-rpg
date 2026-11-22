# 개편 작업 에이전트 공통 규칙

모든 에이전트(자료 조사·구현·테스트)는 작업 전에 이 문서와 `00-Master-Spec.md`를 읽는다.

## 기본
- 작업 폴더: `/Users/seogeon-u/orbis-tower-rpg` (MSW 프로젝트). 여기서만 작업한다.
- 문서·UI 문구·코드 주석은 **한국어**. 코드 식별자는 영어.
- 프로젝트 지침 `CLAUDE.md` / `AGENTS.md`를 따른다. MSW 작업 전 `Skill` 도구로 `msw-general`, `msw-ui-system`을 불러오고, 필요한 참조 문서를 읽는다(경로로 읽지 말고 Skill 도구 사용).
- **커밋·푸시 금지.** 사용자가 요청할 때만 메인 세션이 한다.
- **보안**: `.mcp.json`, `.codex/` 에는 MCP 토큰이 있다. 절대 읽어서 출력하거나 다른 파일로 복사하지 않는다.

## 파일 규칙
- `.ui` / `.model` / `.map`은 **빌더로만** 수정한다 (`.claude/skills/msw-ui-system/scripts/msw_ui_builder.cjs` 의 UIBuilder, msw-general의 ModelBuilder/MapBuilder). 셸 명령에 `.ui` 경로 문자열을 직접 쓰면 가드에 막힌다 → node 스크립트에서 `'ui/'+'Name'+'.'+'ui'`처럼 조립.
- 새 스크립트는 `RootDesk/MyDesk/<기능폴더>/` 아래에. 이번 개편 코드는 가능하면 새 폴더 `RootDesk/MyDesk/Character/` 에 둔다.
- 데이터 표: `RootDesk/MyDesk/Data/<이름>.csv` (UTF-8 BOM) + 같은 이름 `.userdataset` (기존 `RelicTable.userdataset` 형식을 복사해 새 uuid로). `_DataService:GetTable("이름")`으로 읽는다.
- `.codeblock` 파일은 직접 만들거나 고치지 않는다 (메이커 새로고침이 만든다).
- UI는 `Docs/UI-Design-System.md` 를 따른다: 검정 외곽 + 밝은 본문, 파랑=선택, 초록=실행, 버튼 상태 이미지(SpriteSwap, `output/ui/growth_neutral/manifest.json`), 버튼 글자 중앙 정렬, 닫기는 공통 64×64 X.
  - 빌더로 만든 버튼은 Transition이 ColorTint라 상태 이미지가 안 들어간다. 만든 뒤 `ButtonComponent`를 `Transition: 2` + `ImageRUIDs`(hover/pressed/selected/disabled)로 패치한다 (예: 메인 세션 스크래치패드의 btn_states.cjs 방식).
  - 기존 화면 예시: 유물 화면 `ui/RelicUI` + `RootDesk/MyDesk/Lobby/RelicUILogic.mlua`, 성장창 `ui/GrowthUI` + `Lobby/GrowthUILogic.mlua`, 정비소 `ui/WorkshopUI`.

## 다른 세션과 충돌 피하기
- 지금 다른 세션이 **유물 뽑기 연출** 작업으로 `RootDesk/MyDesk/Engrave/PlayerEngrave.mlua`, `RootDesk/MyDesk/Lobby/RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md` 를 고치고 있다(커밋 전).
- 이 파일들은 **되돌리거나 덮어쓰지 않는다.** 꼭 필요하면 아주 작은 추가만 하고, 수정 전 최신 내용을 다시 읽는다.
- 캐릭터·장비 계정 데이터는 `PlayerEngrave`에 넣지 말고 **새 컴포넌트**(예: `Character/PlayerRoster.mlua`, 플레이어 엔티티에 부착)로 만든다. DataStorage 키도 새로 (`OrbisRosterV1` 등).

## 메이커 사용 요령 (Maker MCP)
- 도구가 지연 로드되어 있으면 `ToolSearch`로 `select:mcp__msw-maker-mcp__maker_refresh_workspace,...` 를 먼저 부른다.
- 순서: `maker_run_command stop` (두 번 불러도 됨) → `maker_refresh_workspace` (편집 모드에서만) → `maker_build_logs`로 오류 확인 → `maker_clear_logs` → `maker_run_command play`.
- 실행 중 확인: `maker_execute_script` (`context`: `server_main` 또는 `client`), 결과는 `log("[VRF] ...")` 후 `maker_console_logs`로 검색. 로그는 조금 늦게 뜬다 → 다시 조회.
- 로비 맵 이름 `Lobby`, 탑 맵 `OrbisTower`. 탑 이동: `_TeleportService:TeleportToMapPosition(플레이어, Vector3(0,0,0), "OrbisTower")` (server_main).
- 플레이 직후 첫 클라이언트 스크립트는 UI가 준비 전이라 안 먹을 수 있다 → 한 번 더 실행.
- 화면 확인: `maker_run_command screenshot` → 출력 경로의 PNG를 Read.
- 마우스 클릭 좌표는 창 크기에 따라 어긋난다 → 버튼 테스트는 클라이언트에서 `엔티티:SendEvent(ButtonClickEvent())`로.
- 오류 확인: `maker_console_logs` 검색어 `LEA-`.
- 같은 문맥(server_main 등)에서 앞 스크립트가 아직 도는 중에 `maker_execute_script`를 또 보내면 앞 스크립트가 중단된다. 기다림이 필요한 확인은 한 스크립트 안에서 `wait()`로 처리한다.
- `stop` 직후 바로 `play`하면 "Already playing"이 날 수 있다. 편집 모드로 돌아온 것을 확인하고 `play`.
- 플레이어 엔티티 찾기: `for _, e in ipairs(_UserService.UserEntities.Values) do p = e break end` (server_main).
- 메이커가 꺼져 있으면(도구 없음) 정적 검사만 하고 그 사실을 보고한다.

## mLua 주의
- `@ExecSpace("Server")` 메서드 안에서만 `senderUserId` 사용 가능.
- `JSONEncode`는 빈 테이블에서 실패 → 문자열 형식("id:lv|...")으로 저장하는 편이 안전.
- `string.gsub` 반환값 개수 문제(LEA-1103) → `string.find` + `sub`로 직접 치환.
- `member`는 예약어, 컴포넌트에 없는 필드 대입은 조용히 실패.
- 메서드 설명 주석은 메서드 본문 첫 줄에.
- 수정 후 LSP 진단이 자동으로 돈다 (error 0 유지).

## 보고
- 각 에이전트는 마지막에 한국어로: 한 일, 바꾼 파일, 확인한 것, 확인 못 한 것, 다음 에이전트에게 넘길 질문.

## 테스트 범위 (사용자 지시 2026-11-22)
- 테스트 에이전트는 **기능이 동작하는지만 짧게** 확인한다. 항목당 한두 번, 판은 필요할 때만 1회 이내.
- **밸런스 실측(층 진행·R값·궁극기 주기 등 여러 판 측정)은 하지 않는다.** 밸런스는 사용자가 직접 플레이해 보고 알려 준다.
- 저장 복원 규칙은 그대로 지킨다.

## 화면 오른쪽 위 금지 영역 (사용자 지시 2026-11-22)
- 화면 오른쪽 위에는 MSW 공용 시스템 버튼 2개가 있다 (UI 기준 x 765~940, y 430~510, 1920×1080 가운데 원점).
- 이 영역(여유 포함 대략 x≥740, y≥410)에는 **누를 수 있는 UI(버튼·닫기 X 등)를 두지 않는다.** 닫기 X는 그 왼쪽(예: (708, 504))이나 패널 안 오른쪽 위에 둔다.

## 메이커 공유 (2026-11-22)
- 사용자도 메이커로 직접 플레이한다. **내가 시작하지 않은 플레이는 절대 `stop` 하지 않는다.** 기다려도 끝나지 않으면 메이커 확인을 건너뛰고 정적 검사만 한 뒤 보고한다.

## 창형 화면 배치 기준 (사용자 지시 2026-11-22)
- 창형 화면(지역 선택·캐릭터·캐릭터 뽑기·성장창·정비소 등)은 화면 전체를 꽉 채우지 않는다: 패널 위쪽 끝 y≤400, 양쪽 끝 약 10px(x -950~950).
- 닫기 X는 패널 안 오른쪽 위(안쪽 여백 16). 위·아래 패널의 왼쪽·오른쪽 끝을 일직선으로 맞춘다.
