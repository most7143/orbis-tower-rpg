# 전투 능력치 개편 진행 기록 (S1~S6)

- 작성일: 2026-11-24
- 기준 문서: [Stat-Redesign-Spec.md](Stat-Redesign-Spec.md)
- 상태: S1~S5 구현 완료, S6 일부 구현(아래 "남은 문제" 참고). 커밋 안 함.

---

## 1. 확정 규칙 (메인 담당 결정)

- 공격 타입: **비숍만 마법**, 히어로·나이트로드·보우마스터·바이퍼는 물리. 스킬마다 따로 지정 가능.
- 속성 7종: 무속성(None)·불(Fire)·얼음(Ice)·번개(Lightning)·독(Poison)·성(Holy)·어둠(Dark). 물 없음.
- 배율: 약점 ×1.5, 반감 ×0.5, 무효 ×0.1 (`RunConfig`로 조정).
- 마법은 빗나가지 않음. 물리 빗나감 = 몬스터 회피 − 자기 칸 명중, 0~60%.
- 원본 몬스터 방어·회피·속성 수치는 SSD에 없어 설계값과 `MonsterTable.Trait`(도감 문구) 사용.

## 2. 바꾼 파일

| 파일 | 내용 |
|---|---|
| `RootDesk/MyDesk/Data/ElementTable.csv` + `.userdataset` (새) | 속성 7종 Id·이름·색·아이콘(임시 원형 점) |
| `Data/MonsterTable.csv` | 열 추가 `AtkType, DefP, DefM, Elem, #ElemSource`. `Elem`은 `Trait` 글자에서 변환(166종), 언데드는 설계로 `Holy:W` 추가(35종, `#ElemSource`에 `설계`) |
| `Data/MonsterTypeTable.csv` | 열 추가 `DefP, DefM` (방어형 물리 30, 회피형 마법 10), 행 추가 `마력형`(마법 방어 30, 아직 배정한 몬스터 없음) |
| `Data/MonsterPrefixTable.csv` | 접두사 `마력 결계`(마법 방어 +60) 추가. `단단한`(Def)은 물리 방어만 |
| `Data/CharJobTable.csv` | 열 추가 `AtkType`(P/M), `Element`(직업 기본 속성): 비숍 M·Holy, 나머지 P·None |
| `Data/JobSkillTable.csv` | 열 추가 `DmgType`(전부 빈칸 = 직업 값). `Element`는 기존 한글 값 유지(코드가 한글·Id 둘 다 읽음) |
| `Data/RunConfig.csv` | 키 추가 `ElemWeakMul, ElemResistMul, ElemImmuneMul, MagicDefRatio, MissCap, PlayerEvadeCap, PlayerMissTotalCap, MagicNeverMiss` |
| `Tower/TowerRunManager.mlua` | 표 읽기, `CombatMul`, `HitProfile`, `DefFactor`, 타격 주인(`SetHitSrc`/`ClearHitSrc`/`EnsureHitSrc`), `ActorAcc`, `ActorMissRate`, `DebugCombatMul`, 몬스터 공격 마법 방어·회피 상한 |
| `Tower/HeroCombat.mlua`, `JobCombat.mlua`, `BishopCombat.mlua` | 타격 직전에 타격 주인·스킬 Id 지정 (`SetHitSrc` … `ClearHitSrc`) |
| `Tower/TowerPlayer.mlua` | 기본 공격(`CalcDamage`) 타격 주인 지정, 약점/반감/무효 표시 `ShowElemMark`(클라이언트 빛 효과) |
| `Tooltip/TooltipLogic.mlua` | 스킬 툴팁 종류 줄에 `물리/마법 · 속성(속성색)` 태그 (`CombatTag`) |
| `Character/CharacterUILogic.mlua` | 캐릭터 화면 스킬 툴팁에 공격 타입·속성 전달 |
| `Lobby/RegionSelectLogic.mlua` | 지역 선택 등장 몬스터 이름 뒤에 약점 속성색 점 `●`(최대 2개) |

