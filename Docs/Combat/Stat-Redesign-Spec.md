# 전투 능력치 개편 명세 — 물리/마법 · 속성 · 명중/회피

- 작성일: 2026-11-24
- 상태: **설계 문서 (코드 미변경)**. 사용자 확인 후 구현한다.
- 함께 볼 문서: [../Boss/Boss-Stage-Spec.md](../Boss/Boss-Stage-Spec.md), [../Boss/Field-Boss-Map.md](../Boss/Field-Boss-Map.md), `Docs/Skills/Skill-Overhaul-Spec.md`(다른 담당 작성 중 — 스킬별 물리/마법·속성 태그), [../Job-Redesign/00-Master-Spec.md](../Job-Redesign/00-Master-Spec.md), [../UI-Design-System.md](../UI-Design-System.md)

---

## 0. 한 줄 요약

> 직업마다 **물리/마법** 공격 타입이 있고, 스킬마다 **속성(7종)** 이 있다. 몬스터는 **물리 방어·마법 방어**가 따로 있고, 속성마다 **약점(×1.5)·반감(×0.5)·무효(×0.1)** 이 있다. **마법은 빗나가지 않고**, 물리는 몬스터 회피와 명중으로 빗나갈 수 있다. 화면에는 숫자 대신 **속성 아이콘·색**만 보이고, 설명은 툴팁으로.

---

## 1. 레퍼런스 조사 (모바일 게임)

웹 검색 + 작성자 지식. 수치는 커뮤니티 자료라 버전마다 다를 수 있다(출처 맨 아래).

| 게임 | 시스템 | 핵심 규칙 | 우리 게임에 차용할 단순한 요소 | 채택 |
|---|---|---|---|---|
| 붕괴: 스타레일 | 약점 격파 · 강인도 | 적 체력 위에 흰 강인도 게이지. **약점 속성 공격만** 강인도를 깎고, 0이 되면 격파 → 행동 지연·받는 피해 증가·속성별 디버프 | 보스 **보호막 게이지**: 약점 속성일 때 빨리 깎이고(×1.0), 다른 속성은 느리게(×0.25). 깨지면 **그로기 5초, 받는 피해 +50%** | ✅ 보스전 핵심 |
| 승리의 여신: 니케 | 코어 · 부위 파괴 · 저지 원 | 보스 부위를 부수면 이득, 일부 저지 원은 **속성 우위 유닛만** 깰 수 있고 못 깨면 전멸기 | **패턴 예고 + 저지**: 예고 중 일정 피해(또는 약점 속성 피해)를 넣으면 패턴 취소. 자동 전투에 맞게 "편성으로 해결" | ✅ 보스 1종 |
| 블루 아카이브 | 공격 타입 ↔ 장갑 타입 | 폭발→경장갑, 관통→중장갑, 신비→특수장갑 약 200%, 나머지 감소 | **물리/마법 ↔ 물리 방어/마법 방어** 두 축만. 보스마다 한쪽 방어가 높아 "이번엔 마법 파티" 같은 편성 고민 | ✅ 단순화(3×3 → 2×2) |
| 서머너즈 워 | 속성 상성 순환 | 불>바람>물>불 순환, 불리하면 피해 감소·빗나감(글랜싱) 증가 | 순환 상성은 **채택 안 함**(외울 게 많음). 대신 "불리하면 빗나감" 대신 반감으로 한 번에 표현 | ❌ 참고만 |
| 메이플스토리(PC·클래식) | 몬스터 속성 약점·반감·무효, 명중/회피 | 도감 문구 "○속성 공격에 약함/반감/통하지 않음", 명중 부족·레벨 차이로 MISS. 클래식 기준 **마법은 빗나가지 않음**(NiaMeowDB 검증, 출처마다 다름) | 속성 이름·배율 체계를 그대로. **마법 = 필중, 물리 = 명중 필요** 규칙으로 직업 개성 | ✅ 기준 체계 |
| 메이플스토리M / 메이플 키우기 | 보스 도전 콘텐츠 | 일일 입장 횟수, 단계별 난이도, 첫 클리어 보상 | 보스 탭: **일일 입장 3회**, 단계(1~5) 해금형, 첫 처치 다이아 | ✅ 보스전 운영 |
| 에픽세븐 | 속성 + 보스 기믹 | 속성 우위 시 명중·피해 보정, 보스마다 특정 행동에 반응(디버프 수 등) | 보스 툴팁 한 줄 "○○하면 강해짐/약해짐" — 공략을 글 대신 아이콘 1개 + 툴팁 | ✅ 툴팁 형식 |
| 원신 | 원소 반응 | 두 원소 조합 반응 | 조합 반응은 **채택 안 함**(자동 전투에 과함) | ❌ |
| 쿠키런: 킹덤 | 보스 성격 · 진형 | 보스마다 유리한 진형(앞/뒤), 일일 보스 | 우리 칸 전투(1~3칸)와 연결: 보스 패턴 대상이 "맨 뒤 칸" 같은 식 | ✅ 보스 1종 |

