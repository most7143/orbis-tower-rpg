# 스킬 V2 구현 진행 (Impl-Progress)

- 시작 2026-11-25 · 구현 담당 "skill" · 커밋 안 함
- 기준 문서: `Skill-FX-Tech.md`, `Jobs-Hero-Bishop-NightLord.md`, `Jobs-Bowmaster-Viper.md`
- 사용자 확정 결정 1~14 (모두 권장안) — 지시서 그대로.

## 0. 구현 중 정한 것 (근거)

| 항목 | 정한 것 | 근거 |
|---|---|---|
| 칸 키 이름 | 내부 키는 `Passive(=패시브1)`, `P2`, `P3`, `Main(=액티브1)`, `Sub(=액티브2)`, `Tech(=액티브3)`, `Hyper`, `Ult` | 돌파 저장 키가 `직업.칸`이라 Main/Sub/Tech를 바꾸면 기존 저장 돌파가 사라진다. 화면 표시 순서만 패시브1~3 → 액티브1~3 → 하이퍼 → 궁극기 |
| 5차 V1~V3 칸 | CharSlotTable·CharSkillSlotTable에서 행 삭제, 각성(Awk) 열 비움 | 결정 1·2 |
| 타격 시각 | `SkillActionTable`에 `HitMs, HitCount, HitGapMs, FxId` 열 추가. `HitMs`가 있으면 **서버가 시전 순간부터 HitMs 뒤에 직접 타격**(클라이언트 RequestHit 왕복 안 씀) | Skill-FX-Tech §4-3 |
| 시전 연출 | 새 표 `SkillFxTable`(FxId별 타임라인). 클라이언트가 시전 신호를 받은 순간을 0ms로 재생 | Skill-FX-Tech §4-2 |
| 받는 피해 증가 | 흉터·표식·쇼다운은 몬스터별 종류 칸, 종류끼리 곱, 합계 최대 1.3배(`TowerRunManager.VulnMul`). 숫자 표시 전(`FinalDouble`)에 곱해 숫자와 실제 피해가 같다 | 결정 7 |
| 파티 피해 증가 | 버프 키 `PartyDmg`(판 전체 버프 = 파티 공용), `FinalDouble`에서 곱 | 설계 문서 1절 |

## 1. 진행 상황

- [x] 단계 1: FX 재생기·스킬 표 구조·8칸·캐릭터 화면 8칸
- [x] 단계 2: 히어로 (중단점 1 — 짧은 실행 확인 완료, 4절)
- [x] 단계 3: 비숍·나이트로드·보우마스터·바이퍼 완료 (4절)
- [x] 결정 13: `Docs/Job-Redesign/00-Master-Spec.md` §3 진화표 V2로 갱신

## 2. 바꾼 파일