`PlayerEngrave.mlua`, `PlayerOffline`, `OfflineUI`, `LobbyUI.ui`, `.ui` 파일은 건드리지 않았다.

## 3. 계산 구조

```
FinalDouble(피해)            ← 숫자를 띄우기 직전 단일 관문 (13곳)
  └ CombatMul(대상)          ← 새 배율 하나
       속성 배율 (약점 1.5 × (1+WeakDmg%) / 반감 0.5 / 무효 0.1)
     × 공격 타입 방어 보정 (스킬 타입 ≠ 구성원 기본 타입일 때만)
  └ 각인 '섬멸' (기존)
```

- **방어**: `ActorHitDamage`/`PlayerHitDamage`는 구성원 **기본 공격 타입**의 몬스터 방어(물리 = `DefP`, 마법 = `DefM`)로 깎는다(`DefFactor`). 방어 무시(`DefIgnore`)는 둘 다 적용. 기존처럼 기본 피해에서 한 번만 깎으므로 이중 적용 없음. 스킬 타입이 다르면 `CombatMul`이 `DefFactor(스킬 타입)/DefFactor(기본 타입)` 비율만 곱한다. 지속 피해는 방어 보정 없음(기존에도 방어 없음).
- **몬스터 방어**: 물리 = (몬스터 `DefP` 또는 타입 `DefP` + '단단한') × 등급 배율, 마법 = (몬스터 `DefM` 또는 타입 `DefM` + '마력 결계') × 등급 배율.
- **타격 주인**: 각 직업 전투 컴포넌트가 피해 직전에 `mgr:SetHitSrc(구성원, 스킬Id, 지속여부)`, 끝나면 `mgr:ClearHitSrc(이전값)`. 주인을 모르는 피해(가시 반격·연소 폭발·처형 등)는 무속성·보정 없음(= 1배, 기존과 같음).
- **화상(TickBurn)**: 불 속성 지속 피해(`dmgCtx.elemId = "Fire"`).
- **유물 '속성 피해'(ElemDmg)**: `dmgCtx.elem`(참/거짓) 기존 의미 그대로 유지. 속성 Id는 새 필드 `dmgCtx.elemId`.
- **명중**: `ActorAcc(구성원)` = 자기 편성 칸 망토 명중 + `Bonus("Acc")`. 기존 `Accuracy` 동기화 값(1칸)은 표시용으로 남김.
- **몬스터 공격**: 디버프 빗나감 상한 75%, 회피(`Evade`) 상한 50%, 둘을 합친 빗나감 75% 이하. 몬스터 `AtkType = M`이면 캐릭터 방어력의 50%만 적용.
- **새 효과 키**(값 배치 없음): `WeakDmg`(약점 피해 +%), `ElemPierce`(반감을 보통으로 볼 확률 %), `ShieldDmg`(보스 보호막, 아직 미사용).

## 4. 스킬 개편 담당에게 넘길 데이터 형식

`RootDesk/MyDesk/Data/JobSkillTable.csv` 에 두 열만 채우면 바로 반영된다.

| 열 | 값 | 빈칸이면 |
|---|---|---|
| `Element` | `None/Fire/Ice/Lightning/Poison/Holy/Dark` (권장). 한글 `무/불/얼음/번개(전기)/독/성(신성)/어둠`도 읽음. `출혈`은 무속성 취급 | 직업 기본 속성(`CharJobTable.Element`: 비숍 Holy, 나머지 None) |
| `DmgType` | `P`(물리) / `M`(마법). `물리`/`마법`도 읽음 | 직업 기본 공격 타입(`CharJobTable.AtkType`) |