**결론:** 붕괴 스타레일의 *보호막(강인도)* + 블루 아카이브의 *2축 공격 타입* + 메이플 원본의 *속성 약점/반감/무효·마법 필중* + 니케의 *예고 패턴 저지* + 메이플M류의 *일일 보스 운영*을 조합한다. 순환 상성·원소 반응은 넣지 않는다.

---

## 2. 현재 코드 (변경 지점 조사 결과)

| 항목 | 지금 | 위치 |
|---|---|---|
| 캐릭터 기본 능력치 | 공격력·체력·방어력. `(기본 + 레벨당×(Lv-1)) × (1+차수 보너스%)` | `Character/PlayerRoster.mlua` `CharStat`, `StatPct` / `Data/CharJobTable.csv`(BaseAtk·BaseHp·BaseDef·…PerLv), `CharTierTable.csv`(StatPct) |
| 판 시작 능력치 | 구성원별 `actorBase` = 캐릭터 값 × 계정 % + 편성 칸 장비(무기→공격, 투구→체력, 갑옷→방어) | `Tower/TowerRunManager.mlua` 약 3380~3425행 (`RecalcStats` 부근) |
| 공격력 | `ActorAtk` = 기본 × (1 + AtkPct + 무기 강화 + 여신)% | `TowerRunManager.ActorAtk` |
| 주는 피해 | 공격력(숙련도 최소~최대) × `1 - 몬스터방어/(몬스터방어+DefConstant 100)` × 여러 % | `ActorHitDamage`, `PlayerHitDamage`(2574행) |
| 스킬 피해 맥락 | `_T.dmgCtx = { aoe, single, elem, dot }` (스킬 표 `Element` 열이 있으면 elem=true → 유물 '속성 피해' 적용) | `DmgCtxOf`, `RelicDamageBonus` |
| 최종 피해 통로 | 모든 타격이 숫자를 띄우기 전에 `FinalDouble(damage)` 를 지난다 (13곳) | TowerRunManager 6 · HeroCombat 4 · JobCombat · BishopCombat · TowerPlayer 각 1 |
| 빗나감(우리 공격) | `missRate = max(0, 몬스터 회피 - 명중)` %. 몬스터 회피 = 타입(`MonsterTypeTable.EvadeRate` 5/25/5/10) + 접두사 '날렵한' +20 | `ActorAttack` |
| 명중 | 1칸 망토 `Accuracy` + `Bonus("Acc")` (파티 공용) | 3421행 |
| 빗나감(몬스터 공격) | 히어로 디버프 암흑, 각인·펫 `Evade`%, '피격 무효' | `DoMonsterAttack` |
| 받는 피해 | 몬스터 공격 × `1 - 방어/(방어+100)` × 감소 % | `DoMonsterAttack` |
| 몬스터 방어 | 평소 0. 정예 접두사 '단단한' 60 × 등급 배율(정예 2, 보스 3) | 3820~3866행 |
| 몬스터 데이터 | `MonsterTable.csv`(684종): `Type`(공격·회피·방어·밸런스형), `Trait`(**글자로만** "불 약점, 얼음 반감" 등 — 코드에서 안 씀) | `Data/MonsterTable.csv` |
| 스킬 속성 | `JobSkillTable.Element` — 대부분 빈칸, 히어로·팔라딘 일부 "불/얼음/신성/출혈" | `Data/JobSkillTable.csv` |
| 장비 잠재 | 무기·장갑·망토·투구·갑옷 옵션 (AtkPct·SkillDmg·EliteDmg·Crit·…·UltDmg) | `Data/GearPotTable.csv` |
| 펫 효과 | HpPct·DefPct·Evade·Crit·BossDmg·EliteDmg·… | `Data/PetTable.csv` |
| 각인 | AtkPct·Crit·Evade·DefIgnore 없음(돌파·유물 쪽)… | `Data/TalentTable.csv` |
| 유물 | `ElemDmg`(눈의 여왕: 속성 피해 +%) | `Data/RelicTable.csv` RL16 |