| 파일 | 내용 |
|---|---|
| `Data/SkillActionTable.csv` | 열 `HitMs, HitCount, HitGapMs, FxId` 추가. 히어로 V2 행: H01·H02·H21(새 행)·H08·H30(새)·H22(새)·H10·H07·H06·H13·H09 — HitFx를 새 클립으로, CasterFx는 비우고 FxId 시간표로 |
| `Data/SkillFxTable.csv` + `.userdataset` (새) | 시전 연출 시간표. 열 `FxId,Seq,AtMs,Kind,Clip,Src,Layer,Flip,Offset,Scale,DurMs,LoopClip,Pick,Jitter,Width`. Kind 구현: cast·castFix·attachLoop·shake |
| `Data/CharSlotTable.csv` | 직업마다 8칸(Passive·P2·P3·Main·Sub·Tech·Hyper·Ult), V1~V3 삭제, Awk 비움. 히어로 V2 칸 스킬. 다른 직업 P2·P3는 빈칸(각 직업 차례에 채움) |
| `Data/CharSkillSlotTable.csv` | V1~V3 행 삭제, 히어로 행 V2(쿨·마나·발동 방식) |
| `Data/JobSkillTable.csv` | 히어로 수치·툴팁 V2. 새 행 H30 스피릿 블레이드, H31 파이널 어택, H32 어드밴스드 파이널 어택, H33 콤보 인스팅트(패시브). H21·H22 Base(동작 기준) 해제 |
| `Tower/TowerPlayer.mlua` | `PlayAttackMotion`에 `fxId, serverHit, fxRate` 인자. HitMs가 있으면 서버가 그 시각에 직접 타격(왕복 없음). 새 `SkillFxRows`·`PlaySkillFx`·`PlayFxRow`(클라이언트 시간표 재생기) |
| `Tower/HeroCombat.mlua` | V2 열 읽기, `NewCast`에 hitMs·fxId·간격, 칸 키 8개, 새 `MakeCastV2`·`ApplyBuffV2`·`ApplyScar`, 데스폴트 콤보 규칙, 발할라 검격 2타·8회 |
| `Tower/TowerRunManager.mlua` | `V2VulnCap`, `AddVuln`·`VulnMul`, `FinalDouble`에 `PartyDmg`·받는 피해 배율, 판 시작 초기화 (최소 수정) |
| `Tower/JobCombat.mlua`, `BishopCombat.mlua` | 칸 키 목록만 8칸 키로 |
| `Tower/BattleHUDLogic.mlua` | 툴팁 칸 종류 이름(액티브·패시브·하이퍼·궁극기) |
| `Tarot/TarotLogic.mlua` | P2·P3 = 패시브 카드, V1~V3 분기 삭제 |
| `Character/CharacterUILogic.mlua` | 칸 8개 순서·테두리 등급 |
| 캐릭터 화면 UI (UIBuilder) | Skill9 삭제, Skill1~8 x −336~336 간격 96 |
| `SkillTest/SkillTestManager.mlua` | `PlayAttackMotion` 새 인자 맞춤 |
| `output/preview/v2_impl_hero_fx.png` | 히어로 원본 최대 프레임 + 올린 클립 썸네일(4개 확인) |

### 정한 것 추가

- 쿨타임 발동(Cooldown) 스킬은 지금 엔진에서 마나를 쓰지 않으므로 스피릿 블레이드·스카링 소드·발할라·레이지 업라이징의 마나는 0 (쿨마다 자동 시전).
- 스피릿 블레이드 '받는 피해 감소'는 판 버프라 파티 전체에 적용됨(설계는 히어로만) — 캐릭터별 칸이 없어서.
- 콤보 카운터 수치는 기존 고유 패시브(P_Hero·CharPassiveTable) 그대로.
- 콤보 데스폴트 screen 48f는 크기·모바일 부담으로 이번엔 안 올림.
- 빈 대기 프레임(원본 PNG 없음, 예 브랜디쉬 effect 0번 180ms)은 투명 4×4 스프라이트 `0ed30a40ff5944fb973f1908641ea47d`.
- 업로드 도구: 스크래치패드 `v2/fxbuild.py` (원본 XML → 스프라이트 JSON → 결과 RUID → 클립 JSON).

## 3. 올린 RUID

