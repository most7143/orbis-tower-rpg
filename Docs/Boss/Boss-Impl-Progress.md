# 보스 스테이지 구현 진행 기록 — B1 (데이터·MSW 리소스)

- 작성일: 2026-11-24
- 기준 문서: [Boss-Stage-Spec.md](Boss-Stage-Spec.md) §9·§10 B1, [Field-Boss-Map.md](Field-Boss-Map.md), [field_boss_map.csv](field_boss_map.csv), [Map-Import-Test.md](Map-Import-Test.md)
- 상태: **B1 데이터·리소스 준비만 완료.** 전투·스킬·층 진행 스크립트(`.mlua`)는 고치지 않았다. 커밋 안 함.
- 미리보기: [../../output/preview/field_boss_resources.png](../../output/preview/field_boss_resources.png) — 32종, 왼쪽 원본(SSD `_Resources/Mob/_Canvas`) · 오른쪽 MSW 팩 stand 썸네일. 직접 열어 32종 모두 같은 몬스터임을 확인했다.

---

## 1. 만든 / 바꾼 파일

| 파일 | 내용 |
|---|---|
| `RootDesk/MyDesk/Data/MonsterTable.csv` | **행 추가만** (id 685~712, 28행). 기존 열·행 그대로. 새 행: `Type 밸런스형`, `Trait 필드보스`(표시용 메모, 코드에서 안 읽음), `OrigHp/OrigExp 0`(코드에서 안 씀), `Ranged 0`, `AttackRange 1`, `AtkType·DefP·DefM` 빈칸(타입 기본). `Elem` 은 도감 근거 4종만 — 주니어 발록 `Poison:N\|Holy:W`, 피아누스 `Fire:W`, 그리프 `Poison:W\|Lightning:R`, 마뇽 `Fire:R\|Ice:W` (`#ElemSource 도감`) |
| `RootDesk/MyDesk/Data/FloorTable.csv` | 열 추가 `FieldBossId, BossBgRUID, BossTileSetRUID, #BossBgSourceMapId`. 97층 모두 `FieldBossId` 채움. 배경은 원본 필드보스 맵이 있는 38층 중 37층 채움(§4·§4.1), 나머지는 빈칸 = 층 배경 그대로 |
| `RootDesk/MyDesk/Data/BossTable.csv` + `.userdataset` (새) | 보스전 5종 |
| `RootDesk/MyDesk/Data/BossStepTable.csv` + `.userdataset` (새) | 단계 1~5 배율·보상 (Spec §6.2·§7) |
| `RootDesk/MyDesk/Data/BossConfig.csv` + `.userdataset` (새) | `Key,Value,#Desc` 형식 |
| `output/preview/field_boss_resources.png` (새) | 원본·MSW 비교 미리보기 |
| `map/MapImportTest.map` | 배경 RUID를 읽으려고 시험 맵에만 가져오기 + 저장 (마지막 상태 = 100020101 머쉬맘의 오솔길). 다른 맵은 손대지 않음 |

- 이미 있던 행을 재사용한 필드보스 4종(행 내용 안 바꿈): D.로이 `279`, 자동 경비 시스템 `280`, 설산의 마녀 `240`, 리치 `244`.
- **새 모델 파일 없음**: 탑 몬스터는 모두 `TowerRunManager.MonsterModelId = "towerstarpixie"` 모델 하나로 소환하고 `TowerMonster:ApplyLook(stand, attack, hit, die, hitSound, dieSound)` + `MoveRUID` 로 RUID만 바꿔 끼운다. 보스도 같은 방식 → `BossTable.ModelId = towerstarpixie`, `Scale 1.5`.
- 메이커 새로고침(편집 모드) 1회 → 빌드 로그 오류 0 (경고는 기존 스크립트 것만). 플레이로 `GetTable` 읽기 확인은 다른 담당이 메이커를 쓰고 있어 하지 않았다.

## 2. 데이터 형식

### BossTable.csv
```
BossId,Order,MonsterId,Name,MobId,Level,UnlockFloor,PackId,StandRUID,MoveRUID,AttackRUID,SkillRUID,HitRUID,DieRUID,HitSoundRUID,DieSoundRUID,BgRUID,TileSetRUID,#BgSourceMapId,AtkType,Elem,DefP,DefM,Evade,Mech1,Mech2,TipText,ModelId,Scale,#LevelNote
```

| BossId | 이름 | MonsterId | 원본 ID (팩) | Lv | 해금 층 | DefP/DefM/Evade | Mech1 / Mech2 |
|---|---|---|---|---|---|---|---|
| B1 | 머쉬맘 | 686 | 6130101 | 60 | 3 | 0/0/10 | `Summon:20:2:60` |
| B2 | 주니어 발록 | 694 | 8130100 | 80 | 12 | 30/0/10 | `Shield:15:Holy:25:30` |
| B3 | 피아누스 | 698 | 8510000 | 100 (원본 미상 → 29층 몬스터 Lv) | 29 | 0/20/10 | `Interrupt:15:4:5:Fire:40` |
| B4 | 그리프 | 701 | 8180001 | 105 | 38 | 0/0/40 | `Fly:18:6:2` |
| B5 | 마뇽 | 702 | 8180000 (`mob/2600022.img`) | 105 | 39 | 30/30/10 | `Breath:14:3:3:55` / `Enrage:30:0.67` |

- `AtkType` 5종 모두 `P`. `TipText` 는 Spec §5 툴팁 전체(` · ` 로 구분). `Mech` 형식은 Spec §9 그대로.
- `MonsterId` 는 MonsterTable 행이다 → 필드보스와 보스전이 같은 행(그림·소리)을 쓰고, 보스전 수치·속성·방어는 BossTable 값을 우선 쓰면 된다.

### BossStepTable.csv
`Step,HpMul,AtkMul,GoldMul,BookS,BookM,BookL,Cube,ScrollLow,ScrollMid,ScrollHigh,FirstDia,#Note`

| 단계 | HpMul | AtkMul | GoldMul | 경험의 서 소/중/대 | 큐브 | 주문서 하/중/상 | 첫 처치 다이아 |
|---|---|---|---|---|---|---|---|
| 1 | 0.6 | 0.6 | 3 | 3/0/0 | 0 | 1/0/0 | 50 |
| 2 | 1.2 | 0.9 | 4.5 | 0/1/0 | 0 | 2/0/0 | 50 |
| 3 | 2.4 | 1.3 | 6 | 0/2/0 | 1 | 0/1/0 | 100 |
| 4 | 4.8 | 1.8 | 8 | 0/0/1 | 2 | 0/2/0 | 100 |
| 5 | 9.6 | 2.5 | 10 | 0/0/2 | 3 | 0/0/1 | 200 |

### BossConfig.csv
`DailyTickets 3`, `TimeLimit 180`, `GuardReduce 70`, `GuardCooldown 20`, `GroggySec 5`, `GroggyDmg 50`, `TelegraphInterval 12`, `TelegraphWarnSec 2`, `TelegraphMul 2.5`, `ShieldOtherRate 25`, `BossScale 1.5`, `UnlockFieldBossFloor 3`

## 3. 보스·필드보스 리소스 (msw-search, 32종 = 보스전 5 + 필드보스·대체 27)

- **32종 모두 찾음.** 원본 ID 팩이 없던 2종은 같은 몬스터 팩으로 대체: 마뇽 8180000 → `mob/2600022.img`, 아르마 8641011 → `mob/8641010.img`(원본 이름도 "아르마", SSD 그림도 8641010 만 있음). 미리보기로 같은 그림 확인.
- 규칙: `AttackRUID = attack1` (없으면 `skill1`), `HitSoundRUID = _audio/Damage`, `DieSoundRUID = _audio/Die` (기존 행과 같은 규칙). 빈 동작은 코드가 대기 모션으로 대신한다.
  - attack1 없음 → skill1: 마노, 다일
  - move 없음: 세르프, 피아누스, 아르마
  - `_audio/Die` 없음(죽는 소리 없음): 엘리쟈, 레비아탄, 마뇽(2600022)
  - skill1 없음: 주니어 발록, 하늘 수호령, 리버스 다크소울