**좋은 점:** `FinalDouble` 이 사실상 단일 관문이고 `_T.dmgCtx` 가 이미 있다 → 속성·공격 타입 배율은 여기 한 곳에 넣으면 기본 공격·스킬·지속 피해 모두 적용된다.

---

## 3. 공격 타입 (물리 / 마법)

### 3.1 캐릭터

| 직업 | 공격 타입 | 근거 (원본 메이플) |
|---|---|---|
| 히어로 | 물리 | 전사 |
| 비숍 | **마법** | 마법사 (원본 마력 기반) |
| 나이트로드 | 물리 | 도적 (표창) |
| 보우마스터 | 물리 | 궁수 |
| 바이퍼 | 물리 | 해적 (너클) |

- 기본값은 직업(`CharJobTable.AtkType`). **스킬은 따로 지정 가능**(`JobSkillTable.DmgType`, 빈칸이면 직업 값). 예: 비숍의 모든 스킬 = 마법, 히어로 '소드 오브 버닝 소울'처럼 원본이 특수한 경우만 스킬 표에서 바꾼다. 스킬별 값은 `Skill-Overhaul-Spec.md` 가 정한다.
- 지금 마법 직업이 비숍 하나뿐이라 **마법 방어가 높은 보스 = 비숍 필수**가 되기 쉽다. 그래서 보스의 물리/마법 방어 차이는 **최대 2배**까지만 둔다(§3.3). 이후 마법 직업(예: 불독·썬콜)이 늘면 다시 조정.

### 3.2 몬스터 방어

- 몬스터에 **물리 방어 `DefP`**, **마법 방어 `DefM`** 두 값. 피해 감소 공식은 지금 그대로: `감소율 = 방어 / (방어 + 100)`.
- 일반 몬스터: 둘 다 지금처럼 0 (접두사 '단단한'은 **물리 방어만** +60, 새 접두사 '마력 결계' = 마법 방어 +60 추가).
- 몬스터 `Type` 에 따라 기본값을 준다 (`MonsterTypeTable` 열 추가):

| Type | DefP | DefM | 회피 | 비고 |
|---|---|---|---|---|
| 공격형 | 0 | 0 | 5 | |
| 회피형 | 0 | 10 | 25 | 물리는 빗나가기 쉬움 → 마법 유리 |
| 방어형 | 30 | 0 | 5 | 단단한 껍질 → 마법 유리 |
| 밸런스형 | 0 | 0 | 10 | |
| (새) 마력형 | 0 | 30 | 5 | 마법 저항 → 물리 유리. 정령·언데드 계열에 부여 |

- 등급 배율(정예 ×2, 보스 ×3)은 두 방어에 모두 곱한다(지금 규칙 유지).

### 3.3 몬스터 공격 타입

- 몬스터도 `AtkType`(물리/마법). 기본 물리. 원거리 마법형(`MonsterTable.Ranged=1` 중 정령·마법 계열)만 마법.
- **캐릭터 방어력은 하나**로 유지한다. 마법 공격은 방어력의 **절반만** 적용: `감소율 = (방어×0.5) / (방어×0.5 + 100)`. 방어 스탯을 둘로 나누지 않아 장비·펫·각인 체계를 건드리지 않는다.