| 클립 | RUID | 원본 |
|---|---|---|
| FxV2_1121008_effect | ea354f925e2a42638dfc848a2ccfe343 | 112.img/skill/1121008/effect (18f) |
| FxV2_1121008_hit0 | c04bd755e6334f8eb47a3c5037795cf7 | 112.img/skill/1121008/hit/0 (6f) |
| FxV2_1001005_effect / hit0 | ce2452fe680d4036b9966472cd60da48 / a0c06e4e3cd64be38b1a6c21a1b71290 | 슬래시 블러스트 |
| FxV2_1101011_effect / hit0 / hit1 | e462025d37db4429a279df45903dedb3 / 823ef598083443869a1829e68ffa1450 / 5603719c064541c497b80a9df13e98eb | 브랜디쉬 |
| FxV2_1111010_effect / effect0 / hit0 | d3e783cc2af940bf94af0954a822e1b4 / ab5fc587565b496999981f54e4451e7f / 85d90bc45bd94367b16928ca9fdbe376 | 브레이브 슬래시 |
| FxV2_1101006_effect / affected | 0366b374b24147e5bfc9881f24dab0c5 / d960b518dd634c5f9c8dc0183f7ad7a6 | 스피릿 블레이드 |
| FxV2_1111003_effect / mob | daa8638e9fcf47979c78001c84232bd7 / 009c13421b474ba08094d6171994fd25 | 스카링 소드 |
| FxV2_1121054_effect / effect0 | 2af9e28234f842ac8887b44d705b0079 / f7c2097087bd427885eec9a3caad97a3 | 발할라 |
| FxV2_1121055_summonattack1 | fea9bf12ec49439b98b3a0320f35fac4 | 발할라 검격 (지금은 기존 팩 attack1~6 사용, 예비) |
| FxV2_1111016_effect / hit0 | 65986051a76f4f4fbd56cc0b9c9ea20e / e9bb81f0eb9f45efa3b808e3a8fe03bc | 오라 블레이드 |
| FxV2_1121015_effect / effect0 / hit0 | 9c3e98826bde4515926bd4142be4af99 / ae6e8359357d485287e8446c1dff4dab / 50f5d0c7d28b4a0196b8d140e5cefb56 | 인사이징 |
| FxV2_400011027_effect / hit0 | bd80e5dd22f347a3b83b0593af088d8b / 8fbc190b16c241878cbaebf1f91e9d0a | 콤보 데스폴트 |
| FxV2_1121052_effect / effect0 / effect1 / hit0 | 65bdb2bd7db94c10a48260ca707cd514 / 13e885d5284649bebedb3ebc4a300e35 / de2fbb7f0eed4e468569de62cab4fe87 / be7c1fa228be45f780c540c7e8f3257a | 레이지 업라이징 |
| FxV2_1100002_hit0 / FxV2_1120013_hit0 | ea8937495a8746c3bb0dd94e4307177b / 7e50e9e144dd4d7eaa0413111a2f10fe | 파이널 어택 / 어드밴스드 |
| FxV2_400011074_effect / hit0 | 0fd0109325b34c28bf41f6d13bac720e / 6e1dc88ad07c4893938a6c4e5d72e54e | 공간의 상처 |
| 아이콘(스프라이트) | eacc4e8298614af7bba0661885376ba5 / 87c867d67c51474ea31f8c441425e2ab / 1f475cdd84c3484f8209de529b178da4 | 1101006 / 1100002 / 1120013 icon |

## 4. 검증 기록

### 2026-11-25 히어로 (중단점 1) — 메이커 1회 실행
- refresh 후 빌드 로그: 오류 0 (Info 1401·Warning 45, 경고는 기존 LWA-1111 종류).
- 칸 표: `DebugSkillSlots` 차수 1~5 = `Main=H01` / `+Sub=H30` / `P2=H31,Main=H21,Sub=H22,Tech=H07` / `P2=H32,Main=H08,Sub=H10,Tech=H06,Hyper=H09` / `+P3=H33,Tech=H13,Ult=H12` (설계표와 같음).
- 판(히어로 5차 임시 설정) 장착: `H08,H10,H13,H09,H32,H33,H12`, 궁극기 게이지 칸 정상.
- **서버 기준 타격 시각** (`hit apply reason=timeout wait=`): 발할라 0.191 (표 180ms), 레이지 업라이징 0.813 (810), 콤보 데스폴트 1.575 (1560), 레이징 블로우 0.249 (240) — 프레임 단위 오차 ±15ms 안.
- **클라이언트 시간표 재생**: `[VRF] skill fx H09 rows=4`, `skill fx H08 rows=1` (시전 신호를 받은 순간 재생).
- 버프: `v2 buff H10 PartyDmg=12 Crit=10 dur=12.0`.
- 콤보 소모: `H09 n=10 mul=2.20`, `H06 n=2 mul=1.95`(=1.5×일반 몬스터 1.3).
- 자동 전투 중 히어로 스킬(H08·H09·H10·H12·H13) 자연 시전, 오류 로그 0.
- 테스트 후 되돌림: 히어로 캐릭터 Lv13·3차로 복원(경험치는 0이 됨 — DebugSetChar 동작), 무적 해제, 플레이 종료.


### 2026-11-25 비숍 마무리 (담당 2)