- **주의**: 지금 `H06 인사이징 = 불`, `H14 소드 일루전 = 출혈(=무속성)`, 팔라딘 일부 `불/얼음/신성`이 남아 있다. 개편 문서는 인사이징을 `무`로 제안했으니, 스킬 담당이 `Element`를 `None`(또는 `무`)으로 바꾸면 된다. 빈칸으로 두면 직업 기본값이 된다(히어로 = 무속성).
- 무속성 스킬을 비숍에 두려면 빈칸이 아니라 `None`을 써야 한다(빈칸 = 비숍 기본 Holy).
- 타격 코드 쪽은 이미 스킬 Id(`r.id`)를 넘기고 있다. 새 스킬을 추가할 때도 기존 `ApplyHit`/`DealTo`/`ApplyProcHit` 경로를 쓰면 자동 적용. 새 경로로 `mgr:FinalDouble`을 직접 부르면 그 앞뒤에 `local prev = mgr:SetHitSrc(self.Entity, 스킬Id, 지속여부)` / `mgr:ClearHitSrc(prev)` 를 넣는다.
- 몬스터 속성 형식(`MonsterTable.Elem`): `Fire:W|Ice:R|Poison:N` (W 약점, R 반감, N 무효). 몬스터 `AtkType`(P/M), `DefP`·`DefM`(숫자, 빈칸 = 타입 기본).

## 5. 테스트 담당 확인 목록 (짧게, 밸런스 측정 금지)

- [ ] 빌드 오류 0 (Maker Refresh → 빌드 로그)
- [ ] 판 시작 로그 `[VRF] combat jobs Bishop=M/Holy Hero=P/None`
- [ ] 몬스터 등장 로그에 `def= defM= elem=` 표시 (예: 달팽이 `def=30 ... elem=Poison:R`, 슬라임 `defM=10 elem=Lightning:W`)
- [ ] 서버 스크립트로 배율 확인: `mgr:DebugCombatMul("Fire:W", "Hero", "H06")` → 1.50, `("Holy:R", "Bishop", "")` → 0.50, `("Holy:N", "Bishop", "")` → 0.10, `("Poison:N", "Hero", "")` → 1.00. 로그 `[VRF] debug combatmul ...`
- [ ] 비숍 편성 후 언데드·성 약점 몬스터(예: 좀비 계열) 전투에서 `[VRF] elem hit elem=Holy res=W mul=1.50` 로그 + 몬스터 위 속성색 빛
- [ ] 성 반감 몬스터에서 회색 작은 빛, `res=R mul=0.50`
- [ ] 회피형 몬스터(슬라임 회피 25)에서 물리 구성원 `[VRF] miss actor=... rate=19.0 acc=6.0` 와 MISS 그림, **비숍은 빗나감 로그가 없어야** 함 (`mgr:ActorMissRate(비숍)` = 0)
- [ ] 캐릭터 화면 스킬 툴팁 종류 줄: 비숍 `마법 · 성`(노란 글자), 히어로 `물리`
- [ ] 지역 선택 등장 몬스터 이름 뒤 약점 점(예: 스텀프 주황 ●). 이름이 길면 잘리는지
- [ ] 기존 기능 회귀: 유물 '속성 피해', 각인 '섬멸', 화상 숫자, 광역 스킬 줄 선 몬스터 피해가 그대로 뜨는지

### 구현 담당이 직접 확인한 것 (2026-11-24, 디버그 판 1회)

- Refresh 후 빌드 로그 오류(LEA) 0, 실행 중 `LEA-` 로그 0.
- `combat jobs Bishop=M/Holy Hero=P/None`, 몬스터 로그 `달팽이 def=30 elem=Poison:R`, `슬라임 defM=10 elem=Lightning:W`.
- `DebugCombatMul` 5건 기대값과 일치: 1.50 / 0.50 / 0.10 / 1.00 / 1.50.
- 실제 전투 빗나감: `miss actor=Companion2 job=Bowmaster rate=4.0 acc=6.0` (구성원별 명중 적용).
- 못 본 것: 실제 전투 중 속성 약점 타격(파티에 비숍 없음, 1층 몬스터와 직업 속성이 맞지 않음), 약점 빛 효과·툴팁·지역 선택 점의 화면 모습(스크린샷 안 찍음).