---

## 4. 속성

### 4.1 종류 (원본 메이플 기준 7종)

원본 도감·스킬 표기: 불 · 얼음 · 전기 · 독 · 성 · 어둠 + 무속성. 다른 담당(스킬 개편)과 맞추기 위해 **ID·이름을 이 표로 고정**한다.

| Id | 표시 이름 | 원본 표기 | 색 (아이콘·숫자) | 기존 표기 변환 |
|---|---|---|---|---|
| `None` | 무속성 | 물리/무속성 | 회색 `#8A8F98` | 빈칸 |
| `Fire` | 불 | 불 | 주황 `#F26B2A` | 불 |
| `Ice` | 얼음 | 얼음 | 하늘 `#4FB6E8` | 얼음 |
| `Lightning` | 번개 | 전기 | 노랑 `#F2C230` | 전기 |
| `Poison` | 독 | 독 | 보라 `#9A5BD0` | 독 |
| `Holy` | 성 | 성 | 흰금 `#F5E6A8` | 신성, 성 |
| `Dark` | 어둠 | 어둠 | 남보라 `#5B4A8C` | 어둠 |

- `출혈`은 속성이 아니라 지속 피해 효과다 → `Element=None`, 지속 피해는 기존 `Bleed` 열로.
- 파랑·초록은 UI 의미색(선택·성장)이라 속성 색으로 쓰지 않는다 ([UI-Design-System](../UI-Design-System.md) 준수).

### 4.2 몬스터 속성 상성

| 단계 | 배율 | 화면 | 원본 대응 |
|---|---|---|---|
| 약점 | **×1.5** | 피해 숫자 위 작은 "약점" 표시 + 속성색 | "○속성 공격에 약함" |
| 보통 | ×1.0 | — | |
| 반감 | **×0.5** | 숫자 회색 | "○속성 공격 반감" |
| 무효 | **×0.1** | 숫자 대신 회색 "무효" | "○속성 공격이 통하지 않음" (원본은 1 피해. 자동 전투에서 전투가 멈추지 않게 0.1로) |

- 한 몬스터에 여러 개 가능(예: 주니어 발록 = 독 무효 + 성 약점).
- **데이터 원천:** `MonsterTable.Trait` 글자(예: "불 약점, 얼음 반감", "독 무효", "전기약점")를 변환 스크립트로 새 열 `Elem`(예: `Fire:W|Ice:R`)에 옮긴다. 글자 표기 흔들림("독반감", "전기약점")도 처리. 보스는 원본 몬스터 도감(`_Strings/MonsterBook.xml` 특이사항)을 근거로 채운다 ([Field-Boss-Map](../Boss/Field-Boss-Map.md) §1).
- SSD 원본 `Mob.xml`에 `elemAttr` 수치가 없어(§9) 도감 문구가 없는 몬스터는 **보통**으로 둔다. 설계상 넣는 속성은 `#ElemSource=설계` 로 표시해 원본과 구분.

### 4.3 캐릭터 쪽 속성

- 타격의 속성 = **그 스킬의 `Element`**. 기본 공격은 직업 기본 속성(`CharJobTable.Element`, 대부분 None).
- 직업별 기본 속성 **제안** (최종은 `Skill-Overhaul-Spec.md`):
  - 히어로 None(일부 스킬 불) · 비숍 **Holy** · 나이트로드 None(패시브 원본 '베놈' 계열 → **Poison** 지속 피해) · 보우마스터 None(피닉스·애로우 봄 계열 **Fire**) · 바이퍼 None(에너지 계열 **Lightning**)
- 얼음·어둠은 지금 5직업에 거의 없다 → 이후 직업 추가·장신구(판 안 '속성 부여')로 채운다. 보스 설계는 이 빈칸을 "반감을 피하는 편성" 문제로 쓴다(마뇽).

---

## 5. 명중 · 회피

