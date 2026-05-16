# Chapter 4 — Sample Boss: Soji Abi (소지아비)

This chapter is a complete, end-to-end walkthrough of building a custom boss encounter from scratch. Every design decision, file, error, and fix is documented exactly as it happened during development. Use it as a living reference when building your own encounter.

> **Running example:** The boss we are building is **Soji Abi (소지아비)** — a breath-and-afterimage swordsman encounter with 15 skills, a 9-coin area barrage, and a full Korean locale.

---

## Table of Contents

1. [Concept and Mechanic Design](#1-concept-and-mechanic-design)
2. [Mod Folder Structure](#2-mod-folder-structure)
3. [Registering the Encounter](#3-registering-the-encounter)
4. [Building the Boss Unit](#4-building-the-boss-unit)
5. [Designing 15 Skills](#5-designing-15-skills)
6. [Writing Locale Files](#6-writing-locale-files)
7. [Tuning and Buffing Skills](#7-tuning-and-buffing-skills)
8. [Special Mechanic — VibrationExplosion on Every Hit](#8-special-mechanic--vibrationexplosion-on-every-hit)
9. [Common Pitfalls and Debugging](#9-common-pitfalls-and-debugging)
10. [Final File Checklist](#10-final-file-checklist)

---

## 1. Concept and Mechanic Design

Before touching any JSON, decide what the boss *does*. A well-defined concept prevents scope creep and keeps the skill set coherent.

### 1.1 Boss Identity

| Field | Value |
|---|---|
| Name | 소지아비 (Soji Abi) |
| Unit ID | `6820001` |
| Role | Aggressive sword fighter |
| Appearance | Existing LeiHeng rig |
| Sin Attribute | CRIMSON (boss-exclusive) |

### 1.2 Core Mechanics

Two custom keywords drive the entire kit:

| Keyword | Type | Description |
|---|---|---|
| `Breath` (호흡) | Stacking buff on Self | The boss's power resource. Accumulates from every skill, empowers later skills. |
| `PhantomIncision` (잔상) | Stacking debuff on Target | Applied by Afterimage skills. Interacts with follow-up skills. |

A third keyword is used on the climactic skill:

| Keyword | Type | Description |
|---|---|---|
| `Vibration` (진동) | Stacking debuff on Target | Applied every hit of the final 9-coin barrage. Triggers `VibrationExplosion` on each coin. |

### 1.3 Skill Budget

15 skills total, organized by role:

| Role | Skills | IDs |
|---|---|---|
| Breath builders (basic) | 연격, 이연잔 | 682001010, 682001020 |
| Afterimage openers | 잔상베기, 잔상 해방 | 682001030, 682001060 |
| Heavy/special attacks | 폭압격, 사냥, 처형, 공진 | 682001040, 682001080, 682001100, 682001110 |
| Combo chains | 이연속참, 필연쇄 | 682001130, 682001140 |
| Debuff/utility | 호흡 강탈, 정신 압박, 취약 유도, 취약 강타 | 682001160–682001190 |
| Climax (9-coin barrage) | 광역 난사 | 682001200 |

> **ID range:** The boss unit lives at `6820001` and skills at `682001010–682001200`. This keeps everything in the `682XXXXX` block, far from base-game IDs and other mods. See the [Appendix of Chapter 2](chapter2_EN.md#appendix-recommended-id-scheme) for safe ID ranges.

---

## 2. Mod Folder Structure

The finished mod sits in a single self-contained folder:

```
mods/
└── 소지아비로/
    ├── custom_encounters/
    │   └── encounter.json
    ├── custom_limbus_data/
    │   ├── abnormality-unit/
    │   │   └── soji_unit.json
    │   └── skill/
    │       └── soji_skills.json
    └── custom_limbus_locale/
        └── KR/
            ├── enemyList/
            │   └── soji_locale.json
            └── skillList/
                └── soji_skills_locale.json
```

There is no `modular_lua/` here. Soji Abi uses only JSON ability scripts — all coin effects are handled by built-in `abilityScriptList` entries. Lua is only needed when you require conditional logic that JSON cannot express (see [Chapter 3](chapter3_EN.md)).

---

## 3. Registering the Encounter

### 3.1 encounter.json

This file tells Lethe which unit to spawn and where the stage appears in the menu.

```json
{
  "list": [
    {
      "id": 68200,
      "encounterType": "ABNORMALITY",
      "stageEntrance": "Kiosk_NormalBattle",
      "rewardType": "NormalBattle",
      "enemyUnitIDList": [6820001],
      "waveList": [
        {
          "waveID": 0,
          "enemyList": [
            { "unitID": 6820001, "slotNum": 2 }
          ]
        }
      ]
    }
  ]
}
```

| Field | Notes |
|---|---|
| `id` | Unique encounter ID — `68200` fits our `682XXXXX` block |
| `encounterType` | `"ABNORMALITY"` — single boss fight |
| `stageEntrance` | Use `"Kiosk_NormalBattle"` for most sandbox fights |
| `enemyUnitIDList` | Must list every unit ID that appears in any wave |
| `slotNum` | `2` places the boss at the center of a 5-slot field |

### 3.2 subchapterui.json

Controls how the encounter appears in the menu UI.

```json
{
  "list": [
    {
      "id": 68200,
      "chapter": 682,
      "displayOrder": 1,
      "nameID": 68200,
      "descID": 68200
    }
  ]
}
```

---

## 4. Building the Boss Unit

The boss unit lives in `custom_limbus_data/abnormality-unit/soji_unit.json`.

### 4.1 Full Unit JSON

```json
{
  "list": [
    {
      "id": 6820001,
      "appearance": "1145_Thumb_LeiHengAppearance",
      "unitScriptID": "1145",
      "uniqueAttribute": "CRIMSON",
      "unitKeywordList": ["LITTLE_FINGER", "SOJI"],
      "hp": {
        "defaultStat": 600,
        "incrementByLevel": 0
      },
      "defCorrection": 5,
      "minSpeedList": [4, 4, 4],
      "maxSpeedList": [9, 9, 9],
      "startActionSlotNum": 3,
      "resistInfo": {
        "atkResistList": [
          { "type": "SLASH",     "value": 0.75 },
          { "type": "PENETRATE", "value": 1.0  },
          { "type": "HIT",       "value": 1.5  }
        ]
      },
      "abnormalityPartList": [
        {
          "partID": 68200010,
          "hp": { "defaultStat": 600, "incrementByLevel": 0 },
          "resistInfo": {
            "atkResistList": [
              { "type": "SLASH",     "value": 0.75 },
              { "type": "PENETRATE", "value": 1.0  },
              { "type": "HIT",       "value": 1.5  }
            ]
          }
        }
      ],
      "patternList": [
        { "actionSlotNum": 3, "skillIDList": [682001010, 682001020, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001020, 682001040, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001060, 682001080, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001100, 682001110, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001130, 682001140, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001160, 682001170, 682001030] },
        { "actionSlotNum": 3, "skillIDList": [682001180, 682001190, 682001200] }
      ]
    }
  ]
}
```

### 4.2 Key Fields Explained

#### `uniqueAttribute`
```json
"uniqueAttribute": "CRIMSON"
```
`CRIMSON` is a sin attribute **exclusive to enemies and abnormalities**. It is not one of the seven player sin colors (SCARLET, AMBER, JADE, COBALT, INDIGO, VIOLET, SHAMROCK). Do not use it on player identities — it will crash `SinManager.Init()`. See [Section 9](#9-common-pitfalls-and-debugging) for details.

#### `appearance` and `unitScriptID`
Both reference character `1145` (LeiHeng). The appearance controls the 3D model and animations. The `unitScriptID` controls the C# script responsible for AI behavior and phase transitions — use the ID that matches the appearance rig.

#### `patternList`
Each entry is one round of the boss fight. The boss cycles through patterns in order. The 7-pattern cycle above escalates from basic Breath-building rounds (pattern 0) to the full 9-coin barrage (pattern 6, where `682001200` is in the last slot).

#### `startActionSlotNum`
How many skills the boss gets per round. `3` is standard.

#### `resistInfo`
Soji Abi is a swordsman — slash attacks slide off (×0.75) but blunt hits land harder (×1.5).

---

## 5. Designing 15 Skills

All skills live in `custom_limbus_data/skill/soji_skills.json` as a single `list` array.

### 5.1 Skill Architecture Summary

| ID | Name (KR) | Name (EN) | Coins | Special |
|---|---|---|---|---|
| 682001010 | 연격 | Consecutive Strike | 4 | Breath ×1 per hit |
| 682001020 | 이연잔 | Flowing Blade | 4 | Breath ×1 per hit |
| 682001030 | 잔상베기 | Afterimage Slash | 3 | Breath + PhantomIncision, SuperCoin on coin 3 |
| 682001040 | 폭압격 | Crushing Blow | 4 | CRIMSON heavy strike |
| 682001060 | 잔상 해방 | Afterimage Release | 4 | PhantomIncision burst |
| 682001080 | 사냥 | Hunt | 4 | — |
| 682001100 | 처형 | Execution | 4 | — |
| 682001110 | 공진 | Resonance | 4 | — |
| 682001130 | 이연속참 | Dual Continuous Slash | 4 | — |
| 682001140 | 필연쇄 | Inevitable Chain | 4 | — |
| 682001160 | 호흡 강탈 | Breath Steal | 4 | — |
| 682001170 | 정신 압박 | Mental Pressure | 4 | — |
| 682001180 | 취약 유도 | Weakness Induction | 4 | — |
| 682001190 | 취약 강타 | Weakness Strike | 4 | — |
| 682001200 | 광역 난사 | Area Barrage | **9** | VibrationExplosion + Vibration 5/5 on **all** coins |

### 5.2 Skill Template — 4-Coin Basic Attack

Below is the pattern used for most skills (shown with 연격 / Consecutive Strike):

```json
{
  "id": 682001010,
  "skillTier": 1,
  "skillType": "SKILL",
  "skillData": [
    {
      "attributeType": "CRIMSON",
      "atkType": "SLASH",
      "defType": "ATTACK",
      "targetNum": 1,
      "skillTargetType": "RANDOM",
      "canTeamKill": false,
      "canDuel": true,
      "canChangeTarget": true,
      "skillLevelCorrection": 3,
      "defaultValue": 8,
      "skillMotion": "S1",
      "viewType": "BATTLE",
      "parryingCloseType": "NEAR",
      "abilityScriptList": [
        { "scriptName": "EmptyBody" }
      ],
      "coinList": [
        {
          "operatorType": "ADD",
          "scale": 5,
          "abilityScriptList": [
            {
              "scriptName": "GiveBuffOnSucceedAttack",
              "buffData": {
                "buffKeyword": "Breath",
                "target": "Self",
                "stack": 1,
                "turn": 0,
                "activeRound": 0
              }
            }
          ]
        },
        { "operatorType": "ADD", "scale": 5, "abilityScriptList": [/* same */] },
        { "operatorType": "ADD", "scale": 5, "abilityScriptList": [/* same */] },
        { "operatorType": "ADD", "scale": 7, "abilityScriptList": [/* same */] }
      ]
    }
  ]
}
```

> **Note on `skillTargetType`:** Boss skills use `"RANDOM"` or `"ENCOUNTER"` targeting. Player identity skills use `"FRONT"`. Using `"FRONT"` on a boss will not cause a crash, but the targeting behaviour will be incorrect.

### 5.3 Skill with a Conditional Buff — `GiveBuffBeforeAttackIfOnesideAttack`

Several skills give bonus Breath only on an **uncontested** (one-sided) attack. Use this script in `abilityScriptList` (not inside a coin):

```json
"abilityScriptList": [
  { "scriptName": "EmptyBody" },
  {
    "scriptName": "GiveBuffBeforeAttackIfOnesideAttack",
    "buffData": {
      "buffKeyword": "Breath",
      "target": "Self",
      "stack": 5,
      "turn": 0,
      "activeRound": 0
    }
  }
]
```

This fires before the attack sequence, so the boss enters the clash already powered up.

### 5.4 SuperCoin — The Unflippable Final Coin

The third coin of 잔상베기 (Afterimage Slash) can never be tails. It also deals bonus critical damage:

```json
{
  "operatorType": "ADD",
  "grade": 2,
  "color": "GREY",
  "scale": 10,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "CriticalDmgUpByJsonValue", "value": 0.3 },
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": {
        "buffKeyword": "Breath",
        "target": "Self",
        "stack": 2,
        "turn": 0,
        "activeRound": 0
      }
    },
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": {
        "buffKeyword": "PhantomIncision",
        "target": "Target",
        "stack": 2,
        "turn": 0,
        "activeRound": 0
      }
    }
  ]
}
```

| Script | Effect |
|---|---|
| `SuperCoin` | Coin cannot be flipped to tails by any effect |
| `CriticalDmgUpByJsonValue` | Multiplies critical hit damage by `1 + value` (here ×1.3) |
| `grade: 2` | Renders the coin with a gold/special visual |

---

## 6. Writing Locale Files

### 6.1 Enemy Name — `enemyList/soji_locale.json`

The key is the **unit ID as a string**.

```json
{
  "6820001": {
    "a": 0,
    "id": "6820001",
    "name": "소지 아비",
    "desc": ""
  }
}
```

### 6.2 Skill Descriptions — `skillList/soji_skills_locale.json`

Each skill ID maps to a `levelList` array. Each level contains a `coinlist` array whose `coindescs` array must have **exactly as many entries as the skill has coins**.

```json
{
  "682001010": {
    "id": "682001010",
    "levelList": [
      {
        "level": 1,
        "name": "연격",
        "desc": "대상을 연속으로 베어 호흡을 쌓는다.",
        "flavor": "",
        "rawDesc": "",
        "abName": "",
        "coinlist": [
          {
            "coindescs": [
              { "desc": "[적중시] 호흡 1 획득", "summary": "" },
              { "desc": "[적중시] 호흡 1 획득", "summary": "" },
              { "desc": "[적중시] 호흡 1 획득", "summary": "" },
              { "desc": "[적중시] 호흡 1 획득", "summary": "" }
            ]
          }
        ]
      }
    ]
  }
}
```

> **Coin count mismatch warning:** If `coindescs` has a different number of entries than the actual `coinList` in your skill JSON, the in-game tooltip will display incorrectly or crash the UI. Always count your coins before writing locale.

### 6.3 Locale for the 9-Coin Barrage (광역 난사)

All 9 coins share the same Vibration Explosion effect. Coin 9 additionally grants Breath, so it gets its own description:

```json
"682001200": {
  "id": "682001200",
  "levelList": [
    {
      "level": 1,
      "name": "광역 난사",
      "desc": "광역으로 칼을 난사한다. 모든 적중마다 진동 폭발.",
      "flavor": "",
      "rawDesc": "",
      "abName": "",
      "coinlist": [
        {
          "coindescs": [
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여", "summary": "" },
            { "desc": "[파괴불가] [진동 폭발] [적중시] 진동 5/5턴 부여, 호흡 3 획득", "summary": "" }
          ]
        }
      ]
    }
  ]
}
```

---

## 7. Tuning and Buffing Skills

Out-of-the-box, a custom boss cloned from low-tier base game data often feels weak in sandbox play. The following parameters control difficulty directly:

| Parameter | Effect | Recommended buff |
|---|---|---|
| `defaultValue` | Base damage before coins | Raise by 2–4 from the source reference |
| `scale` | Coin damage multiplier | Raise by 1–3 per coin |
| `skillLevelCorrection` | Flat power bonus (can be negative) | Remove negatives; set to 2–5 |

**Before buffing — 연격 (original):**
```json
"defaultValue": 4,
"skillLevelCorrection": -1,
"coinList": [
  { "operatorType": "ADD", "scale": 3 },
  { "operatorType": "ADD", "scale": 3 },
  { "operatorType": "ADD", "scale": 3 },
  { "operatorType": "ADD", "scale": 5 }
]
```

**After buffing:**
```json
"defaultValue": 8,
"skillLevelCorrection": 3,
"coinList": [
  { "operatorType": "ADD", "scale": 5 },
  { "operatorType": "ADD", "scale": 5 },
  { "operatorType": "ADD", "scale": 5 },
  { "operatorType": "ADD", "scale": 7 }
]
```

Total power ceiling goes from roughly `4 + (−1) + 3+3+3+5 = 17` to `8 + 3 + 5+5+5+7 = 33`. Apply similar multipliers to all 15 skills for consistent difficulty.

---

## 8. Special Mechanic — VibrationExplosion on Every Hit

The centerpiece of the Soji Abi fight is `광역 난사` (Area Barrage): a 9-coin skill where **every single coin** triggers a Vibration Explosion and applies 5 stacks of Vibration for 5 turns.

### 8.1 Why This Is Powerful

`VibrationExplosion` deals bonus damage equal to the target's current Vibration stack count, then resets it. By applying 5 new stacks on the same hit that triggers the explosion, you create a chain:

```
Coin 1: Explosion (0 Vibration) → Apply Vibration 5
Coin 2: Explosion (5 Vibration) → Apply Vibration 5
Coin 3: Explosion (5 Vibration) → Apply Vibration 5
...
Coin 9: Explosion (5 Vibration) → Apply Vibration 5 + Breath 3
```

From coin 2 onward every explosion fires at full Vibration stacks.

### 8.2 Per-Coin Script Pattern

Apply this script block to every coin in the barrage. The order within `abilityScriptList` matters: explosion fires first, then Vibration is reapplied for the next coin.

```json
{
  "operatorType": "ADD",
  "scale": 6,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "barrageRandomS2" },
    {
      "scriptName": "VibrationExplosionOnSucceedAttack"
    },
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": {
        "buffKeyword": "Vibration",
        "target": "Target",
        "stack": 5,
        "turn": 5,
        "activeRound": 0
      }
    }
  ]
}
```

| Script | Role |
|---|---|
| `SuperCoin` | All 9 coins are unflippable — boss always lands all hits |
| `barrageRandomS2` | Triggers the rapid-fire animation frame per coin |
| `VibrationExplosionOnSucceedAttack` | Detonates current Vibration stacks on hit |
| `GiveBuffOnSucceedAttack: Vibration` | Reloads 5 stacks for the next coin |

### 8.3 The Final Coin (Coin 9)

The last coin additionally grants the boss 3 Breath:

```json
{
  "operatorType": "ADD",
  "scale": 6,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "s3_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": {
        "buffKeyword": "Vibration",
        "target": "Target",
        "stack": 5,
        "turn": 5,
        "activeRound": 0
      }
    },
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": {
        "buffKeyword": "Breath",
        "target": "Self",
        "stack": 3,
        "turn": 0,
        "activeRound": 0
      }
    }
  ]
}
```

> The animation script switches to `s3_frame0` for the closing blow, giving it a visually distinct final hit.

---

## 9. Common Pitfalls and Debugging

### 9.1 CRIMSON on a Player Identity

**Symptom:**
```
KeyNotFoundException: The given key 'CRIMSON' was not present in the dictionary.
  at ActionSlotDetail.GetOneSkillID (SinActionModel sinaction)
  at SinActionModel_Player.DequeueSin ()
  at SinManager.Init ()
```

**Cause:** `SinManager` builds sin-resource queues for every player unit. It expects only the seven canonical sin colors (SCARLET, AMBER, JADE, COBALT, INDIGO, VIOLET, SHAMROCK). `CRIMSON` is a boss-exclusive attribute and is not in the player sin dictionary.

The crash can be triggered by any of these on a **player identity**:
- A skill in `attributeList` with `"attributeType": "CRIMSON"`
- An `appearance` or `sdPortrait` that pulls in a boss character's skill pool
- A `unitKeywordList` entry ending in `_ENEMY` that links the unit to enemy data

**Fix:** Use only the seven valid sin colors in player identity skills. Never reference a boss unit ID in `sdPortrait`. Remove any `_ENEMY` suffix keywords from identities.

### 9.2 `attributeList` with `"number": 0` on All Skills

**Symptom:** Same `KeyNotFoundException: 'CRIMSON'` crash, even after confirming no skill has `attributeType: CRIMSON`.

**Cause:** When every skill in `attributeList` has `"number": 0`, the game treats the sin pool as empty and falls back to loading sin data from the character referenced by the `appearance` field. If that character is an enemy (e.g., LeiHeng's enemy rig) with CRIMSON skills in the base game, those skills enter the player sin pool and crash `SinManager`.

**Fix:** Set `number` to a positive value matching the skill tier:

```json
"attributeList": [
  { "skillId": 6821010, "number": 3 },
  { "skillId": 6821020, "number": 3 },
  { "skillId": 6821030, "number": 2 },
  { "skillId": 6821100, "number": 1 }
]
```

Reference from working mods:

| Skill Tier | Typical `number` Value |
|---|---|
| Tier 1 attack | 3 |
| Tier 2 attack | 2 |
| Defense / counter | 1 |

### 9.3 `unitScriptID` Null Crash on Identity

**Symptom:**
```
ArgumentNullException: Value cannot be null (type)
  at BattleUnitModel_Player.SetUnitScript()
```

**Cause:** Copying a boss's `unitScriptID` (e.g., `"unitScriptID": "1322"`) into a player identity JSON. The boss script class cannot be instantiated as a player unit script.

**Fix:** Remove `unitScriptID` entirely from the personality JSON. The game uses a default player script when the field is absent.

### 9.4 Locale Coin Count Mismatch

**Symptom:** In-game skill tooltip shows blank or misaligned coin descriptions, or the UI crashes when hovering the skill.

**Cause:** The number of `coindescs` entries in the locale file does not match the number of actual coins in the skill JSON.

**Fix:** Count coins in `soji_skills.json` and set exactly that many `coindescs` entries in the locale file. For `광역 난사` with 9 coins, there must be exactly 9 `coindescs`.

### 9.5 require-condition.json Errors

**Symptom:** Lines like:
```
[Error : Unity] Modular/TIMING:RoundStart/LUA:starteffects/... 에 해당되는 ID가 없습니다.
```

**Cause:** These are from *other mods* that depend on `require-condition.json` entries. They are not caused by Soji Abi and do not affect your boss. The errors appear because the modular passive system is checking IDs from every loaded mod simultaneously.

**Fix:** No action needed. These are noisy but harmless warnings from unrelated mods.

---

## 10. Final File Checklist

Use this before every test run.

### Boss Encounter Files

- [ ] `custom_encounters/encounter.json` — encounter ID, wave list, unit ID
- [ ] `custom_limbus_data/abnormality-unit/soji_unit.json` — unit stats, patternList, resistInfo
- [ ] `custom_limbus_data/skill/soji_skills.json` — all 15 skills, `attributeType`, coinList
- [ ] `custom_limbus_locale/KR/enemyList/soji_locale.json` — unit name under unit ID key
- [ ] `custom_limbus_locale/KR/skillList/soji_skills_locale.json` — all 15 skills, coindescs count matches

### Per-Skill Verification

For each skill in `soji_skills.json`:
- [ ] `id` is unique and in the `682001XXX` range
- [ ] `attributeType` is `CRIMSON` (or appropriate boss type — never `CRIMSON` on player)
- [ ] `atkType` matches the animation (SLASH / PENETRATE / HIT)
- [ ] `skillMotion` string exists in the appearance rig
- [ ] `coinList` length matches `coindescs` length in locale

### Stat Sanity Check

| Stat | Reasonable range for a buffed custom boss |
|---|---|
| `defaultValue` | 6–12 |
| Coin `scale` | 4–10 |
| `skillLevelCorrection` | 2–5 (avoid negatives) |
| `hp.defaultStat` | 500–900 |
| `defCorrection` | 3–7 |

---

*Next steps: if you want dynamic phase transitions, escalating patterns based on HP thresholds, or Lua-driven skill slot manipulation, continue with [Chapter 3 — Advanced Lua](chapter3_EN.md).*