- 레벨 "미상" 보스는 그 보스가 처음 나오는 층의 대표 몬스터(`FloorTable.MonsterId`) 레벨을 넣었다. 몬스터 능력치는 층 몬스터 레벨(`StageMonsterLevel`)로 계산되므로 표시·로그용이다.

| 원본 ID | 이름 | MonsterTable id | 층 | 팩 | stand | move | attack | hit1 | die1 |
|---|---|---|---|---|---|---|---|---|---|
| 2220000 | 마노 | 685 | 1~2 | `mob/2220000.img` | `e035bb90c053401b88de2159dfa230eb` | `3dcd0dc63d2d491b9b8d39b3b9d0a214` | `c05453dd21fd4ed581d193930ab4c331` (skill1) | `452cb740ddcb4837a46b75d7935e2ffc` | `f430051f6fc34f2eb56fe5e62b346eac` |
| 6130101 | 머쉬맘 | 686 | 3 | `mob/6130101.img` | `86015373d23b4b05bb8fcc6086354652` | `c745828d831b42739b9a58c07f334864` | `90c5456e2a8a45478df02500797a9afd` | `90fee9416184420a9796a679751d1ec5` | `ec90f0b64a814c4980de89f496d360ec` |
| 3220000 | 스텀피 | 687 | 4,14,16,17,18 | `mob/3220000.img` | `0b6c2f6faccc44da980ce832ab636bf6` | `36e33516befa4b2db894d0c7f105b97e` | `3a3034172ea14fe0b5e636604e352306` | `68ca2cd0ecc04b318660b2b3f14e2acd` | `e1b4fc8f0f0f4991b13728ad30c40039` |
| 7110300 | D.로이 | 279 (기존) | 5 | `mob/7110300.img` | `cc7b8c037a7843b0ba5a23845927b1af` | `f2a4cb325642427d9905757a993cf76d` | `64d8111d09ba4f4896352b2b6db7ead0` | `ac3c8b7671c8429287ce897a8ac26a3f` | `18b3e855652449ccb397747e98d59fcc` |
| 3220001 | 데우 | 688 | 6,13,15,22 | `mob/3220001.img` | `6cf930e280f44694a0a0feedbfd855ed` | `196bfc9fe38b4266b279f1b3be53c5dc` | `4a7d9f008b334c1187c087d6f5e5cdd3` | `f86cb206cdbc4954aaa1760ad6e25113` | `7c264f46b77a41ecb52c50b0f191d249` |
| 6220000 | 다일 | 689 | 7 | `mob/6220000.img` | `002f7f9d3abc49928a922e8eb3038cf1` | `99712d6acafb4374adc0b10ff16dcbca` | `75ec4896515b4423b1d3e26259426b32` (skill1) | `794f9746118f4e73bc99f66c68f96286` | `e133539edc3d486eaa17345a4859661d` |
| 5220002 | 파우스트 | 690 | 8 | `mob/5220002.img` | `f7be8b50fd7743eb92b1235ab68ab3f0` | `f0b1b3ee713444e2873711317f5de60d` | `95b69d3223ce4a6c86e45a0557e6d392` | `a7dc481428ae490c95ddd584456259da` | `5cd0f6ea2253494ca18d1bfd1cf62192` |
| 5250004 | 차우 | 691 | 9 | `mob/5250004.img` | `7c33c64898bc4d2a8ae80475885974b3` | `951e0badd1c14c7fb547c58f6741fbe5` | `b61b06dd618e422aad962caa0917b04a` | `b81d6f5d44234ecdbe5cad373fe5f0cb` | `5c1d9e14dfc34f55a54811e0599bf947` |
| 6300005 | 좀비 머쉬맘 | 692 | 10 | `mob/6300005.img` | `a908b6976a9844aea9408a80ec9bdc6b` | `d607a7dc605648fbbba53727bf631739` | `b5e4a508d0e842e1823c2a258d217737` | `452efbf925a34f2291d1ac00b1cd40a6` | `e65ef0b7866f498fb942ae749705fd97` |
| 7220001 | 구미호 | 693 | 11 | `mob/7220001.img` | `959ea412aeac4065bdb2c58e8be26d91` | `730a07e8f4b7415d820a39deb22c3f9e` | `84dcb999264745eabe9fcc1cd895dff0` | `a735d722d25149f7a36325e4085ea03a` | `8f69efaf07514bd7a5fe9c37f3bb5c94` |
| 8130100 | 주니어 발록 | 694 | 12 | `mob/8130100.img` | `4b3ad414c1184c02b5eb459b29ee8ece` | `4f60f3e6658d439aa7aa35f23a155d9d` | `45a54fc4b202417bab825bf820698d7e` | `47ef73f20aa245f9ad2cc12c8f4c11da` | `6489dfbcae58487fb64d9978065b282c` |
| 8220000 | 엘리쟈 | 695 | 19~20 | `mob/8220000.img` | `d458879e2e2949f2bb2e915ff67fbc64` | `e4e1fb8384994d8890b786f7b9df93a3` | `8c933520034f4a3a8fb4163495ab714e` | `5ee4cd91882446e99905bea46f26e8e6` | `1ebdf7b4f5b64af3a385a18551a410fe` |
| 4220000 | 세르프 | 696 | 21,26,27,30,31,32 | `mob/4220000.img` | `2d5ced662b8848dcbba8dd0e12233f7e` | — | `084e6984a902414c95110b0321f62466` | `704897beea32477bbc42493e4327f138` | `005be73fcd974fbb9e20bd999480afb0` |
| 5220003 | 타이머 | 697 | 23~25 | `mob/5220003.img` | `cbdd230e5cd34b7b9eeb1cd206d19f34` | `684685ca8dc84aac920f77e5c5d871f1` | `9a4f5adadc25455e88c0394ee41f346d` | `a8f2cac815354c40a9147389bc070d78` | `cb0d2a03101b4210b142a74cd178de88` |
| 8510000 | 피아누스 | 698 | 28~29 | `mob/8510000.img` | `207c5d6dcee243ad9a141397244ba9a9` | — | `e76534125d994b9c8a1f3bb520c4533f` | `09dd05ae2710401fb33adce7e376a16e` | `d63b783f0ba847fda76f340ccb8995ea` |
| 7090000 | 자동 경비 시스템 | 280 (기존) | 33 | `mob/7090000.img` | `decf2b41a0a64551b44b7bd69b953f4f` | `b272d3d6f61242ad99e9cf5d8010086a` | `8edd6b3d474c44079d995740e215fa53` | `b2eed4f3b4574df5bad6d8af628a103d` | `5e84c1cd39d346deb3201c5b56ac13cf` |
| 7220000 | 태륜 | 699 | 34 | `mob/7220000.img` | `0e1e782401624d7693959777c0b16ebe` | `52eef131b7604e758e7dd92fb803a64e` | `2979b16c037649adaf543912a4127c77` | `bd722884c2c245d7b8955fdc6c5d6ebb` | `3c45042f3e544edeadccb1c66061e440` |
| 6090001 | 설산의 마녀 | 240 (기존) | 35 | `mob/6090001.img` | `4b50438f59154f41aba918b2d28bb8d2` | `8816e805196a4f09aca79f27de12ade8` | `bdd2cc8707254cbeac714bbc260b367e` | `79d70b581e22411abbd0f142c038b7ad` | `ba9ce08382224dd9ad9e3a97428d4cdd` |
| 6090000 | 리치 | 244 (기존) | 36 | `mob/6090000.img` | `8b20897f0de04e76a742460b6cbadb41` | `82965e5a9a6a4a9295a9dcb72c158c10` | `2eb9f0c278734f558d87ac12159e09b5` | `e6bb40a549684e9f9225671eb568835f` | `ef23a1eab16e4c56adcf4938e3378706` |
| 8220001 | 스노우맨 | 700 | 37 | `mob/8220001.img` | `bf93196a2da842eaaf93c944453416d8` | `881ed191bfad4861b63aa94d2a9a2bb8` | `022330aea10449ac970a86ee17db8709` | `4ace032a3dcf4e6a9ee28925c92a93c6` | `23d06abe8f804666beaee7246a586f05` |
| 8180001 | 그리프 | 701 | 38 | `mob/8180001.img` | `3a3d17c98a3944a1ae169d33cca2c99c` | `74191c32e6c240d5a9daee58eac09ecb` | `3bab112c7c0649a3bc29594364f523ff` | `6079cbd6f14d472d8e7e7cf98d94fb9e` | `a0fab07a4b9e4fbca86ae109b76db138` |
| 8180000 | 마뇽 | 702 | 39 | `mob/2600022.img` | `b8cdd7920392456f99272649c7cb7c2d` | `7988e04f93f749e39d199da40324bf5f` | `4aa4b090aa304f898a8f91caeb59d736` | `1631e67423e247b1bae0bd1183769752` | `a0bde46d583747ba96bc550b2fc00ee6` |
| 8220003 | 레비아탄 | 703 | 40 | `mob/8220003.img` | `063d6cb856c54130ac74f81842b3f95c` | `9386a203ce5f4551b0dd9d6b8d97bfcd` | `423dd803460b44b1b77ac0f6e4f91eca` | `ed0dd890d0fa4e96bdffc14e55c43632` | `6412dc6e9529402c8c8ac9ebda64d487` |
| 8220005 | 릴리노흐 | 704 | 41 | `mob/8220005.img` | `463c309d7e7445d1b713d2cae736f655` | `8e7d819e07c24ef598cf40f6b4034fe1` | `a37584f96fea4676b4b917cf85d6ce29` | `0c158245b669433db00221dd0020fe6c` | `af9929bbca594db784754375b5e223c4` |
| 8220004 | 도도 | 705 | 42~48 | `mob/8220004.img` | `2d63f90f9d464f7192713ad9209e9e63` | `a247729732f0464f85bd2842d5c064a3` | `a75f362f66ca44f59d02819b21b814c9` | `71860db25d9f490db35856503a32b77f` | `a1881289fff94a54a817044e59cadd1c` |
| 4300013 | 락 스피릿 | 706 | 49 | `mob/4300013.img` | `9178aab124574eda956911eca2a2d1fc` | `befdea508c8a4aaa881007cfba8e62d4` | `9dbd3a4f72e34b2da87e263ce83439a3` | `ea764c966574459f907480154022a027` | `d77838be493941659371ad466f1eeeb6` |
| 8800400 | 타란튤로스 | 707 | 50~61 | `mob/8800400.img` | `24fc04602f0e44138f683136f8cf6304` | `73d052cfeb4c4705bc3f0e413a8ca851` | `330220276913485eba804183db73e20e` | `ec8ebcd78c454fc7a51e3ed1293cfeaf` | `6ab0b0196a6145c389d0631eb5c6424c` |
| 8620012 | 변형된 스텀피 | 708 | 62 | `mob/8620012.img` | `fe3077e91e5442579b6cfaa6b4c59469` | `5f23e67504a7462fa7c8d9810b9d2652` | `167e61f3b3014c019cb01c32260aa7bb` | `8e7127d18d384991b15febff4f3c8497` | `063d7e0a7933497289da2910e07412a5` |
| 2400259 | 하늘 수호령 | 709 | 63~64 | `mob/2400259.img` | `f68ae919f0eb4076906a358dc0d52dd0` | `c615cd4fa34740d6999135eeebe58737` | `8713795422324af1b68598dc2a88bd56` | `d524ec66735541b68c7582854b16b429` | `ed8d1be8131c45dfb8f737bef9f36a05` |
| 8240067 | 게오르크 | 710 | 65~68 | `mob/8240067.img` | `880cef820134491ab91b76c1aa6a22e1` | `462ea9aeaa774a72b62d27e6acfbdf27` | `1a9f18523bd04e01988e5236149773c5` | `ba1313a060574299bc43d9565308fb53` | `5fce3f898dd54738b71d9b435449ccb7` |
| 8641011 | 아르마 | 711 | 69~87 | `mob/8641010.img` | `622c9f8ba72c4f198f100f99818b2f9c` | — | `4bbc8ea4efb94105a2f6246a4f253280` | `18ce05e7686b463484428d8a59010a38` | `fe6eee07a1a04dbbb1ea991832fd42d3` |
| 8644719 | 리버스 다크소울 | 712 | 88~97 | `mob/8644719.img` | `c0216928f03146fd8564d9b5c20f93ba` | `c05e5653264c488e85b715ae06e034b3` | `07cd2b02a8874e9d9ba95ec26ef93d24` | `bf2057f87d534060b661c1ff96c3fdd0` | `f07f840a5c0b488e9c134932dc391b62` |