원본 메이플은 버전마다 공식이 달라(레벨 차 보정 등) 그대로 쓰지 않는다. 지금 공식을 거의 유지하고 **마법 필중**만 더한다.

### 5.1 우리 공격이 빗나갈 확률

```
물리:  빗나감% = clamp(몬스터 회피 - 명중, 0, 60)
마법:  빗나감% = 0      (원본 클래식 규칙 차용: 마법은 빗나가지 않음)
```
- 몬스터 회피 = 타입 기본(§3.2 표) + 접두사 + 보스 패턴 보너스.
- 명중 = 망토 `Accuracy` + `Bonus("Acc")`. **파티 공용 → 구성원별**로 바꾼다(자기 편성 칸 망토, `EquipStatFor("Cape", 칸)`). 지금 망토 1칸 값만 쓰는 건 5-C 단계 임시 처리였다.
- 레벨 차 보정은 넣지 않는다: 캐릭터 레벨(1~40)과 몬스터 원본 레벨(1~200+)의 척도가 달라 오해만 생긴다.
- 상한 60%: 회피형 보스 + 명중 0이어도 전투가 끝나게.

### 5.2 몬스터 공격이 빗나갈 확률

```
빗나감% = min(75, 암흑 등 디버프 확률) → 아니면 캐릭터 회피%(각인·펫·잠재 Evade 합, 상한 50) → 아니면 피격 무효
```
- 지금 순서 그대로. 상한만 명시(회피 50%, 합계 75%).
- 보스의 **필중 패턴**(예고 후 쓰는 큰 공격)은 회피·암흑을 무시한다(보스 문서 §5).

---

## 6. 최종 피해 공식 (한 곳에서 계산)

```
피해 = 기존 피해(공격력·숙련도·각종 %)
     × 방어 감소   ← 타격 타입에 맞는 몬스터 방어(DefP 또는 DefM), 방어 무시 % 적용
     × 속성 배율   ← 약점 1.5 / 보통 1 / 반감 0.5 / 무효 0.1
     × (1 + 약점 추가 피해%)   ← 약점일 때만 (새 효과 키 WeakDmg)
     × (1 + 그로기 피해%)      ← 보스 보호막이 깨진 동안 +50%
```
- **구현 위치:** `FinalDouble` 앞에서 부르는 새 메서드 `CombatMul(target, ctx)`. `_T.dmgCtx` 에 `dmgType`(P/M), `elem`(Id), `target`(엔티티)을 넣는다. 지금 `ActorHitDamage`/`PlayerHitDamage` 의 `monDef` 계산은 `CombatMul` 로 옮긴다(이중 적용 금지).
- `dmgCtx.elem` 는 지금 boolean → **문자열 Id**로 바꾸고, 유물 `ElemDmg`(속성 피해 +%)는 `elem ~= "None"` 일 때 적용(지금과 같은 의미).
- 지속 피해(화상 등)도 그 스킬의 속성을 따른다(`dmgCtx = { dot = true, elem = "Fire" }`).

---

## 7. 기존 장비 · 각인 · 펫 · 유물과의 관계

| 기존 효과 | 개편 후 |
|---|---|
| 방어력·DefPct (갑옷·투구·펫 분홍색토끼) | 그대로. 물리 100%, 마법 50% 적용 |
| DefIgnore (돌파·슬롯 마일스톤) | 물리·마법 방어 모두 무시 |
| Accuracy (망토 1단계 장비 능력치) | 그대로, **구성원별 적용**으로 변경 |
| Evade (펫 디노 걸·각인) | 그대로 + 상한 50% |
| EliteDmg / BossDmg (잠재·펫 반반·피에르) | 그대로. 속성 배율과 곱연산 |
| ElemDmg (유물 눈의 여왕) | 무속성이 아닌 타격에 적용 (지금과 동일 의미) |
| **새 효과 키** | `WeakDmg`(약점 공격 피해 +%), `ShieldDmg`(보호막 피해 +%), `ElemPierce`(반감을 보통으로 보는 확률%) — 망토 잠재·펫·유물 후보. 이번 단계에서는 **키만 만들고** 옵션 배치는 하지 않는다 |