- 원본 클립 28개 업로드(스프라이트 381장, 액티브 effect·hit 위주 + 설치·붙음 꼭 필요한 것만). 아이콘 3개(B30·B31·B32).
- 클립 RUID (원본 노드 → RUID): `2001008_effect` 6bab31b30e644d89be245c48426f1f92 · `2001008_hit0` cf110cd4c60942bcbedcda2db958b58d · `2301005_effect` 788043eb8a3345698ce28b981f3ff268 · `2301005_hit0` 9bd3dacc9a1f45168e841dc44efaff26 · `2311004_effect` fce74e9c8ace49e598587c6c96750ccf · `2311004_hit0` 250a2abc854548019ad450c03e912f24 · `2321007_effect` fd904659a9e9429fbbc7ad06464b3ec9 · `2321007_hit0` da7261616fa9484994f2e05bd75ca5ed · `2301002_effect` ce80510e32bf4e948dc43e2d054f8283 · `2301002_affected` 5c530ea362ca456ba54e0a9d157ed4ab · `2311009_effect` 52e7923e197e47adb2d51db06d8ecc11 · `2311009_repeat` 7ce7db40234e42028c8f05a8eba57667 · `2311015_effect` cae33aaa2ffd4a208eb2983bdfc1987e · `2321008_effect0` 622706e12dbc4b28bedfa388f6e0b8cd · `2321008_hit0` 4f9276402d1e4f4396695fad734bc23e · `400021086_keydown` 0d777c9d3dcd4b939d41b33dea022a88 · `400021086_hit0` 0d891247e5a844a99ca8a45b7de4052e · `2321052_special` 990d46c71e954c2baa56ea4c42697ad0 · `2321052_hit0` 05bf7a0e518f49fda605ccd032d305c2 · `2321055_affected` b3a924dc70b64028a7b54c6b3b155f74 · `400021003_effect` b3a91cbab77c401fb405461bc35db192 · `400021070_effect` 80a25c0278a9409eb368270dc920ad3b · `400021070_hit0` 73c1120707ae454b95742c28aaab7154 · `2311011_tile0` 276eeaae79544e9ebdedb167de203dd8 · `2311011_affected` 955c1391cc5a4ae2912547b233f1b624 · `2321015_tile0` c1af11597bfe4f61ae3bd3c188e8994d · `2321015_affected` 0cd3e18bc3c44c59b35255bc375732e0 · `2321006_effect` 4ca8b06651724cacb2d9a1cf779e6954
- 아이콘: B30 홀리 파운틴 `178005ae1be943d6bbfe2e2f08cce755`, B31 홀리 워터 `4c1e92f509dc448da4f7a72f028c28bb`, B32 리저렉션 `f2e47e08409a49c997201a3e9c061063`
- 데이터: SkillFxTable B01·B02·B03·B06·B21·B10(문 + 1020ms 흔들기)·B14·B15(keydown 붙는 반복 2.4초). SkillActionTable B01·B02·B03·B06·B07(=제네시스 B21 동작 기준, HitMs 1080/60)·B10(MobFx=축복 2321055 affected)·B14·B15의 HitFx를 새 hit 클립으로, CasterFx는 FxId로 옮김. B04·B05·B12 CasterFx 새 effect.
- 새 스킬 행 B30 홀리 파운틴(3차 P2)·B31 홀리 워터(4~5차 P2)·B32 리저렉션(5차 P3), Trigger=Passive. CharSlotTable·CharSkillSlotTable P2·P3 채움. B10 BuffTime 15(생명의 축복).
- 코드 `BishopCombat`
  - `TickHolyPassive`: 아군 HP 60% 이하 + 성수 3 → 자동 설치(재사용 6초, 동작 없음). 파운틴 = tile 루프 8초·1초마다 파티 3% + affected / 워터 = tile 루프 5초, 0.27초에 가장 다친 아군 15% + 매초 파티 2%.
  - `TickShell` 재작성: 2차 힐(B20이 B04 동작 기준) = 아군 80% 이하 → 0.54초에 파티 12% + affected. 3차~ 매직쉘 = 보호막 없는 아군이 있으면 파티 전원 0.3초에 보호막·회복 10%(3차 3회 8초, 4차 5회 12초). `OnShellEnd` = 4차에서 깨지거나 시간이 다 될 때 HP 8%.
  - `ResurrectPct`: B32 장착 시 층(Floor)마다 1번, 쓰러지려는 아군 HP 40% + 막기 99회 2초(무적) + 빛. B32 장착 중 비숍 피해 +10%(Strike).
  - 헤븐즈 도어 생명의 축복: `SetDeathWardPct(15초, 30%)`.