(기존 행 279·280·240·244 는 원래 행의 RUID를 그대로 쓴다. 위 표의 값은 팩 조회 결과다.)

## 4. 배경 RUID (보스전 5종 원본 맵)

시험 맵 `MapImportTest` 에 `maker_import_maplestory_map` → `save` → MapBuilder로 `BackgroundComponent.TemplateRUID` 와 타일맵 층별 `TileSetRUID` 를 읽었다. 끝나고 `Lobby` 로 돌아옴.

| 보스 | 원본 맵 | 배경 템플릿 RUID | 타일셋 RUID | 들어간 곳 |
|---|---|---|---|---|
| 머쉬맘 | 100020101 머쉬맘의 오솔길 | `e9693e2f342347bfadab2b24f4fbc001` | `46701ff2021b4d1fb21fbf5790b1ab14` | BossTable B1, FloorTable 3층 |
| 주니어 발록 | 105030500 금지된 제단 | `ab649a3d072948ac975e32a0fccd9c5c` | `60aac899206348fc984f9fbfbf7fdd17` | B2, 12층 |
| 피아누스 | 230040420 피아누스의 동굴 | `f1dadc8053c2497f806905394e3690f8` | `694c9423d9ac4d42b65a68ec0d2f1106` | B3, 28·29층 |
| 그리프 | 240020101 그리프의 숲 | `0cd16a9422ea44bdbd1e082aa6381fbc` | `16bedc23c0a042c2b3b692e0b70b0f5e` | B4, 38층 |
| 마뇽 | 240020401 마뇽의 숲 | `d9f5d0618cb14f2b8d4bd0e9843bf5e6` | `16bedc23c0a042c2b3b692e0b70b0f5e` | B5, 39층 |

- **주의 — 타일셋 정정:** 가져온 맵에는 타일맵 층이 8개 있고, 비어 있는 층은 전부 `9dfea3808bbd49a5877d8624df21b1c7` 이다(기본값). 실제 타일이 깔린 층은 1~2개뿐이라 **타일이 가장 많은 층의 타일셋**을 골랐다. [Map-Import-Test.md](Map-Import-Test.md) 에 예로 적힌 `9dfea380…`(머쉬맘의 오솔길)은 빈 층 값이고, 실제 타일셋은 `46701ff2…` 다(1층 메이플 아일랜드·3층 헤네시스 층 타일셋과 같음).
- 그리프·마뇽 숲은 타일셋이 같다(미나르숲).