### 테스트 후 수정 (2026-11-24)

- 지역 선택 약점 점을 이름 **앞**으로 옮김 (`RegionSelectLogic.WeakDots` .. 이름). 긴 이름은 뒤쪽만 잘리고 점은 항상 보인다.
- 속성 색 변경 (성·번개가 비슷했음): 번개 `#22C3A6`(청록), 성 `#F2C230`(금색), 어둠 `#4A3F66`(짙은 남보라). 불·얼음·독은 그대로. `ElementTable.csv`와 `RegionSelectLogic`에 반영.
- **아직 옛 색이 남은 곳** (스킬 담당이 고치는 중이라 손대지 않음): `Tower/TowerPlayer.mlua` `ShowElemMark`(번개 노랑·성 연금색·어둠), `Tooltip/TooltipLogic.mlua` `CombatTag`(번개 `#F2C230`, 성 `#F5E6A8`, 어둠 `#9C8AD6`). 번개는 `#22C3A6`, 성은 `#F2C230`으로 맞춰야 한다(검은 툴팁 바탕에서는 어둠을 밝은 남보라로 둬도 됨).

## 6. 남은 문제 / 못 한 것

- **지역 선택 약점 점**: 원래 계획은 몬스터 그림 오른쪽 아래 원형 점 스프라이트였지만, 그 화면이 `ui/LobbyUI.ui` 안에 있고 다른 담당이 고치는 중이라 `.ui`를 건드리지 않았다. 대신 이름 글자 뒤 리치 텍스트 `●`로 임시 표시. 나중에 `Mon0~17` 아래 `Weak0/Weak1` 스프라이트(RUID `64a8800e417d4321853484516bb3c9b2`, 속성색 틴트)를 추가하고 `WeakDots` 대신 쓰면 된다. 몬스터 칸 툴팁(이름/약점/반감/무효)도 같은 이유로 미구현.
- **캐릭터 화면 공격 타입 아이콘(검/별 지팡이)**: 미구현. 지금은 스킬 툴팁 글자로만 표시. 원본 데이터에서 알맞은 아이콘을 찾지 못함(`_Resources/UI`에 속성·공격 타입 아이콘 세트 없음). `CharacterUI.ui` 수정 필요.
- **속성 아이콘**: 원본에서 찾지 못해 흰 원형 점(32×32, 회색 테두리)을 직접 만들어 올림 — RUID `64a8800e417d4321853484516bb3c9b2`, `ElementTable.IconRUID` 전부 이 값(색은 `Color` 열로 틴트). 원본 리소스를 고른 것이 아니라 비교 미리보기는 만들지 않음.
- **피해 숫자 색**: 데미지 스킨은 숫자 색을 바꿀 수 없어 숫자 대신 몬스터 위 빛(파티클 `CircleBurst`)으로 표시. 엔진 자동 숫자(기본 공격)도 같은 빛만 나온다. "약점/무효" 글자는 넣지 않음(글자 최소화).
- **빗나감 표시**: 기존 데미지 스킨 MISS 그림(`DamageSkinTextType.Miss`) 그대로 사용.
- **몬스터 마법 공격**: 데이터상 `Ranged=1` 몬스터가 없어 `AtkType=M` 몬스터가 아직 없다. 코드만 있음.
- **마력형 타입**: 행만 추가, 배정 몬스터 없음 (타입을 바꾸면 체력·공격 배율도 바뀌어 기존 밸런스가 흔들림).
- **보스 그로기 배율·보호막(ShieldDmg)**: 보스 문서 단계에서 `CombatMul`에 곱을 하나 더 넣으면 된다.
- **대기열 몬스터 방어**: 광역 피해의 기본값은 맨 앞 몬스터 방어 기준(기존과 같음). 속성은 각 대기열 몬스터 정보로 판정.