- 능력치 숫자(공격력·체력·방어력)는 바꾸지 않는다. 기존 저장 데이터·성장 수치는 그대로 (CLAUDE.md "테마 교체만으로 성장 수치 변경 금지" 취지).

---

## 8. UI 표시 (텍스트 최소)

| 위치 | 표시 | 툴팁 |
|---|---|---|
| 캐릭터 화면 (편성 칸·캐릭터 카드) | 직업 아이콘 옆 **공격 타입 아이콘**(물리 = 검, 마법 = 별 지팡이) 1개 + 기본 속성 원형 점 1개 | "물리 공격 · 명중이 필요합니다" / "마법 공격 · 빗나가지 않습니다" |
| 스킬 툴팁 | 제목 줄 오른쪽에 속성 아이콘 | 속성 이름 |
| 지역 선택 → 등장 몬스터 칸 | 몬스터 그림 오른쪽 아래 **약점 속성 점**(최대 2), 반감·무효는 그림 안 함 | 몬스터 이름 / 약점: 불 / 반감: 얼음 / 무효: 독 |
| 전투 중 피해 숫자 | 약점 = 속성색 숫자 + 작은 "약점", 반감 = 회색, 무효 = 회색 "무효" | — |
| 보스 체력바 | §보스 문서 8장 | |

- 아이콘 리소스: 원본 메이플 데이터에서 찾는다(`_Resources/UI/...`의 속성 아이콘, 없으면 각 속성 대표 스킬 아이콘: 불=플레임 기어류, 얼음=콜드빔류, 번개=썬더볼트류, 독=포이즌 브레스류, 성=홀리 애로우, 어둠=다크 사이트류). 찾기 전까지는 속성색 원형(16×16) 임시. 찾으면 `output/preview/`에 원본·MSW 비교 미리보기를 만든다(사용자 확정 규칙).

---

## 9. 데이터셋 스키마 (변경·추가)

모두 `RootDesk/MyDesk/Data/` (UTF-8 BOM CSV + 같은 이름 `.userdataset`). 열 추가는 기존 열 뒤에 붙여 기존 코드가 깨지지 않게.

**ElementTable.csv (새)**
```
Id,Name,Color,IconRUID,#Desc
None,무속성,8A8F98,,
Fire,불,F26B2A,,
Ice,얼음,4FB6E8,,
Lightning,번개,F2C230,,원본 표기 '전기'
Poison,독,9A5BD0,,
Holy,성,F5E6A8,,
Dark,어둠,5B4A8C,,
```

**MonsterTable.csv (열 추가)** — `AtkType`(P/M, 빈칸=P), `DefP`, `DefM`(빈칸=타입 기본), `Elem`(`Fire:W|Ice:R|Poison:N`; W=약점 R=반감 N=무효), `#ElemSource`(도감/Trait/설계)

**MonsterTypeTable.csv (열 추가)** — `DefP`, `DefM`. 행 추가: `마력형,1,1,5,0,30,마법 저항`

**MonsterPrefixTable.csv (행 추가)** — `마력 결계,DefM,60,마법 방어 +60`

**CharJobTable.csv (열 추가)** — `AtkType`(P/M), `Element`(기본 속성 Id)

**JobSkillTable.csv (열 추가·변경)** — `DmgType`(P/M/빈칸=직업). `Element` 값을 한글 → Id로 변환(불→Fire, 얼음→Ice, 신성→Holy, 출혈→빈칸)

**RunConfig.csv (키 추가)**
```
ElemWeakMul,1.5
ElemResistMul,0.5
ElemImmuneMul,0.1
MagicDefRatio,0.5      # 캐릭터 방어력이 마법 공격에 적용되는 비율
MissCap,60             # 물리 빗나감 상한
PlayerEvadeCap,50
MagicNeverMiss,1
```

---

## 10. 바꿀 파일과 순서 (구현 단계)

각 단계 끝에 `[VRF]` 로그로 짧게 기능만 확인(밸런스 측정 금지, 01-Agent-Rules 테스트 범위).