- 코드 `TowerRunManager`(최소): `SetDeathWardPct`·`WardHp`(축복이면 최대 HP %, 아니면 1)·`ResurrectPct`(비숍에게 물음) — 메인·구성원 쓰러짐 처리에서 축복 다음 순서로 호출. `HasBlock`·`EndBlock`, `TryBlock`이 깨질 때 `owner:OnShellEnd(e)`도 부름.
- 검증(메이커 1회 플레이, 비숍 5차 임시 편성 후 원래대로 저장 복구): 빌드 오류 0. 로그 `v2 bishop holy passive B31 ... left=5 dur=5`(자연 발동), 리저렉션 `main hp=435`(=40%)·같은 층 두 번째는 쓰러짐(1회 제한 확인), 축복 `mate1 hp=219/731`(30%), 매직쉘 4차 `blocks=5 dur=12 enh=true` + 12초 뒤 `shell end heal` 3명, 시간표 `skill fx B10/B14/B06/B15` 재생. 런타임 오류(attempt to·LEA) 0.
- 확인 못 함: 3차 홀리 파운틴(4~5차만 확인), 2차 힐 실제 발동, 이펙트 화면 크기(스크린샷은 이동 중이라 이펙트 없음 → 5단계에서 다시).
- 미리보기: `output/preview/v2_impl_bishop_fx.png`

### 2026-11-25 나이트로드 (담당 2)

- 원본 팩 재사용(업로드 안 함, MSW 원본 팩 `skill/<img>/skill/<id>`): 럭키 세븐 4001344 effect·hit/0, 슈리켄 챌린지 4111015 effect·effect0·hit/0, 풍마수리검 400041020 effect·effect0·hit/0, 비전서 400041038 effect·summon/summoned·stand·die, 표식 4100011 mob·4120018 mob, 스로우 블래스팅 400041062 effect·hit/0, 쉐도우 파트너 4111002 special/<동작> 24개(`TowerPlayer.PartnerClips`). 팩 조회 도구 스크래치패드 `v2/packq.py`(결과 `packs.json`).
- 새로 올림(팩 없음) 클립 16개·스프라이트 196장: `4111010_effect` 80725dcc21544a64aa69e86d30892cce · `4111010_hit0` 083fcf87cca44b209979620e6e4b0c7c · `4121013_effect` eab225f549994cfe8a5e5fdc39629e5a · `4121013_hit0` 48dad679ded14274a15555eec4d889b1 · `4101013_effect` a814d43f8d974881aaa0ab41d6b21e1c · `4101014_effect` 3bad1078bbbd4ef2bf07dde4803afdce · `4101014_hit0` b2b5663316fd4ebba57d7e55219f92db · `4121017_effect` 48f30d301cb24fce861a4e786708214e · `4121017_hit0` c96d5b98569d485895ab29e274ae5b55 · `4121017_mob` 466400584a5c4185b0d38e436fc4db42 · `4111007_summonstand` d10160b3596b4e1f9182640b1f72ee4f · `4111007_hit0` 2945a3857f9748f482a07837c03dac13 · `4121016_effect` e767a7e6bb8d427e8eaaf7e15cbfcf1d · `4121016_hit0` 10275be6f0ae47dab2ea64a0261e8221 · `4121052_effect` e7f8d54434c646a6a8927da9f7bbc598 · `4121052_hit0` aa25b4dd91f548bcb0f1df88a539f425. 아이콘 N30 스로우 블래스팅 `09f00dcb17bc49778646ef444ae4d68c`.
- 칸: Main N01→N01(×1.14)→N03 트리플 스로우→N06→N06 / Sub -→N20→N22→N08→N08 / Tech -→-→N05 다크 플레어(쿨 자동)→N07→N14 / Hyper N10 / Ult N15 / P2 N04 쉐도우 파트너(3~5차, Passive) / P3 N30 스로우 블래스팅(5차). N20·N22 Base 해제(전용 동작 행 추가). 수치·쿨·마나 = 설계 7절.
- `JobCombat`(나이트로드 분기)
  - 표식: `AddMarks` = 적마다 1개(머리 위 mob 반복 그림 + `AddVuln(mark, 8%/4차 12%)`), `MarkHit` = 공격 1번마다 표식이 있으면 `MarkBurst`(표창 2/3개 × 100/130%, 0.34초 뒤, 0.5초 재부착 금지), 없으면 40/50% 부착. 예전 중첩·DealTo 표식 배율 제거, 표식 최대 1.
  - 쉐도우 파트너: `PartnerMul` 3차 0.5/4차 0.65/5차 0.7. 스킬은 `ApplyCast`에서 타격마다 +0.09초 복사(ApplyHit, 숫자·hit 그림 포함), 기본 공격은 ExtraHit 0.09초. 분신 타격은 4차부터 확률 절반 부착만.
  - 스로우 블래스팅: 투척 적중 8번마다 0.9초 뒤 150%×3(5명).
  - 쇼다운: 맞은 적 8초 공격력 −15%·`AddVuln(showdown,10%)` + 0.33초 뒤 어둠의 수리검 3×(40%×2). 포 시즌: 필드 표식 수 × 10%(최대 60%) 피해, 적중 순간 모두 폭발. 써든레이드: 맞은 적 모두 표식.
  - 궁극기 비전서 `StartSecretBook`: 캐릭터 앞 위 두루마리 10초, 0.99초마다 표창 6개(60%), 10.36초 폭발 400%×3(12명).
  - 소환물 유지 그림 `SummonLoops`(N05 다크 플레어 stand).
