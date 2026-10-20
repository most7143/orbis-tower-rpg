# 오르비스탑 RPG — As-built log

> 월드 구현의 현재 상태 기록. 마일스톤이 바뀌어도 유지된다. ⚠️확인 = 조사 추정, 사용자 확인 필요.

## Current state (by system)
| System | Built | Where (key files) | Notes / gotchas |
|---|---|---|---|
| 맵 | `.map` 3개, 모두 MapleTile(0) | `map/Lobby` · `map/OrbisTower` · `map/SkillTest` | 월드 시작 = Lobby. 템플릿 맵은 삭제됨. SkillTest는 테스트 전용(출시 전 로비 버튼 정리 필요 → M2 Phase 1) |
| 판 진행 (층·스테이지·패배·보상·레벨) | 맵 `@Component` | `Tower/TowerRunManager` (OrbisTower 맵) | 층당 10스테이지, 10번째 = 접두사 강화 몬스터. 테스트 플래그 `TestFirstPickTopGrade`·`TestAlwaysTopGrade`가 **true로 남아 있음** (매 선택 미스틱 확정) |
| 플레이어 전투 | `@Component` | `Tower/TowerPlayer` | 서버가 데미지 결정 → 클라 공격 동작 0.3초 뒤 RequestHit로 적용(HitSyncDelay) |
| 몬스터 | `.model` 1개 + 테이블 | `Models/Monsters/TowerStarPixie` · `Tower/TowerMonster` · `Data/MonsterTable`(315종)·`MonsterTypeTable`·`MonsterPrefixTable`·`FloorTable` | 외형·능력치·피격/사망 사운드 모두 테이블 기준 |
| 장비 강화 (판 안) | 테이블 + 탭 UI | `Data/EquipTable`·`EquipSlotTable` · `Tower/MainPanelLogic` · `ui/MainPanel` | 6부위×30티어, 0~10강, 판 종료 시 초기화 |
| 패시브 스킬 + 스킬석 (판 안) | 테이블 + 탭 UI | `Data/SkillTable` · `MainPanelLogic` | M1 계획의 '레벨 해금 스킬 3종'을 대체한 구현 |
| 특성 발현·전직·직업 스킬 (판 안) | `@Logic` + 테이블 | `Tarot/TarotLogic` · `Data/TraitTable`·`TraitGradeTable`·`JobSkillTable` · `ui/TarotUI` | 등급 확률 50/30/14/5/1. 3차 전직 후 카드 선택 중단 |
| 보스 거래 (판 안) | `@Logic` + 테이블 | `Tower/DealUILogic` · `Data/DealTable` · `ui/DealUI` | 천사/악마/상인 3택, 이번 판만 적용 |
| 각인 (아웃게임) | 서버 컴포넌트 + UI | `Engrave/PlayerEngrave`·`EngraveUILogic` · `Data/EngraveTable` · `ui/EngraveUI` | 포인트 = 처음 클리어한 층 수(막히면 0). 저장 키 `OrbisEngrave` |
| 심볼 (아웃게임) | 테이블 + UI | `Symbol/SymbolUILogic` · `Data/SymbolTierTable`·`SymbolPermTable` · `ui/SymbolUI` | 보스 처치 시 50/10/1% 판정. 저장 키 `OrbisSymbol` (PlayerEngrave) |
| 은총 (아웃게임) | 테이블 + UI | `Grace/GraceUILogic` · `Data/GraceOptionTable`·`GraceGradeTable`·`GraceStepTable` · `ui/GraceUI` | 재화 = 여신의 깃털. 저장 키 `OrbisGrace` |
| 최고 기록 저장 | DataStorage | 키 `OrbisTower` | 갱신 시에만 저장 |
| 로비 | `@Logic` + UI | `Lobby/LobbyManager`·`LobbyUILogic` · `ui/LobbyUI` | 업적 버튼은 '준비 중' 안내만 (→ 로드맵 Backlog '업적') |
| 전투 HUD·결과·중단 | `@Logic` + UI | `Tower/BattleHUDLogic`·`ResultPopupLogic` · `ui/BattleHUD`·`ResultPopup`·`ConfirmUI` | 가로 1920×1080 기준, 비16:9 화면 재배치 로직 있음 |
| UI 테마 | `.ui` 12개 리스타일 | `ui/*.ui` | casualrpg 팩의 회색 계열 부품 + 금색 강조. `SkillTestUI`·타로 카드 내부·은총 줄 배경은 런타임 색이라 미변경 |
| 효과음 | 일부 | `TowerMonster` (몬스터 피격·사망) | 레벨업·버튼 효과음 없음 |
| 템플릿 잔여 스크립트 | 사용 여부 ⚠️확인 | `MonsterManager`·`MonsterDeadEvent`·`PlayerAttack`·`PlayerHit`·`UIPopup`·`UIToast` | 템플릿 기본 제공분. UIPopup/UIToast는 게임 코드에서 호출 없음 |
| 저장소 | Git + GitHub | `github.com/most7143/orbis-tower-rpg` (공개) | `.mcp.json`(인증 토큰)·`.mswai/`·`.claude/`는 .gitignore로 제외 |

## Standing issues & handoff rules   (update in place — never re-append)
| Issue / rule | Workaround / rule | Count | First → last seen |
|---|---|---|---|
| 테스트 플래그가 켜진 채 남음 | `TowerRunManager`의 `TestFirstPickTopGrade`·`TestAlwaysTopGrade`를 출시·밸런스 테스트 전 false로 | 1 | 10-14 → 10-20 |
| UITransform의 UIScale만으로는 화면에 반영 안 됨 | Scale에도 같은 값 지정 | 1 | 10-17 → 10-17 |
| 스크롤 목록 안 줄의 WorldPosition에 스크롤 이동량 미반영 | 정규화 스크롤 위치 × 스크롤 범위를 직접 더함 | 1 | 10-17 → 10-17 |
| ScreenToUIPosition이 비16:9에서 실제 UI 배율과 다름 | `BattleHUDLogic.ToUI` 보정 사용 | 1 | 10-17 → 10-17 |
| 회색 아트 위 색 틴트는 곱해져 탁해짐 | 강조는 라벨 색(금색)으로, 버튼 틴트는 흰색/회색만 | 1 | 10-20 → 10-20 |

## Log
### 2026-10-20 Seed — 기존 작업 조사로 기록 시작
M1 GDD·코드·데이터·맵 구조를 조사해 위 현재 상태를 작성. M1 체크리스트에 없던 시스템(특성·거래·각인·심볼·은총)은 구현돼 있으나 검증 기록이 없어 M2 Phase 1 검증 항목으로 등록.

### 2026-10-20 Milestone M1 complete — summary
- 출시물: 자동전투 → 층 상승 → 메소·경험치 → 장비 강화(6부위) → 패배 시 1층 초기화, 로비·결과 팝업·최고 기록 저장, 몬스터 315종 테이블화.
- 범위 축소: '스킬 강화'를 M1 컨셉에서 제외하고, 미검증 항목은 M2 Phase 1로 이관 (보관된 M1 GDD §9 2026-10-20).
- M1 진행 중 계획 밖으로 추가된 시스템: 특성 발현·전직·직업 스킬, 보스 거래, 각인, 심볼, 은총, 중단하기, 가로 화면 전환, 회색 UI 테마.
- 상세 기록: `Archive/OrbisTower-M1-GDD.md`.