| 단계 | 내용 | 파일 |
|---|---|---|
| S1 데이터 | ElementTable 새로 만들기, MonsterTable `Trait`→`Elem` 변환 스크립트(노드, 스크래치패드에서 실행), 열 추가, JobSkillTable Element Id 변환, RunConfig 키 | `Data/*.csv`, `.userdataset`(기존 형식 복사·새 uuid) |
| S2 읽기 | 몬스터 정보에 `atkType·defP·defM·elem` 로드, 직업 `atkType·element`, 스킬 `dmgType` 로드 | `TowerRunManager` `LoadMonsterTables`, `LoadJobTables`, `LoadRunConfig` |
| S3 계산 | `CombatMul(target, ctx)` 추가, `FinalDouble` 호출 전 적용. `ActorHitDamage`·`PlayerHitDamage`의 `monDef` 제거→`CombatMul`. `DmgCtxOf`가 `dmgType·elem·target` 채우기. 마법 필중(`ActorAttack` 빗나감 판정) | `TowerRunManager.mlua`, `HeroCombat/BishopCombat/JobCombat.mlua`의 dmgCtx 세팅 부분(13개 `FinalDouble` 호출 지점 확인) |
| S4 방어 | 몬스터 마법 공격 시 방어 50% 적용, 회피 상한 | `DoMonsterAttack` |
| S5 명중 구성원별 | `Accuracy` → `ActorAcc(actor)` | `TowerRunManager` 3421행, `ActorAttack` |
| S6 표시 | 피해 숫자 색·"약점/무효" (서버가 `ShowDamage` 에 플래그 전달), 지역 선택 몬스터 약점 점, 캐릭터 화면 공격 타입 아이콘, 툴팁 문구 | `Tower/TowerPlayer.mlua`(숫자), `Lobby/RegionSelectLogic.mlua` `FillMonsters`, `Character/CharacterUILogic.mlua`, `Tooltip/TooltipLogic.mlua`, `.ui`는 UIBuilder로만 |

- `PlayerEngrave.mlua`, `RelicUILogic.mlua`, `ui/RelicUI`, `Docs/UI-Design-System.md` 는 다른 세션 작업 중 → 건드리지 않는다.

---

## 11. 확인 못 한 것 / 정할 것

- **원본 수치 부재:** SSD `Mob/Mob.xml`에 몬스터별 `info`(elemAttr·PADamage·MADamage·PDDamage·MDDamage·acc·eva)가 없다. 속성은 도감 문구와 프로젝트 `MonsterTable.Trait` 로만 근거를 댔고, 물리/마법 방어 값은 **설계값**이다. 원본 수치가 들어간 데이터가 생기면 `DefP/DefM`·회피를 원본 비율로 다시 뽑을 수 있다.
- 직업별 기본 속성·스킬별 속성은 `Skill-Overhaul-Spec.md` 와 최종 대조 필요.
- 속성 아이콘 RUID 미정 (구현 S6에서 원본 데이터 → msw-search).

출처: [Honkai: Star Rail Wiki — Toughness](https://honkai-star-rail.fandom.com/wiki/Toughness), [Icy Veins — HSR Combat Basics](https://www.icy-veins.com/honkai-star-rail/combat-basics), [AppGamer — Blue Archive Attack and Defense Types](https://www.appgamer.com/blue-archive/attack-and-defense-types-guide), [nikke.gg — Anomaly Interception: Kraken](https://nikke.gg/anomaly-interception-kraken/), [nikke.gg — Damage Formula](https://nikke.gg/damage-formula/), [Ellia Bot — Summoners War damage calculations](https://elliabot.neocities.org/game_mechanics/damage_calculations), [NiaMeowDB — MapleStory Classic accuracy sim](https://meowdb.com/msclassic/es/accuracy-sim), [ayumilove — MapleStory Formula](https://ayumilove.net/maplestory-formula/). 메이플스토리M·에픽세븐·원신·쿠키런: 작성자 지식(게임 내 시스템 일반 설명) 기준.