- `TowerPlayer`(클라이언트): `PartnerOn`(@Sync) 켜지면 캐릭터 뒤(−0.45, +0.05)에 stand1 반복, 공격마다 `PartnerDelay` 0.09초 뒤 첫 동작 이름의 special 클립 + 시전 이펙트·시간표를 같은 위치 차이·알파 0.6으로 재생(`PlaySkillFx`/`PlayFxRow`에 shift 인자 추가).
- 검증(메이커 1회, 나이트로드 5차 임시 편성 후 저장 복구): 빌드 오류 0. 로그 `v2 nl mark on`(자연 기본 공격)·`mark burst n=3 pct=130`(자연)·`four seasons marks=1`·`partner copy N10 n=4 mul=0.7`·`throw blasting`·`secret book start`→10초 뒤 `end burst`, 클라이언트 `v2 partner follow action=swingO1 fx=N10`. 런타임 오류 0.
- 남은 점: 분신 그림이 궁극기 동작(alert)도 따라함(서버 피해는 안 따라함). 포 시즌 뒤 레이어 그림이 캐릭터를 덮어 보이는지 5단계에서 확인 필요.

### 2026-11-25 보우마스터·바이퍼 (담당 2)

- 원본 팩 재사용: 애로우 블로우 3001004, 바람의 시 3101005(hit/0~6 무작위), 애로우 플래터 3111013(prepare·keydown·hit), 애로우 레인 영역 400030002(tile/0·hit), 피닉스 3111005(summoned·stand·hit), 퀴버 풀버스트 400031028 effect, 윈드 오브 프레이 3121052(effect·effect0·mob·hit), 잔영의 시 400031020(effect·repeat·hit), 실루엣 미라주 400031053 hit, 퓨리어스 차지 400051042, 훅 봄버 5121016, 하울링 피스트 400051070, 전함 노틸러스 5121013, 넵투누스 5141500 hit·5141503 effect.
- 새로 올림 클립 13개·스프라이트 131장: `3121020_keydown` 4a07ae4cbb0b4665b20f87513f8e8a39 · `3121020_hit0` 85a8a128024f407c932eac831379530b · `400031002_effect` bf19efb4407448bb82ab28867869bcfe · `5001002_effect` 61e9337876994d28a9085ee27468be8a · `5001002_hit0` a51e9ec5a5dc4ad291538d92477e3330 · `5101012_effect` 6b668bfff773434e9ff8b28a2092410a · `5101012_hit0` 16da7466375b4d7f95c868437758aa17 · `5121007_effect` 073c5ebe2dce4be9b2dff7d775bc0018 · `5121007_hit0` 486146a9ffff4e568800b68e8276b52c · `5111009_effect` 0cb4c8aec53d4c91a304ed7e111e2b14 · `5111009_hit0` 9db863326cf1409293164a876f65b296 · `5111002_effect` 365c42a07048407bab7fd352bfb5a079 · `5111002_hit0` a6a43e982b864540a57ffa81daeaeae9. 아이콘 6개(N03 트리플 `ca6e8bfe0fbe40bea93e2ef2879ce12f`, A14 실루엣 미라주 `e01f636da26541ba89ece92882ebdfde`, A10 애로우 레인 `0c69cb8935ef4460a2457ec4c816c628`, A12 퀴버 풀버스트 `f1439dd7379045eaac836017985c20f5`, V15 넵투누스 `f2835a108aad4c1491fdfc80d87a183c`, V27 라이트닝 폼 `56d90b1440614f6384933d79bc479a2d`).
- 칸(설계 B문서 2절): 보우마스터 Main A01→A20→A20(×1.18)→A06→A06(×1.2) / Sub -→A05 플래터(설치 8초)→A05→A05(×1.44)→A10 레인 / Tech A04 피닉스→A04→A12 풀버스트 / Hyper A24 / Ult A15 잔영의 시 / P2 A08 샤프 아이즈(파티 치명·치명 피해 +8%, 4차~ +12%, 판 버프) / P3 A14 실루엣 미라주(액티브 시전 시 환영 화살 5발 45%, 1.5초 제한). 바이퍼 Main V01→V20→V20(×1.16)→V24→V24 / Sub -→V03 스크류 펀치→V03→V03(×1.25)→V14 퓨리어스 차지 / Tech V23→V26→V12 하울링 피스트 / **Hyper V25 전함 노틸러스**(1800ms 첫 타, 강진동 1800ms) / **Ult V15 리버레이트 넵투누스**(5141503 수룡 2배, 1140ms부터 16타) / P2 V22 럭키 다이스(층마다, 4차~ 40% 두 개·같은 눈 2배) / P3 V27 라이트닝 폼(20초마다 8초 치명 20%·공격력 15%). A20·A24·V20·V22~V27 Base 해제(전용 동작 행).
- **돌진 300ms 복귀**: 시간표 Kind `dash`(Offset x=거리 px, DurMs=돌진 시간) → `TowerPlayer.DashVisual`이 아바타 몸 엔티티 로컬 위치만 밀었다가 0.3초에 원위치(판정·편성 위치는 그대로). V03 150px/510ms, V14 200px/380ms.
- `JobCombat` 공용 수정: **HitGapMs가 있으면 타 간격으로 사용**(이전에는 HitGap 0.12가 쓰여 V2 간격이 무시됐음 — 나이트로드도 함께 적용). 소환물 유지 그림 A05·A04·A10·A15 추가.
- 검증(메이커 1회, 보우마스터·바이퍼·히어로 5차 임시 편성 후 저장 복구): 빌드 오류 0. 로그 `v2 bm sharp eyes party crit+12`, `v2 viper dice 6`, `v2 bm silhouette mirage`, `ult cast job=Bowmaster id=A15` / `job=Viper id=V15`, 클라이언트 `v2 dash visual dist=2.00 out=0.38`. 런타임 오류 0.