### 4.1 필드보스 원본 맵 (FloorTable `BossBgRUID`·`BossTileSetRUID`)

같은 방법(시험 맵 가져오기 → 저장 → MapBuilder 읽기)으로 메이커 목록에 있는 21개 맵을 모두 읽었다. 위 5종과 합쳐 **필드보스 층 38층 중 37층** 채움. 빈칸은 49층(락 스피릿, `103040500` 메이커 목록에 없음) 하나.

| 원본 맵 | 층 | 배경 템플릿 RUID | 타일셋 RUID |
|---|---|---|---|
| 102020500 돌풍지대 (스텀피) | 4 | `4d7cf405dcbd445bbf01467b84aff4bf` | `e80a4b6e22d34348837d2ecf30e7cf74` |
| 103030300 보이지 않는 위험 (다일) | 7 | `ee2c3d69ab0541b98fd77c5c33735528` | `901f885ef94f4a32961bf6cc64e3ec86` |
| 300010420 동굴 깊은 곳 (차우) | 9 | `ab42396e04cc4fdbb602720c7c3b26ff` | (타일 없음) |
| 105010301 개미굴2 (좀비 머쉬맘) | 10 | `ec5936bcd4be4142b56299da5077e052` | `c6c37b62677843ac9a4a832815fdab32` |
| 260010500 메마른 사막 (데우) | 13 | `d982f8bcda104e29ba497710523147cb` | `40e1e636a76f402c8ea18e5ae62f5555` |
| 200010302 검은빛의정원2 (엘리쟈) | 19·20 | `6edad744fb3a4daca5ed43f9b5292152` | `1af79cf0020f467ba3acc65df135c6b5` |
| 220050200 잃어버린시간<2> (타이머) | 23~25 | `2ef8a0ae09314c9ebdbefba284dedce7` | `b16077cf6ddf423491347306cdb646bd` |
| 230020100 바다 해초 탑 (세르프) | 26·27 | `4e74020e4dd24ad9b790ab74180be9c9` | `afc8c630d5b14bfa81f851cb53284ecb` |
| 261020300 연구소 C-1 구역 (자동 경비 시스템) | 33 | `73d655f21bbb4c9693d6cb93a42cad68` | (타일 없음) |
| 250010304 떠돌이곰의 영토 (태륜) | 34 | `23a1044871f4451b9f09d978c59fb6ae` | `8d52f413ee1541c39139245692ed8c09` |
| 211050000 차디찬벌판 (설산의 마녀) | 35 | `1dc9af34148d4286a10fa896cb28fa13` | `df1a1fee05874794a624c2bccbb1574d` |
| 211041400 죽은나무의숲4 (리치) | 36 | `4b7a63d784ef41a8b08f7bfd77c647ef` | `0d7f72a8926349518a2912f9f678d354` |
| 211040500 날카로운절벽3 (스노우맨) | 37 | `187a40079c2844108c2f1a3dfaa323fa` | `df1a1fee05874794a624c2bccbb1574d` |
| 240040401 레비아탄의 협곡 | 40 | `df8286d0dd874340a79458895a2edaac` | (타일 없음) |
| 270020500 후회의 길5 (릴리노흐) | 41 | `aa66e1d65dcf4013b82ded80c54e4016` | (타일 없음) |
| 240093300 암벽 거인의 심장 (타란튤로스) | 50~53 | `4814b371a8134430acb1d7c823b7610c` | (타일 없음) |
| 273020400 칼바람의 묘지 (변형된 스텀피) | 62 | `fb304f3970284021a7d87a3bf032ff0e` | `739f8a7cfbf5427aad66e0814a38081d` |
| 410000121 수호령의 산책로 (하늘 수호령) | 63 | `84bdd7e396e74eeb97c4fa3feedd0c37` | (타일 없음) |
| 350042400 블랙헤븐 갑판7 (게오르크) | 65·66 | `1cb8db806e6f4db5b16a8102c3123acc` | (타일 없음) |
| 450001230 아르마의 은신처 | 69 | `eccd257c48924b0db9c37691c37e4ccd` | (타일 없음) |
| 450011520 고통의 미궁 중심부2 (리버스 다크소울) | 88~90 | `433cf33531294c27a6ed80795526c154` | `46b6d72d164f4e2886ba56d9ee388670` |

- **(타일 없음)** = 가져온 맵의 8개 타일맵 층이 모두 비어 있다(바닥을 타일 대신 오브젝트로 그린 맵으로 보임). `BossTileSetRUID` 빈칸 → B2 구현 때 **배경만 바꾸고 타일셋은 층 것을 유지**하도록 처리할 것. 배경과 바닥 그림이 어울리지 않으면 그 층은 배경도 빼는 것을 검토.
- 같은 보스가 이어지는 대체 층(타란튤로스 54~61층, 아르마 70~87층, 리버스 다크소울 91~97층 등)은 원본 맵이 없어 빈칸 = 층 배경 그대로.
- 시험 맵 `MapImportTest.map` 의 마지막 저장 상태는 `260010500`(메마른 사막)이다.

## 5. 다음 단계(B2~B7) 구현자가 알 점

- **필드보스 연결(B2):** 지금 `TowerRunManager` 의 층 표 읽기는 새 열을 읽지 않는다. `FieldBossId`(MonsterTable id), `BossBgRUID`, `BossTileSetRUID` 를 층 정보에 추가로 읽고, 보스 웨이브 마지막 1마리를 `self._T.monsterById[FieldBossId]` 로 바꿔 기존 `SpawnWaveMonster(info, x)` 에 넘기면 된다. 배경은 기존 층 배경 교체(`ChangeBackgroundByTemplateRUID` + `FloorTileSet`)를 같은 방식으로.
- 한 보스를 여러 층이 쓴다: 아르마 69~87층(19층), 타란튤로스 50~61층, 리버스 다크소울 88~97층, 도도 42~48층, 세르프 21·26·27·30~32층, 스텀피 4·14·16~18층 등. 대체 보스 층은 `#BossBgSourceMapId` 빈칸 = 층 배경 그대로.
- 속성 4종 외 필드보스는 `Elem` 빈칸(약점·반감 없음). 공통 예고 강타·방어 버튼(B3)은 `BossConfig` 의 `Telegraph*`·`Guard*` 키를 쓰면 된다.
- 보스전 수치(B5): "해금 층 층 보스 × HpMul" → `BossTable.UnlockFloor` + `BossStepTable`. 보상 금색 메소는 해금 층 클리어 금색 메소 × `GoldMul`.
- 보스 크기: 필드보스는 기존 `BossScale`, 보스전은 `BossTable.Scale`(1.5).
- **MonsterTable 덮어쓰기 주의:** 작업 중 다른 담당이 데이터 파일 여러 개(MonsterTable·RunConfig·CharJobTable·MonsterTypeTable 등)를 한꺼번에 다시 쓰면서 추가한 28행이 한 번 사라졌다(2026-11-24 01:05 KST). 다시 추가하고 새로고침했다. MonsterTable 을 통째로 다시 쓰는 담당은 **id 685~712 행을 지우지 말 것**(지금 713줄 = 머리줄 + 712행).
- 새 데이터셋 3종은 이번 새로고침으로 등록했다(빌드 로그 오류 0). 플레이 중 `_DataService:GetTable("BossTable")` 읽기는 아직 확인 안 함 → B2/B4 첫 플레이 때 `[VRF]` 로그로 행 수 확인할 것.

## 6. 확인 못 한 것 / 남은 일