### 2026-11-25 히어로 추가 확인·수정 (담당 2)

- 로그로 확인: 스피릿 블레이드 `v2 buff H30 PartyDmg=8 DmgReduce=10`, 스카링 소드 `v2 buff H22 ... Scar=6` + 몬스터 받는 피해 칸 `scar=1` 활성, 파이널 어택 `v2 final attack H32 after H09`.
- **버그 수정**: 공간의 상처(H33)가 레이징 블로우 돌파 1을 얻으면 발동하지 않던 문제 → `MakeCastV2`에서 돌파 1일 때도 콤보 8 이상이면 발동. 수정 후 `v2 rift H33 n=3` 자연 발동 확인.

### 2026-11-25 연출 화면 확인 (스크린샷, output/verification)

- `v2_bishop_angelray.png`(루트 화면, 매직쉘 반복 그림이 3명에게 붙음), `v2_nightlord_quad.png`(포 시즌 효과), `v2_party_hero_bm_viper_1~5.png`, `v2_party_nl_hero_bishop_1~4.png`(전투 중: 시전 이펙트·몬스터 hit 그림·피해 숫자 함께 표시).
- 판단: 올린 원본 클립 크기는 기존 팩 클립과 같은 배율로 보임(2배로 커지는 현상 없음). 시전 이펙트·hit·숫자는 같은 순간에 보임. 쇼다운 챌린지 effect(816×412 원본)가 화면 절반 가까이 덮는 것은 원본 크기 그대로라 유지.
- 확인 못 함: 'back' 층(OrderInLayer −2)이 아바타 뒤로 그려지는지 화면에서 확실히 구분하지 못함(겹친 장면만 찍힘), 쉐도우 파트너 분신·돌진 복귀 움직임이 찍힌 장면 없음(로그로만 확인), 화면 흔들기 체감.