- 필드보스 원본 맵 배경은 §4.1 에서 모두 채웠다. 남은 빈칸은 49층(`103040500` 메이커 목록에 없음) 하나, 타일셋 빈칸은 10개 맵(바닥 타일 없음).
- 아르마 원본 ID 8641011 은 SSD 그림 폴더가 없어 8641010 과 비교했다. 아르마 팩 stand 는 빛 덩어리 모양(원본 stand 도 같음) → 전투에서 어색하면 다른 동작으로 바꿀 것.
- 원본 `stand/0` 이 없어 다른 프레임으로 비교: 머쉬맘·그리프 `move/0`, 게오르크 `hit1/0`.
- 피아누스 레벨은 원본 미상이라 29층 몬스터 레벨(100)로 둠.

---

# B2~B7 구현 진행 (2026-11-24, 구현 담당)

- 상태: B2·B3·B4·B5·B7 구현, B6 기믹 5종(+분노) 구현. 메이커 새로고침 빌드 오류(LEA) 0. 커밋 안 함.
- 메인 담당 확정 사항: 보스전 5종, 층마다 필드보스(예고 강타만), 방어 버튼 1개(−70%, 쿨 20초), 단계 1~5, 하루 3회 공용·클리어 때만 차감, 첫 처치 다이아, 보스 탭은 3층 필드보스 첫 처치로 열림.

## 7. 먼저 고친 것 — 화상 dmgCtx 수명

- 원인: 화상 틱(`TickBurn`)이 `_T.dmgCtx = {dot, elemId="Fire"}` 를 세운 채 `FinalDouble → ApplyRolledDamage` 를 불렀고, `CombatMul` 이 **dmgCtx 만 보고** 지속 피해·불 속성을 정했다. 그 사이(처치 연쇄 등)나 오류로 해제가 빠지면 다음 기본 공격(`TowerPlayer.CalcDamage`)이 불·지속(방어 보정 없음)을 물려받았다.
- 수정: 지속 여부·지속 속성은 **타격 주인(`hitSrc`)에만** 싣는다. `TickBurn` 은 `SetHitSrc(nil,"",true)` + `hitSrc.elemId="Fire"` → 처리 → `ClearHitSrc(prev)`. `CombatMul` 은 dmgCtx 를 배율에 쓰지 않는다(유물 피해 종류 `RelicDamageBonus` 에만 남음). 기본 공격은 새 `SwapDmgCtx(nil)` 로 dmgCtx 를 비운 채 계산하고 되돌린다. 각인 화상 판정(`TryTalentBurn`)도 `hitSrc.dot` 을 함께 본다.

## 8. 바꾼 / 만든 파일

| 파일 | 내용 |
|---|---|
| `RootDesk/MyDesk/Boss/PlayerBoss.mlua` (새) | 저장 키 `OrbisBossV1` JSON `{v, fk, best, first, day, used}` (모두 문자열·정수). 필드보스 첫 처치 층(`FieldKills`), 보스별 최고 단계(`BossBest`), 첫 처치 다이아 비트(`BossFirst`), 오늘 쓴 도전권(UTC+9 날짜). `CheckEntry`, `TakePending`, `CalcReward`, `Settle`(먼저 저장 → 성공 시 지급, 오프라인 보상과 같은 순서), `MarkFieldKill`. 디버그(메이커 전용): `DebugUnlockAll`, `DebugSetUsed` — 메모리만 바꾸고 그 뒤 이 접속 동안 저장을 막는다 |
| `Global/DefaultPlayer.model` | ModelBuilder 로 `script.PlayerBoss` 추가 |
| `RootDesk/MyDesk/Tower/BossMechanics.mlua` (새) | OrbisTower 맵 루트에 붙는 기믹 컴포넌트. 공통 예고 강타·방어 버튼(`RequestGuard`)·제한 시간·그로기 + Summon/Shield/Interrupt/Fly/Breath/Enrage. HUD 표시 값 @Sync (0.1초마다 바뀐 것만) |
| `map/OrbisTower.map` | MapBuilder `upsertComponent` 로 루트에 `script.BossMechanics` |
| `Tower/TowerRunManager.mlua` | B2 층 표 새 열·`PickWaveInfo`·보스 배경, B5 `RunMode`/`SetupRunMode`/`OnBossFront`/`BossModeClear`/`DebugStartBoss`, B6 연결(`CombatMul` 보스 배율 곱 하나, `ApplyMonsterDamage` 보호막 흡수·저지 게이지, 소환물 `BossSummon`·`KillQueued` 분기, 날기 중 `PullIfUnreachable` 생략, 해일 실패 `ActorInterval` 둔화), dmgCtx 수정 |
| `Tower/TowerPlayer.mlua` | 기본 공격 `CalcDamage` 가 dmgCtx 를 물려받지 않게 (`SwapDmgCtx`) |
| `Lobby/LobbyManager.mlua` | `RequestBossPlay(bossId, step)`: `PlayerBoss.CheckEntry` → `PendingBoss="B1:1"` → 탑 이동 (에너지 안 씀) |
| `Tower/BattleHUDLogic.mlua` | 보스 HUD(`UpdateBossHud`), 방어 버튼 클릭·툴팁. **스킬 아이콘 표 순서 수정**(메인 담당 요청): JobSkillTable 먼저(새 아이콘 우선), HeroSkillTable 은 없는 Id 만 |
| `ui/BattleHUD.ui` (UIBuilder) | `BossBar/Shield`(흰 게이지, 체력바 아래), `BossBar/Groggy`, `BossBar/Timer`, `BossBar/Weak0·1`(속성색 점, 툴팁), `BossBar/Aura`(졸개 보호), `BossWarn`(예고 띠 + 파란 저지 게이지), `GuardBtn`(128×128, button 4상태 SpriteSwap, 쿨타임 원형 채움, 예고 중 파란 테두리) |
| `Tower/ResultPopupLogic.mlua` | 보스전 결과: 제목 "머쉬맘 1단계 처치!/실패", 별·미션·구간 상자 숨김, 경험의 서 보상 칸, 다시 도전 = 도전권 검사(에너지 아님, 버튼 아래 "n/3"), 지역 선택으로 = 보스 탭으로 |
| `Lobby/BossSelectLogic.mlua` (새) | 보스 탭 화면 전체 (탭 전환, 목록, 그림, 속성·공략 아이콘, 단계, 보상, 도전, 도전권) |
| `Lobby/RegionSelectLogic.mlua` | 최소 수정: `PendingBossTab` 속성, `Open` 에서 `_BossSelectLogic:OnRegionOpen` 호출 |
| `ui/LobbyUI.ui` (UIBuilder, `Root/RegionSelect` 아래만) | `TabRegion`/`TabBoss`(+`Lock`)/`TabLine`, `BossView`(BossRow0~4, Pano, BossArt, Elem0~2, Mech0~1, Step0~4, Reward0~5, Challenge, TicketPill). `regionPlaque` 숨김. `Root/BtnOffline` 은 건드리지 않음 |

건드리지 않음: `PlayerEngrave.mlua`, `PlayerOffline`, `OfflineUILogic`, `OfflineUI`, `.codeblock`, 데이터 CSV(읽기만).

## 9. 단계별 구현 내용

### B2 층별 필드보스
- `FloorTable.FieldBossId` 로 보스 스테이지(`StageKind=="boss"`) 마지막 웨이브 마지막 1마리를 고른다(`PickWaveInfo`, 대기열 채우기·바로 소환 둘 다). 수치는 기존 층 보스(`BossHpMul`·`BossAtkMul`·`BossDefMul`) 그대로, 속성은 MonsterTable `Elem`.
- 배경: 보스 스테이지 동안 `BossBgRUID` 로 `ChangeBackgroundByTemplateRUID`, 다음 층에서 층 배경으로 자동 복귀(`ApplyFloorLook` 이 소환마다 판정). **타일셋은 `BossTileSetRUID` 가 있을 때만** 바꾸고 빈칸이면 층 타일셋 유지(지시대로). 바닥 타일이 없는 원본 맵 10개 층은 배경만 바뀌므로 바닥과 어색할 수 있다 — 화면으로 확인하지 못해 배경을 빼지는 않았다(테스트 담당 확인 항목).
- 필드보스를 처치하면 `PlayerBoss:MarkFieldKill(층)` (디버그 판은 기록 안 함) → 3층이면 보스 탭 열림, 각 보스 해금 층이면 그 보스 열림.

### B3 공통 예고 강타·방어 버튼·HUD
- 필드보스·보스전 공통: `TelegraphInterval`(12초)마다 `TelegraphWarnSec`(2초) 예고(보스 붉은 틴트 + HUD 예고 띠) → 살아 있는 맨 앞 구성원에게 보스 공격 × `TelegraphMul`(2.5, 방어력 적용, 빗나감 없음).
- 방어 버튼: 예고 중 누르면 그 패턴 피해 −`GuardReduce`%(70), 쿨타임 `GuardCooldown`(20초). 예고가 없을 때 누르면 아무 일 없음(쿨타임 안 돎). 쿨타임 중에는 `Enable=false`(Disabled 이미지 + 입력 차단). 장벽 폭발·해일·브레스에도 적용.
- HUD 위치: 기존 `BossBar`(위 가운데)를 그대로 쓰고 그 안/아래에 추가(스펙 좌표 대신 기존 배치 유지). 방어 버튼은 스펙의 (760,-380)이 구성원 패널(P3)과 겹쳐 그 위 (760, 아래 기준 250)로 옮겼다 — 금지 영역(오른쪽 위) 밖. 보스전은 체력 숫자 대신 남은 시간.

### B4 저장·입장
- 보스 탭 [도전] → `LobbyManager.RequestBossPlay` → `PlayerBoss.CheckEntry`(해금·단계·도전권) → `PendingBoss` → 탑. 탑 `StartRun` 이 `SetupRunMode` 에서 꺼낸다(다시 도전 > 디버그 > 로비 예약 순).
- 도전권은 클리어 정산(`Settle`) 때만 1장 차감. 저장 실패면 차감·기록을 되돌리고 지급하지 않는다.

### B5 보스 모드 판
- `RunMode="boss"`: 해금 층·마지막 스테이지·마지막 웨이브, 웨이브 1마리, 판 시작 돌파 없음, 처치 보상(깃털·주문서·열쇠)·포탈·상점 없음, 계정 경험치 없음. 스킬은 판 시작 칸 스킬(레벨 1) 그대로.
- 보스 수치: 그 층 보스(필드보스 계산) × `BossStepTable.HpMul/AtkMul`, 방어 = `BossTable.DefP/DefM × BossDefMul`, 회피 = `BossTable.Evade`, 속성 = `BossTable.Elem`, 크기 = `BossTable.Scale`, 배경·타일셋 = `BossTable.BgRUID/TileSetRUID`, 제한 시간 `TimeLimit`(180초, 넘으면 실패).
- 클리어 → `BossModeClear` → `PlayerBoss.Settle` (금색 메소 = 해금 층 보스 스테이지 금색 메소 × GoldMul, 경험의 서, 레드 큐브(계정 Lv25 전이면 같은 수의 중급 주문서), 주문서, 첫 처치 다이아) → 결과 팝업.

### B6 보스 기믹 (BossTable Mech1/Mech2)
| 보스 | 기믹 | 구현 |
|---|---|---|
| 머쉬맘 | Summon:20:2:60 | 10초 뒤부터 20초마다 주황버섯(MonsterTable 8) 2마리를 몬스터 2·3칸에. 소환물은 공격하지 않고, 살아 있으면 보스 받는 피해 ×0.4 (HUD 초록 방패). 처치해도 웨이브 수·보상 없음 |
| 주니어 발록 | Shield:15:Holy:25:30 | 시작·체력 50%에 게이지(최대 체력 15%). 성 100%, 다른 속성 `ShieldOtherRate` 25%만 깎고 게이지가 있는 동안 체력 피해 0. 깨지면 그로기 5초(공격 멈춤, 받는 피해 +50%). 25초 안에 못 깨면 3초 예고 후 장벽 폭발(파티 최대 체력 30%) + 게이지 재생성 |
| 피아누스 | Interrupt:15:4:5:Fire:40 | 15초마다 4초 예고 + 저지 게이지(최대 체력 5%, 불 2배). 채우면 취소 + 그로기 3초, 실패하면 파티 최대 체력 40% + 3초 공격 간격 ÷0.7 |
| 그리프 | Fly:18:6:2 | 18초마다 6초 동안 몬스터 2칸으로 이동 → 사거리 1칸 직업은 기존 `CanReach` 로 못 침, 날기 중 1칸으로 당겨 오지 않음. 회피 40 은 BossTable |
| 마뇽 | Breath:14:3:3:55 / Enrage:30:0.67 | 14초마다 3초 예고 후 3번 칸(살아 있는 맨 뒤) 최대 체력 55%. 체력 30% 이하에서 공격 간격 ×0.67 (한 번) |

- 그로기·졸개 감소는 `CombatMul` 안의 **곱 하나**(`mech:DamageMul()`), 맨 앞 보스에게만. 보호막은 배율이 아니라 흡수(`Absorb`)라 `ApplyMonsterDamage` 에서 처리.
- 날아오르기는 칸 이동만 있고 공중 높이·연출은 없다.

### B7 보스 탭 UI
- 목록 패널 위쪽 안에 [지역] [보스] 탭(160×52, tab 계열, 선택 = 파랑, 아래 파란 연결선). 보스 탭 잠김 = Disabled 이미지 + 입력 차단 + 자물쇠, 누르면 툴팁 "3층 보스를 처음 쓰러뜨리면 열립니다."
- 보스 탭: 지역 화면 요소를 끄고 `BossView` 를 켠다. 목록 5행(그림·이름·"최고/5"·자물쇠), 파노라마(해금 층 `PanoRUID`) + 보스 그림, 속성 아이콘(약점 속성색 ▲ / 반감 흐린 색 ▼ / 무효 회색 ×, 툴팁), 공략 아이콘 1~2개(툴팁), 단계 1~5(선택 파랑·잠김 Disabled·클리어 체크), 보상 아이콘(+수량, 첫 처치 다이아는 파란 테두리), [도전](초록 action 4상태), "오늘 3/3 · 22:03:43".
- 탭·행·단계 버튼은 지역 목록처럼 상태 이미지를 직접 칠한다(Normal/Hover/Pressed/Selected/Selected Hover/Pressed/Disabled).

## 10. 임시 리소스 (새 원본 리소스를 고르지 않음 — 이미 MSW 에 있는 RUID 재사용)

| 쓰는 곳 | 지금 RUID | 바꿀 것 |
|---|---|---|
| 방어 버튼·졸개 보호 아이콘 | `c71f28b127ec4a18975abd3e594bbb7a` (홀리 매직쉘 스킬 아이콘) | 방패 전용 아이콘 |
| 공략 아이콘(소환·해일·날기·브레스·분노) | 그 보스/주황버섯 동작 클립의 `thumbnail://` | 기믹 전용 아이콘 |
| 도전권 아이콘 | `3d5c35546d4944598ed6e1be129ff12b` (에너지 아이콘) | 도전권 아이콘 |
| 예고 표시 | 보스 붉은 틴트 + HUD "!" | 머리 위 느낌표·바닥 띠 이펙트 |