### (히어로) 확인 못 한 것
- 화면에서 이펙트 크기·위치(올린 원본 PNG가 기존 팩 클립과 같은 크기로 보이는지)와 뒤 레이어(OrderInLayer −2)가 실제로 캐릭터 뒤에 그려지는지 — 스크린샷 미요청이라 안 봄.
- 스카링 소드 흉터·파이널 어택·공간의 상처·스피릿 블레이드는 시험 루프가 그 차례까지 가기 전에 정리해 로그 증거 없음(코드만 검토).
- 카메라 흔들기 세기(Scale 0.06~0.08) 체감.
- 캐릭터 화면 8칸 툴팁 실제 표시.

## 5. 다음에 이어서 할 곳

- 다섯 직업 V2 칸·수치·연출·짧은 실행 확인 완료(4절). 남은 일:
  1. 화면 확인 보강: 쉐도우 파트너 분신·바이퍼 돌진 복귀·'back' 층 그림을 한 장면씩 단독 시전으로 찍기(몬스터 체력 고정 + 한 직업만 반복 시전 프로브는 4절 방식 그대로).
  2. 단순화한 것(설계보다 약함): 보우마스터 퀴버 카트리지 마법/흡혈 화살 모드·플레시 미라주 잔영, 실루엣 미라주 '큰 피해 50% 감소', 바이퍼 씨 서펜트·서펜트 스톤·어썰트, 라이트닝 폼의 에너지 오브, 비숍 디바인 퍼니시먼트 3단 연출(keydown 루프만), 엔젤레이 유도·관통, 피스메이커 오브제 이동, 제네시스 기둥 6개, 프레이 반복 구간(repeatIdx). 나이트로드 분신 그림이 궁극기 동작도 따라함.
  3. 원본 팩 클립은 예전 판이라 최신과 프레임이 다를 수 있음(나이트로드·보우마스터·바이퍼 일부). 필요하면 `v2/fxbuild.py`로 최신 원본을 올려 교체.
  4. 히어로 캐릭터 경험치: 앞 담당의 DebugSetChar로 0이 된 상태(Lv13·3차는 유지). 원래 경험치 값 기록이 없어 되돌리지 못함.
- 시험 방법(재사용): 메이커 play → 서버 스크립트로 `PlayerRoster._T.chars` 백업(`_T.vrfBak`) 후 직업 5차·편성 변경, LobbyManager로 탑 이동, 클라이언트 스크립트로 카드·포탈 자동 선택, 끝나면 백업 복구 + `SaveRoster`·`SaveSkills`. 업로드 결과 RUID는 `v2/getres.py`(하위 담당 대화 기록에서 꺼냄)로 파일 저장.