새로 고른 원본 리소스가 없어 원본·MSW 비교 미리보기(`output/preview/`)는 만들지 않았다.


## 11. 구현 담당이 직접 확인한 것 (메이커 플레이 1회, 디버그 경로만 — 저장 데이터 변경 없음)

- Refresh 후 빌드 로그 `LEA` 0, 실행 중 `LEA-`·`attempt to` 로그 0.
- 로비: `PlayerBoss` 불러오기 `boss loaded new tickets=3`, 해금 전 `CheckEntry(B1,1)=locked` → `DebugUnlockAll` 뒤 `ok`, `B1:2=step`(1단계 미클리어). 보스 탭 열기: `BossView` 켜짐·지역 목록/입장 꺼짐, 1행 "머쉬맘", 도전권 줄 "오늘 3/3 · 22:03:43", [도전] 활성, 주니어 발록 선택 시 속성 아이콘 2개(성 약점·독 무효), 2단계 잠김. 지역 탭으로 돌아오면 원래대로.
- 보스전(`DebugStartBoss`): 머쉬맘 1단계 `field boss id=686 ... mode=boss` → 처치 → `boss settle debug (no save) B1:1:435:3:0:0:0:1:0:0:50` → 결과 화면(보상 4칸).
- 기믹(보스 체력을 테스트용으로 300배): 머쉬맘 소환 `summon n=2`·`aura=true mul=0.40`·소환물 처치 `alive=false`; 예고 강타 + 클라이언트 방어 버튼(`SendEvent(ButtonClickEvent)`) `guard used kind=smash` → `dmg=21 guarded=true cut=0.30` (방어 없을 때 같은 보스 `dmg=298~386`); 주니어 발록 보호막 `gauge=630 elem=Holy`, 무속성 타격 `rate=0.25`, HUD 비율 0.90→0.40; 그리프 `fly start cell=2 sec=6`; 피아누스 해일 실패 `kind=wave pct=40 gauge=2054/139500`; 마뇽 분노 `enrage hp=0.25 intervalMul=0.67`, 브레스 예고 시작 `kind=breath`.
- 필드보스(3층 보스 스테이지 웨이브 1마리로 바꿔 시험): `field boss id=686 name=머쉬맘 floor=3 mode=normal`, 이름 "보스 머쉬맘", 기믹 `mode=false`(예고 강타만), 보스 타일셋 적용, 처치 `field boss killed floor=3 debugRun=true` (디버그 판이라 첫 처치 기록 안 함).

### 못 본 것
- 화면(스크린샷) 확인 전부: 보스 탭 배치·글자 넘침, HUD 보호막·예고 띠·방어 버튼 모습, 결과 화면 보스 문구, 보스 배경과 층 바닥의 어울림.
- 실제 저장 경로: 일반 판 필드보스 첫 처치 → `MarkFieldKill` 저장, 보스전 실제 정산(도전권 차감·첫 처치 다이아·보상 지급) → 재접속 복원. (저장 데이터를 바꾸지 않으려고 디버그 판만 돌렸다.)
- 로비 [도전] → `RequestBossPlay` → 탑 이동 → `SetupRunMode` 의 예약 경로(디버그 경로로만 시작함), 보스전 다시 도전, 제한 시간 초과 실패, 장벽 폭발·보호막 파괴 그로기, 해일 저지 성공, 브레스 피해 적용 순간.

## 12. 테스트 담당 체크리스트 (짧게, 밸런스 측정 금지)

- [ ] 빌드 오류 0, 실행 중 `LEA-` 0
- [ ] 로비 지역 선택: [지역]/[보스] 탭. 3층 필드보스 처치 전 보스 탭 Disabled + 자물쇠 + 누르면 툴팁(입력 차단)
- [ ] 서버 `p:GetComponent("script.PlayerBoss"):DebugUnlockAll()` 뒤 보스 탭: 5행, 속성·공략 아이콘 툴팁, 단계 선택(파랑)·잠김, 보상 아이콘·첫 처치 다이아 테두리, 도전권 줄, 긴 이름·오른쪽 위 금지 영역 겹침 없음
- [ ] 탑에서 서버 `mgr:DebugGod(true) mgr:DebugStartBoss("B1",1)`(B1~B5): HUD 보스 이름·체력·남은 시간, 약점 점, 예고 띠, 방어 버튼(예고 중 테두리, 누르면 쿨타임 원형·Disabled), 보스 붉은 틴트
- [ ] 보스전 결과: 제목 "머쉬맘 1단계 처치!", 별·미션·구간 상자 숨김, 보상(금색 메소·경험의 서·주문서·다이아), 다시 도전 버튼 "3/3"
- [ ] 일반 판 각 층 마지막 보스 = 필드보스(로그 `field boss id=`), 보스 스테이지 배경 교체·다음 층 복귀 (특히 타일 없는 층 4·9·33·40·41·50·63·65·69 류는 바닥과 어울리는지)
- [ ] 실제 계정(디버그 아님) 3층 필드보스 첫 처치 → 로그 `boss field kill floor=3 ... save=0` → 로비 보스 탭 열림 → 재접속 후 유지
- [ ] 실제 보스전 1회 클리어 → 도전권 2/3, 첫 처치 다이아 지급(같은 단계 두 번째는 없음), 실패는 차감 없음
- [ ] 회귀: 화상 지속 피해 뒤 기본 공격이 불 속성으로 뜨지 않는지(`[VRF] elem hit elem=Fire` 가 화상 외 기본 공격에 안 나옴), HUD 스킬 아이콘이 새 아이콘(H08·N06·A06 등)

## 13. 남은 문제

- (해결 §14) 보스전 1단계가 너무 약했다 → 체력 배율 조정.
- 날아오르기 공중 연출·예고 느낌표/바닥 띠 이펙트·전용 아이콘 없음(§10 임시).
- 소환물은 공격하지 않는다(보스 공격력으로 때리는 문제를 피하려고). 졸개 공격이 필요하면 별도 공격력 계산을 추가해야 한다.
- (해결 §14) 보스전 칸 스킬은 최대 레벨로 시작.
- 방어 버튼 위치를 스펙 좌표에서 옮겼다(구성원 패널과 겹침, §9 B3).


## 14. 조정 (메인 담당 결정, 2026-11-24) — 보스전 체력·스킬 레벨

### 14.1 보스전 칸 스킬 최대 레벨
- `TowerRunManager.BossMaxSkills()`: 보스전 `PickJob` 에서 구성원마다 장착 칸 스킬을 최대 레벨까지 자동 성장시킨다. 별 옵션은 기존 자동 성장(`RollCardOption`)과 같은 무작위이고, **돌파 레벨은 돌파 없이 레벨만** 올린다. 판 시작 돌파·처치 보상·포탈·상점은 계속 없다.
- 확인: 3인 편성에서 `[VRF] boss max skills levels+40`.

### 14.2 체력 배율 근거 (측정 1회 + 층 비례 가정)
- 측정: 이 계정(최고 클리어 20층 = "20층을 막 깬 파티")으로 보스전(칸 스킬 최대 레벨, 판 안 성장 없음) 머쉬맘 체력을 크게 올려 `DebugDpsProbe(10)` 1회 → **파티 피해 820/초** (`dealt=8204`, 10초, 방어 0 보스).
- 같은 계정 기준 20층 필드보스(엘리쟈) 체력 = **11,000** (기존 층 보스 공식 `BossHpMul 20`). → "그 층을 막 깬 파티"는 그 층 보스를 보스전 조건에서 약 11,000/820 ≈ **13.4초**에 잡는다.
- 가정: 층을 막 깬 파티의 화력은 그 층 보스 체력과 같은 비율로 커진다(층 보스 체력 = 층 몬스터 레벨 공식 × 20으로 층마다 일정하게 커지고, 게임이 층마다 비슷한 처치 시간을 목표로 함). 그러면 해금 층 f 의 1단계 시간 ≈ 13.4초 × HpMul.
- 1단계 60초 → HpMul = 60 / 13.4 ≈ **4.5** (기믹으로 화력이 줄어드는 몫은 여유로 둠). 보스전 기본 체력 = 해금 층 보스 체력(머쉬맘 2,500 / 주니어 발록 7,000 / 그리프 20,000 …) × HpMul.
- 단계: 체력 ×약 1.45씩, 공격도 단계마다 오른다. 같은 화력이면 2단계 약 87초, 3단계 약 127초, 4단계 약 187초(제한 180초 → 장비·각인 성장이 필요), 5단계 약 270초(화력 1.5배 이상 필요).

| 단계 | HpMul (전) → (후) | AtkMul (전) → (후) | 같은 화력 예상 시간 |
|---|---|---|---|
| 1 | 0.6 → **4.5** | 0.6 → **0.8** | 약 60초 |
| 2 | 1.2 → **6.5** | 0.9 → **1.1** | 약 87초 |
| 3 | 2.4 → **9.5** | 1.3 → **1.5** | 약 127초 |
| 4 | 4.8 → **14** | 1.8 → **2.0** | 약 187초 |
| 5 | 9.6 → **20** | 2.5 → **2.6** | 약 268초 |

- 바꾼 곳: `Data/BossStepTable.csv` 의 `HpMul`·`AtkMul`·1단계 `#Note` 만 (다른 열·행 그대로). 코드는 그대로 `그 층 보스 수치 × 배율`.
- 한계: 측정 1회, 기준 파티 1개(20층 계정), 방어 0 보스 기준. 실제로는 직업 편성·속성 상성·기믹 대응에 따라 크게 달라진다 → 사용자 플레이 후 조정.

### 14.3 조정 후 확인 (메이커 플레이 1회, 디버그 판·무적)
- 머쉬맘 1단계: 체력 11,250 (2,500 × 4.5), 20층 계정 파티로 **16.4초** 처치 → 3층을 막 깬 파티 화력(820 × 2,500/11,000 ≈ 186/초)으로 환산하면 약 60초. 그 16초 안에 소환(`summon n=2`, 10초)·예고 강타(`kind=smash`, 12초) 각 1회.
- 주니어 발록 1단계: 체력 31,500, 성 속성 없는 파티(히어로·보우마스터·바이퍼 계열)로 약 50초 동안 **보호막을 한 번도 못 깼다** — 25초마다 장벽 폭발(`kind=blast pct=30`) 2회 후 게이지가 다시 가득 차 체력이 줄지 않음. 스펙("못 깨면 게이지 다시 가득") 그대로라 **성 속성 없이는 클리어 불가**다. 의도(비숍 핵심)가 맞는지 사용자 확인 필요. 완화안: 폭발 후 게이지를 절반만 채우기, 또는 다른 속성 비율 25% → 40%.

### 14.4 주니어 발록 완화 (메인 담당 결정)
- `BossConfig.ShieldOtherRate` 25 → **40** (성 외 속성이 보호막을 깎는 비율), 새 키 `ShieldRefillPct` **50** (장벽 폭발 뒤 게이지를 최대치의 50%로 다시 채움). 코드: `BossMechanics.MakeShieldPct(pct)`, 폭발 처리에서 `ShieldRefillPct` 사용.
- 확인(무적 디버그 판 1회, 성 없는 파티, 주니어 발록 1단계 체력 31,500): 게이지 4,725 → 25초에 장벽 폭발 1회 → 게이지 2,362(절반) → 시작 약 **34초 뒤 보호막 파괴·그로기 5초** (`boss shield broken -> groggy 5`). 180초 안에 최소 1회 파괴 조건 충족. 끝까지 처치 시간은 재지 않았다.


## 15. 통합 테스트 화면 수정 (2026-11-24)

| # | 문제 | 수정 |
|---|---|---|
| 1 | 보스 탭에서 아래 구간 보상 줄이 꺼져 로비 메뉴 버튼이 비침 | `BossSelectLogic.RegionOnly` 에서 `sectionRewards`(아래 줄 배경) 제외 → 보스 탭에서도 가림막 유지. 로비 버튼 입력은 원래 `RegionSelect/Dim`(RaycastTarget=true, 화면 전체)이 막는다 |
| 2 | HUD 1번 이름 "1 크루세이더 Lv.13" 두 줄 | `Members/P1~P3/Job` 글자 자동 축소(BestFit, 14~22) |
| 3 | 보스 탭 첫 열기 빈 칸 / 선택 행 | 탭을 열 때 바로 그린 뒤 다음 틱에 한 번 더 그림, 열 때마다 선택을 다시 정함(열린 보스 중 가장 뒤, 그 보스의 열린 최고 단계) |
| 4 | 약점 점이 바 밖 왼쪽, 배경에 묻힘 | 보스 바 안쪽 이름 왼쪽(어두운 바 위)으로 이동, 이름은 오른쪽으로 |
| 5 | 보호막 선이 얇음 | 12px, 보스 바를 누르면 보호막 툴팁 |
| 6 | 임시 아이콘 | 원본에서 골라 업로드(§15.1), 미리보기 `output/preview/boss_icons.png` |
| 7 | 결과 화면 빈 공간·첫 보상 칸 잘림 | 보스전이면 별 자리에 보스 그림(`Result2/BossArt`), 보상 묶음 100 위로. 보상 칸 Loot0~7 전체 +16(마스크 왼쪽 잘림, 일반 결과에도 적용). 다시 도전 비용 아이콘 = 도전권 |
| 8 | 긴 몬스터 이름이 같아 보임 | 지역 선택 `monsterList/Mon0~17/Name` 글자 자동 축소(BestFit, 최소 12) |
| 9 | 탭 전환 후 툴팁 남음 | 탭 전환 때 닫고 다음 틱에 한 번 더 닫음 |
| 10 | 피아누스가 너무 큼 | `BossTable` B3 `Scale` 1.5 → 0.7 |

### 15.1 원본에서 고른 아이콘 (SSD `_Resources`, MSW 업로드)

| 쓰는 곳 | 원본 | MSW RUID |
|---|---|---|
| 방어 버튼·졸개 보호 | `Character/Shield/_Canvas/01092017.img/info/icon.png` (골드 에인션트 실드) | `545b80cc9b144ebcb9565a56dc888b58` |
| 도전권 줄·다시 도전 비용 | `Item/Etc/_Canvas/0400.img/04001431/info/icon.png` (골든 티켓) | `7e932b7340f74fba8372c5b217f93000` |
| 머쉬맘 소환 | `Item/Etc/_Canvas/0400.img/04006001/info/icon.png` (소환의 돌) | `5dd6b605d6a1452988313b93badc28cb` |
| 마뇽 브레스 | `Skill/_Canvas/2217.img/skill/22171063/icon.png` (드래곤 브레스) | `ab2bdcbf2bd847918c1ef7a68bdc4459` |
| 마뇽 분노 | `Skill/_Canvas/112.img/skill/1120010/icon.png` (인레이지) | `b7d9fffd7efb4c80958dae821ef4e196` |

- 파워 가드(1201007) 아이콘은 SSD 에 그림이 없어 방패 장비 아이콘을 썼다. 원본을 고치지 않아 `수정한리소스` 기록은 없다.
- 확인 스크린샷: `output/verification/fix_01_bosstab.png`, `fix_02_B3_hud.png`(약점 점·이름 한 줄·피아누스 크기·방패 아이콘), `fix_03_B2_shield.png`, `fix_04_result.png`.
