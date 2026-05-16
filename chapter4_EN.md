# Chapter 4 — Full Walkthrough Sample: Soji Abi (소지아비)

> **This chapter follows the exact same section order as Chapter 2 and Chapter 3, applied to a real boss built from scratch.**  
> Every file shown is the actual file used. Every mistake documented is real.

*Previous: [Chapter 3 — Advanced Lua](./chapter3_EN.md)*

---

## Table of Contents

**Part A — Building the Encounter (follows Chapter 2 structure)**

1. [Before You Begin](#1-before-you-begin)
2. [Mod Folder Structure](#2-mod-folder-structure)
3. [Registering a Battle Stage](#3-registering-a-battle-stage)
4. [Creating a Custom Boss Unit](#4-creating-a-custom-boss-unit)
5. [Creating Skills](#5-creating-skills)
6. [Creating a Custom Passive](#6-creating-a-custom-passive)
7. [Pattern Design](#7-pattern-design)
8. [Testing and Debugging](#8-testing-and-debugging)

**Part B — Adding Lua (follows Chapter 3 structure)**

9. [Why Lua?](#9-why-lua)
10. [File Structure and Wiring](#10-file-structure-and-wiring)
11. [TIMING — When Does It Execute?](#11-timing--when-does-it-execute)
12. [Available Lua Functions Used](#12-available-lua-functions-used)
13. [The Core Pattern: Direct Functions vs. Two-Step](#13-the-core-pattern-direct-functions-vs-two-step)
14. [Practical Examples: Three Lua Scripts](#14-practical-examples-three-lua-scripts)
15. [Debugging Lua Scripts](#15-debugging-lua-scripts)

---

# Part A — Building the Encounter

---

## 1. Before You Begin

### What This Chapter Builds

This chapter walks through **Soji Abi (소지아비)** — a 15-skill sword-fighter boss with a 7-slot action pattern and a 9-coin area barrage that chains Vibration Explosions on every hit.

End result:
- A custom battle node in the Playground area
- A boss using LeiHeng's model and animations
- 15 custom skills driven by two keywords: **Breath (호흡)** and **PhantomIncision (잔상)**
- A dynamic 3-cycle pattern system controlled entirely by Lua
- Three Lua scripts for animation control and pattern management

### Prerequisites

Same as Chapter 2. You need LetheLauncher, a text editor (VS Code recommended), and basic JSON knowledge.

### Concept Planning

Before writing any file, the boss design was pinned down on paper:

| Property | Decision | Reason |
|---|---|---|
| Model | LeiHeng (`1322_Pinky_Father2pAppearance`) | Sword animations fit a fluid, rhythmic fighting style |
| Boss attribute | `INDIGO` | Distinct from typical CRIMSON boss color |
| Slots per turn | 7 | Creates pressure without removing pattern variety |
| Stage level | 77 | Upper mid-game difficulty |
| Core keywords | `Breath`, `PhantomIncision` | Two-phase loop: accumulate → release |

**The two-phase design:**

```
Phase A — Breath accumulation  (연격, 이연잔, 잔상베기)
         ↓
Phase B — Payoff attacks using accumulated Breath
         ↓
Climax  — 광역 난사: 9 coins, each detonating Vibration and reloading 5 stacks
```

**ID scheme:**

```
encounter:   682001
unit:        6820001
part:        6820101   (unit + 100)
passive:     6820901   (unit + 900)
skills:      682001010 … 682001200   (steps of 10)
```

Always search `dumpedData/` (Ctrl+Shift+F in VS Code) to confirm these IDs are unused before writing anything. See Chapter 1, Section 5.

### What JSON Can Do vs. What Requires Lua

| JSON alone | Requires Lua |
|---|---|
| All 15 skill effects | Specifying animation frames per coin |
| Boss stats and resistances | Dynamic per-round pattern rotation |
| Passive registration | Forcing specific skills into specific slots |
| patternList (fallback) | Random frame selection on SuperCoin skills |

Sections 1–8 use only JSON. Lua is introduced starting at Section 9.

---

## 2. Mod Folder Structure

### Basic Layout

```
BepInEx/plugins/Lethe/mods/
└── 소지아비로/
    ├── custom_encounters/
    │   └── 소지아비로/
    │       ├── encounter.json
    │       └── subchapterui.json
    ├── custom_limbus_data/
    │   ├── abnormality-unit/
    │   │   └── soji.json
    │   ├── abnormality-part/
    │   │   └── soji_part.json
    │   ├── passive/
    │   │   └── SojiPassive.json
    │   └── skill/
    │       └── soji_skills.json
    ├── custom_limbus_locale/
    │   └── KR/
    │       ├── enemyList/
    │       │   └── soji_locale.json
    │       └── skillList/
    │           └── soji_skills_locale.json
    └── modular_lua/
        ├── SojiMotion.lua
        ├── SojiBarrage.lua
        └── SojiPattern.lua
```

### Folder Roles

| Folder | Role | Required? |
|---|---|---|
| `custom_encounters/` | Register the battle stage | **Required** |
| `custom_limbus_data/abnormality-unit/` | Define the boss unit | Required for custom boss |
| `custom_limbus_data/abnormality-part/` | Define the boss body stats | Required for custom boss |
| `custom_limbus_data/passive/` | Define custom passives | Required if using custom passives |
| `custom_limbus_data/skill/` | Define custom skills | Required for custom skills |
| `custom_limbus_locale/KR/` | Name and description text | Optional but strongly recommended |
| `modular_lua/` | Lua scripts | Required for animation and pattern logic |

---

## 3. Registering a Battle Stage

Two files, both inside the same subfolder:

```
custom_encounters/소지아비로/
├── encounter.json
└── subchapterui.json
```

### 3-1. encounter.json — The Battle Configuration

```json
{
    "id": 682001,
    "stageLevel": 77,
    "stageType": "Abnormality",
    "isBatonPassOn": true,
    "participantInfo": { "min": 1, "max": 12 },
    "battleCameraInfo": {
        "defaultPosX": 0.0,
        "defaultPosY": 0.0,
        "defaultPosZ": 40.0
    },
    "waveList": [
        {
            "battleMapInfo": {
                "mapName": "Cp9_HouseSpidersRoooftop",
                "mapSize": 33.33
            },
            "unitList": [
                { "unitID": 6820001, "unitCount": 1, "unitLevel": 77 }
            ],
            "bgmList": ["Battle_Cp9_Boss_5", "Battle_Cp9_Boss_6"],
            "allyPositionID": 175,
            "enemyPositionID": 164
        }
    ],
    "staminaCost": 0,
    "turnLimit": 99,
    "rewardList": [],
    "effectiveResistCondition": 1.2
}
```

**Key field descriptions:**

| Field | Description | Value used |
|---|---|---|
| `id` | Unique stage ID | `682001` |
| `stageType` | `"Abnormality"` for a boss fight | `"Abnormality"` |
| `isBatonPassOn` | Whether Baton Pass is allowed | `true` |
| `mapName` | Battle background map | `"Cp9_HouseSpidersRoooftop"` (Spider House rooftop) |
| `unitID` | The enemy unit to spawn | `6820001` |
| `unitLevel` | Enemy level | `77` |
| `bgmList` | Background music tracks | Two tracks — second activates mid-fight |
| `allyPositionID` / `enemyPositionID` | Spawn positions on the map | `175` / `164` |
| `staminaCost` | Set to `0` for sandbox access | `0` |
| `effectiveResistCondition` | EGO passive resist threshold | `1.2` |

### 3-2. subchapterui.json — Map Node Placement

```json
{
    "chapterId": 2,
    "subchapterId": 682001,
    "nodeId": 682001,
    "isUnlockByUnlockCode": false,
    "unlockCode": 101,
    "relatedData": { "chapterId": 2, "subchapterId": 0, "nodeId": 0 },
    "uiConfig": {
        "customChapterText": "소지아비로",
        "chapterTagIconType": "BATTLE",
        "region": "p",
        "mapAreaId": 101,
        "timeLine": "PLAYGROUND"
    },
    "type": "STAGE_NODE"
}
```

**Key field descriptions:**

| Field | Description |
|---|---|
| `subchapterId` / `nodeId` | **Must match** `encounter.json`'s `id` — all three must be `682001` |
| `isUnlockByUnlockCode: false` | Always accessible, no unlock required |
| `customChapterText` | Text shown on the map node |
| `timeLine: "PLAYGROUND"` | Places the node in the Playground area |

> Always use `timeLine: "PLAYGROUND"` for custom mods. It is always accessible regardless of story progress.

---

## 4. Creating a Custom Boss Unit

Two files are required:

```
custom_limbus_data/
├── abnormality-unit/soji.json
└── abnormality-part/soji_part.json
```

### 4-1. Finding Reference Data in dumpedData

Before writing the unit file, the reference character (LeiHeng, ID 1322) was located in `dumpedData/identities/`. Key values extracted:

```json
"appearance": "1322_Pinky_Father2pAppearance"

"passiveSet": {
  "passiveIdList": [132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998]
}
```

The appearance string is used directly. The passive IDs are copied as-is — these are base-game passives that require no custom files.

### 4-2. abnormality-unit — soji.json (key sections)

```json
{
  "list": [
    {
      "id": 6820001,
      "appearance": "1322_Pinky_Father2pAppearance",
      "classType": "ZAYIN",
      "attributeType": "INDIGO",
      "unitScriptID": "8380",
      "startActionSlotNum": 7,
      "maxActionSlotNum": 7,
      "specialDuelRank": 5,
      "hp": {
        "defaultStat": 532,
        "incrementByLevel": 18
      },
      "unitKeywordList": ["SMALL", "FINGER", "LITTLE_FINGER_FATHER_ENEMY"],
      "associationList": ["LITTLE_FINGER", "SPIDER_HOUSE"],
      "patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
      "abnormalityPartList": [6820101],
      "initBuffList": [],
      "hasMp": true,
      "panicType": 1070,
      "mentalConditionInfo": {
        "add": [{ "level": 1, "conditionIDList": [
          { "conditionID": "OnWinDuelAsParryingCountMultiply10AndPlus20Percent" }
        ]}],
        "min": [{ "level": 1, "conditionIDList": [
          { "conditionID": "OnLoseDuelAsParryingCountMultiply3AndPlus10Percent" }
        ]}]
      },
      "attributeList": [
        { "skillId": 682001010, "number": 0 },
        { "skillId": 682001020, "number": 0 },
        { "skillId": 682001030, "number": 0 },
        { "skillId": 682001040, "number": 0 },
        { "skillId": 682001060, "number": 0 },
        { "skillId": 682001080, "number": 0 },
        { "skillId": 682001100, "number": 0 },
        { "skillId": 682001110, "number": 0 },
        { "skillId": 682001130, "number": 0 },
        { "skillId": 682001140, "number": 0 },
        { "skillId": 682001160, "number": 0 },
        { "skillId": 682001170, "number": 0 },
        { "skillId": 682001180, "number": 0 },
        { "skillId": 682001190, "number": 0 },
        { "skillId": 682001200, "number": 0 }
      ],
      "passiveSet": {
        "passiveIdList": [132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998, 6820901]
      },
      "patternList": [ /* see Section 7 */ ]
    }
  ]
}
```

**Key field descriptions:**

| Field | Description | Value used |
|---|---|---|
| `id` | Unique unit ID | `6820001` |
| `appearance` | Character model to use | Copied from dumpedData |
| `classType` | Boss grade (`ZAYIN`, `TETH`, `HE`, `WAW`, `ALEPH`) | `"ZAYIN"` |
| `attributeType` | Boss sin color | `"INDIGO"` |
| `unitScriptID` | AI behavior script ID | `"8380"` (copied from reference boss) |
| `startActionSlotNum` | Skills used per turn | `7` |
| `abnormalityPartList` | Part IDs this unit owns | `[6820101]` |
| `patternID` | Pattern selection mode | `"PickByPattern_Abnormality_UptoActionSlotCnt"` |

> ⚠️ **Never put `999` in `startActionSlotNum`.** The engine processes it literally and breaks. Always use an explicit number.

**`attributeList` — registering available skills:**

```json
"attributeList": [
  { "skillId": 682001010, "number": 0 },
  { "skillId": 682001020, "number": 0 },
  ...
]
```

Every skill the boss can use must be listed here. Skills not in `attributeList` cannot be used in `patternList` or by `skillslotreplace()` in Lua.

> For boss (abnormality) units, `"number": 0` on all entries is correct — unlike player identities where `number` indicates sin cost. Boss units do not use the sin resource system.

**`passiveSet` — adding the custom passive:**

```json
"passiveSet": {
  "passiveIdList": [132219, 132224, ..., 9999998, 6820901]
}
```

IDs `132219`–`132230` were copied from the reference character's data. They are base-game passives that work with no custom files. `9999998` is a universal abnormality passive — always include it. `6820901` is the one custom passive, which triggers the Lua pattern controller (see Section 6).

**`mentalConditionInfo` — morale thresholds:**

```json
"mentalConditionInfo": {
  "add": [{ "level": 1, "conditionIDList": [
    { "conditionID": "OnWinDuelAsParryingCountMultiply10AndPlus20Percent" }
  ]}],
  "min": [{ "level": 1, "conditionIDList": [
    { "conditionID": "OnLoseDuelAsParryingCountMultiply3AndPlus10Percent" }
  ]}]
}
```

These were copied from the reference boss found in `dumpedData/limbus_data/abnormality-unit/`. They control how the boss's morale changes when it wins or loses clashes. Copying from a similar existing boss is the safest approach.

### 4-3. abnormality-part — soji_part.json

```json
{
  "list": [
    {
      "id": 6820101,
      "hp": { "defaultStat": 532, "incrementByLevel": 18 },
      "minSpeedList": [2],
      "maxSpeedList": [5],
      "partType": "BODY",
      "isDestroyable": "false",
      "spreadMpEffectToAbnormality": "true",
      "resistInfo": {
        "atkResistList": [
          { "type": "SLASH",     "value": 1.2 },
          { "type": "PENETRATE", "value": 1.0 },
          { "type": "HIT",       "value": 1.0 }
        ],
        "attributeResistList": [
          { "type": "CRIMSON",  "value": 1.2 },
          { "type": "SCARLET",  "value": 1.0 },
          { "type": "AMBER",    "value": 0.8 },
          { "type": "SHAMROCK", "value": 0.8 },
          { "type": "AZURE",    "value": 1.2 },
          { "type": "INDIGO",   "value": 0.8 },
          { "type": "VIOLET",   "value": 1.0 },
          { "type": "WHITE",    "value": 2.0 },
          { "type": "BLACK",    "value": 2.0 }
        ]
      },
      "passiveSet": { "passiveIdList": [132205] }
    }
  ]
}
```

The part's `id` (`6820101`) must match the number in `abnormalityPartList` in the unit file. The resistance profile: the boss is weak to slash (`1.2` = takes more damage), resistant to AMBER and SHAMROCK (`0.8`), and immune to WHITE/BLACK (`2.0`).

**Resistance value reference:**

| value | Effect |
|---|---|
| `0.5` | Weak — 50% extra damage taken |
| `1.0` | Normal |
| `2.0` | Resistant — 50% damage reduction |
| `0.0` | Immune |

---

## 5. Creating Skills

All 15 skills live in a single file: `custom_limbus_data/skill/soji_skills.json`.

### 5-1. Skill Overview

| ID | Name | Coins | Sin | Role |
|---|---|---|---|---|
| 682001010 | 연격 | 4 | VIOLET | Basic Breath builder, AoE 5 targets |
| 682001020 | 이연잔 | 5 | VIOLET | Main Breath builder, appears in every pattern |
| 682001030 | 잔상베기 | 1 | INDIGO | SuperCoin, PhantomIncision, conditional Breath |
| 682001040 | 폭압격 | — | INDIGO | Forced clash, target count reacts to duel result |
| 682001060 | 잔상 해방 | — | INDIGO | PhantomIncision burst |
| 682001080 | 사냥 | — | INDIGO | Hunt |
| 682001100 | 처형 | — | INDIGO | Heavy execution |
| 682001110 | 공진 | — | INDIGO | Vibration application |
| 682001130 | 이연속참 | — | VIOLET | Breath + chain |
| 682001140 | 필연쇄 | — | INDIGO | Chain skill |
| 682001160 | 호흡 강탈 | — | INDIGO | Debuff / Breath drain |
| 682001170 | 정신 압박 | — | INDIGO | Mental pressure |
| 682001180 | 취약 유도 | — | INDIGO | Weakness induction |
| 682001190 | 취약 강타 | — | INDIGO | Weakness strike |
| 682001200 | 광역 난사 | 9 | VIOLET | Climax — Vibration chain on all 9 coins |

### 5-2. Basic Skill Structure — 연격 (682001010)

```json
{
  "id": 682001010,
  "skillTier": 1,
  "skillType": "SKILL",
  "skillData": [{
    "attributeType": "VIOLET",
    "atkType": "SLASH",
    "defType": "ATTACK",
    "targetNum": 5,
    "skillTargetType": "RANDOM",
    "canDuel": true,
    "canChangeTarget": true,
    "skillLevelCorrection": 0,
    "defaultValue": 10,
    "skillMotion": "S1",
    "viewType": "ENCOUNTER",
    "parryingCloseType": "NEAR",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      {
        "scriptName": "GiveBuffOnUse",
        "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 0, "turn": 2 }
      }
    ],
    "coinList": [
      {
        "operatorType": "ADD",
        "scale": 4,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0" }
        ]
      },
      { /* coin[1] — same as coin[0] */ },
      { /* coin[2] — same as coin[0] */ },
      {
        "operatorType": "ADD",
        "scale": 7,
        "abilityScriptList": [
          { "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 2, "turn": 0 } },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame1" }
        ]
      }
    ]
  }]
}
```

**Skill base field descriptions:**

| Field | Description | Value |
|---|---|---|
| `skillTier` | Grade (1=basic, 2=intermediate, 3=advanced) | `1` |
| `attributeType` | Sin color | `"VIOLET"` |
| `atkType` | Attack type (`SLASH`, `PENETRATE`, `HIT`) | `"SLASH"` |
| `defType` | Defense type (`ATTACK`, `GUARD`, `EVADE`, `COUNTER`) | `"ATTACK"` |
| `targetNum` | Number of targets | `5` (AoE) |
| `skillTargetType` | How targets are selected | `"RANDOM"` |
| `defaultValue` | Base attack power | `10` |
| `skillLevelCorrection` | Power bonus per unit level | `0` |
| `skillMotion` | Animation to play | `"S1"` |
| `viewType` | Camera mode (`"BATTLE"` = single target, `"ENCOUNTER"` = all units) | `"ENCOUNTER"` |

**`abilityScriptList` (top-level) — fires when the skill is selected:**

`EmptyBody` is required on every abnormality skill. It declares a boss-type attack body and must always be present.

```json
{
  "scriptName": "GiveBuffOnUse",
  "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 0, "turn": 2 }
}
```

`stack: 0` does not add 0 stacks. It **refreshes the Breath buff's duration to 2 turns without changing the current count**. This keeps Breath alive across rounds — every time this skill is used, the timer resets. This is the mechanism that makes Breath a persistent resource.

> `GiveBuffOnUse` + `stack: 0` + `turn: N` = duration refresh only. Use it to maintain a buff without inflating stacks.

**Coin breakdown:**

| Coin | scale | Effect |
|---|---|---|
| coin[0] | 4 | Breath +1, plays S1 frame 0 |
| coin[1] | 4 | Breath +1, plays S1 frame 0 |
| coin[2] | 4 | Breath +1, plays S1 frame 0 |
| coin[3] | 7 | Breath +2, plays S1 frame 1 (finishing animation) |

The last coin has a higher `scale` (7 vs 4) for more damage on the final hit, and grants double Breath. The Lua call on each coin (`LUAMAIN:s1_frame0`, `s1_frame1`) controls which animation frame plays — explained fully in Section 14.

### 5-3. Coin System — Unbreakable and SuperCoin

These are two independent concepts often used together:

| Concept | JSON | Effect |
|---|---|---|
| **Unbreakable Coin** | `"grade": 2, "color": "GREY"` | The coin fires even when the player wins the clash |
| **SuperCoin** | `"scriptName": "SuperCoin"` in `abilityScriptList` | The coin flip is always Heads — power bonus always applies |

Used together on 잔상베기's single coin:

```json
{
  "operatorType": "ADD",
  "grade": 2,
  "color": "GREY",
  "scale": 9,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "CriticalDmgUpByJsonValue", "value": 0.3 },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "PhantomIncision", "target": "Target", "stack": 1, "turn": 0 } },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "PhantomIncision", "target": "Target", "stack": 1, "turn": 0 } }
  ]
}
```

Three effects layer on this single coin:
- Always fires, even if the player wins the clash (Unbreakable)
- Always deals full power bonus (SuperCoin)
- Always deals +30% critical damage (`CriticalDmgUpByJsonValue, value: 0.3`)

Note two separate `GiveBuffOnSucceedAttack` entries for PhantomIncision — both resolve sequentially, applying 2 total stacks in one hit.

### 5-4. Commonly Used abilityScripts

**Top-level scripts (fire when skill is selected):**

```json
"abilityScriptList": [
  { "scriptName": "EmptyBody" },
  {
    "scriptName": "GiveBuffBeforeAttackIfOnesideAttack",
    "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 5, "turn": 0 }
  },
  { "scriptName": "TagetNumAdderOnLoseDuel", "value": -2 }
]
```

| scriptName | Effect |
|---|---|
| `EmptyBody` | Required on every boss skill |
| `GiveBuffBeforeAttackIfOnesideAttack` | Grants Breath only if no clash occurs (player left the slot open) |
| `TagetNumAdderOnLoseDuel` | Adjusts target count when the boss loses a duel (`value: -2` = two fewer targets) |

`GiveBuffBeforeAttackIfOnesideAttack` fires *before* damage — so the Breath gained is already active when the hit resolves. If the player does contest the slot, this script never fires.

**Per-coin scripts (fire on each individual coin):**

| scriptName | Effect |
|---|---|
| `SuperCoin` | Coin flip always Heads |
| `GiveBuffOnSucceedAttack` | Apply a buff on a successful hit |
| `GiveBuffOnUse` | Apply a buff when the skill is selected |
| `VibrationExplosionOnSucceedAttack` | Detonate current Vibration stacks on target |
| `CriticalDmgUpByJsonValue` | Multiply critical hit damage by `1 + value` |

### 5-5. Practical Example: The 9-Coin Barrage — 광역 난사 (682001200)

Every one of the 9 coins has the same structure (coins 0–6). Coins 7 and 8 differ only in their animation call.

**Coins 0–6 (all identical):**

```json
{
  "operatorType": "ADD",
  "grade": 2,
  "color": "GREY",
  "scale": 6,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } }
  ]
}
```

**Coin 7 (fixed S6[0] animation):**

```json
{
  "operatorType": "ADD", "grade": 2, "color": "GREY", "scale": 6,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s6_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } }
  ]
}
```

**Coin 8 (final blow — fixed S3[0], grants Breath to self):**

```json
{
  "operatorType": "ADD", "grade": 2, "color": "GREY", "scale": 6,
  "abilityScriptList": [
    { "scriptName": "SuperCoin" },
    { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s3_frame0" },
    { "scriptName": "VibrationExplosionOnSucceedAttack" },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 } },
    { "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 3, "turn": 0 } }
  ]
}
```

**The Vibration damage chain:**

```
Coin 0: Explosion (0 stacks initially) → Apply Vibration 5 for 5 turns
Coin 1: Explosion (5 stacks)           → Apply Vibration 5 for 5 turns
Coin 2: Explosion (5 stacks)           → Apply Vibration 5 for 5 turns
...
Coin 7: Explosion (5 stacks), S6[0]   → Apply Vibration 5 for 5 turns
Coin 8: Explosion (5 stacks), S3[0]   → Apply Vibration 5 + Breath 3 to self
```

From coin 1 onward, every explosion fires at maximum stacks.

**Script execution order per coin:**

```
① SuperCoin              → flip bypassed, attack resolves
② Modular/ChangeMotion   → random or fixed animation frame (Lua)
③ VibrationExplosion     → detonate current stacks on target
④ GiveBuffOnSucceedAttack → reload Vibration 5 for next coin
```

Explosion (③) fires before reload (④). This is intentional — detonate what's there, then refill.

### 5-6. Locale Files

**enemyList/soji_locale.json — Boss display name:**

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

The top-level key is the unit ID as a **string**. Do not use a `dataList` array — that is the buff locale format, not enemy locale.

**skillList/soji_skills_locale.json — Skill descriptions:**

```json
{
  "682001010": {
    "id": "682001010",
    "levelList": [{
      "level": 1,
      "name": "연격",
      "desc": "대상을 연속으로 베어 호흡을 쌓는다.",
      "coinlist": [{
        "coindescs": [
          { "desc": "[적중시] 호흡 1 획득", "summary": "" },
          { "desc": "[적중시] 호흡 1 획득", "summary": "" },
          { "desc": "[적중시] 호흡 1 획득", "summary": "" },
          { "desc": "[적중시] 호흡 2 획득", "summary": "" }
        ]
      }]
    }]
  }
}
```

The `coindescs` array **must have exactly as many entries as the skill has coins**. 연격 has 4 coins → 4 `coindescs`. 광역 난사 has 9 coins → 9 `coindescs`. Mismatches cause broken or blank tooltips.

---

## 6. Creating a Custom Passive

Unlike Chapter 2's custom buffs, Soji Abi requires a **custom passive** — not for stat effects, but to fire a Lua function once per round. One file is needed:

```
custom_limbus_data/passive/SojiPassive.json
```

### 6-1. SojiPassive.json

```json
{
  "list": [
    {
      "id": 6820901,
      "attributeStockCondition": [],
      "requireIDList": [
        "Modular/TIMING:AfterSlots/LUA:SojiPattern/LUAMAIN:onSkillUse"
      ]
    }
  ]
}
```

This passive has no stat effects. Its only purpose is to trigger the Lua pattern controller. The `requireIDList` entry connects it to the ModularSkillScripts system.

Breaking down the entry:

```
Modular / TIMING:AfterSlots / LUA:SojiPattern / LUAMAIN:onSkillUse
  │              │                  │                    │
  │              │                  │                    └─ Function to call
  │              │                  └─ modular_lua/SojiPattern.lua
  │              └─ Fires after slot assignment, before skill execution
  └─ ModularSkillScripts system
```

`TIMING:AfterSlots` is the only timing where `skillslotreplace()` works correctly:

| Timing | `skillslotreplace()` result |
|---|---|
| Before `AfterSlots` | No effect — slots are not yet assigned |
| **`AfterSlots`** | ✓ **Works — slots are assigned but not yet executed** |
| During skill execution | No effect — skills are already resolving |

### 6-2. Registering the Passive on the Unit

The custom passive ID (`6820901`) is added at the end of `passiveSet.passiveIdList` in `soji.json`:

```json
"passiveSet": {
  "passiveIdList": [
    132219, 132224, 132220, 132222, 132223, 132214, 132230,
    9999998,
    6820901
  ]
}
```

No other registration is needed. The game reads the passive file automatically from `custom_limbus_data/passive/`.

---

## 7. Pattern Design

### 7-1. patternList Structure

The `patternList` in `soji.json` cycles through one entry per turn. Each entry defines the skill assigned to each of the 7 slots:

```json
"patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
"patternList": [
  {
    "slotList": [
      {"skillParentList": [{"skillChildList": [{"skillID": 682001200, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001030, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001110, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001100, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001140, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001060, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 682001020, "chance": 1}], "chance": 1}]}
    ]
  },
  { /* pattern 2 */ },
  { /* pattern 3 ... */ }
]
```

One slot written compactly:

```json
{"skillParentList": [{"skillChildList": [{"skillID": 682001200, "chance": 1}], "chance": 1}]}
```

`chance: 1` means this skill is always selected for this slot with 100% probability. The nested `skillParentList` → `skillChildList` structure is required — it supports weighted randomization, but `chance: 1` keeps it deterministic.

### 7-2. Soji Abi's Pattern Design Philosophy

The `patternList` in this mod is a **fallback only**. The real pattern logic is handled by Lua (Section 14). However, the `patternList` must still be present with valid skill IDs — the engine requires it.

All 6 patterns in `soji.json` share one rule: **682001200 (광역 난사) is always in slot 0**. This serves as a safe default in case the Lua override fails for any reason.

The Lua controller replaces all 7 slots every round before any skill executes, using a 3-cycle system with random variants per cycle:

| Cycle | Rounds | Focus | Slot 0 guaranteed |
|---|---|---|---|
| Cycle 1 | 1, 4, 7… | Area barrage + heavy pressure | 682001200 always |
| Cycle 2 | 2, 5, 8… | Breath accumulation | 682001020 or 682001140 |
| Cycle 3 | 3, 6, 9… | PhantomIncision pressure | 682001110 or 682001040 |

See Section 14-3 for the full Lua implementation.

---

## 8. Testing and Debugging

### 8-1. Applying the Mod

1. Copy the finished `소지아비로/` folder to:
   ```
   BepInEx/plugins/Lethe/mods/소지아비로/
   ```
2. Launch the game using LetheLauncher.
3. Go to the stage select screen → Playground area.
4. If the `소지아비로` node appears, the encounter registration worked.

> File changes made while the game is running are not reflected. Always restart after making changes.

### 8-2. Checking Errors

Open `BepInEx/LogOutput.log`. Relevant lines:

```
[Info   :ModularSkillScripts] [SojiPattern] Round=1 Cycle=1 Slot0=682001200 Slots=7
[Info   :ModularSkillScripts] [SojiMotion] S1[0]
[Info   :ModularSkillScripts] [SojiBarrage] S2[2]
```

Errors look like:

```
[Error  :ModularSkillScripts] attempt to index a nil value (global 'something')
```

### 8-3. Common Mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Stage node doesn't appear | `subchapterId`/`nodeId` doesn't match `encounter.json`'s `id` | Make all three values equal |
| Boss uses only 1–2 skills | `startActionSlotNum` set to `999` | Change to an explicit number like `7` |
| Skill never appears | Skill ID missing from `attributeList` | Add it |
| Part ID error | `abnormalityPartList` ID doesn't match part's `id` | Align both to `6820101` |
| Tooltip shows wrong coin count | `coindescs` count doesn't match `coinList` length | Count coins and match exactly |
| Lua function never fires | File name or function name misspelled | `LUA:SojiMotion` must match `SojiMotion.lua` exactly |

---

# Part B — Adding Lua

---

## 9. Why Lua?

JSON defines *what* happens. Lua defines *when and how* it happens based on runtime state.

**Things JSON cannot do on its own:**

| Desired behavior | Reason JSON fails |
|---|---|
| Play S1 frame 0 for coins 0–2, frame 1 for coin 3 | JSON has no per-coin frame selector |
| Pick a random animation frame each hit | JSON has no `random()` |
| Replace specific slots with specific skills each round | JSON has no `skillslotreplace()` |
| Rotate through 3 different cycles across rounds | JSON has no round counter or state |

All four of these are required for Soji Abi. Hence three Lua files.

---

## 10. File Structure and Wiring

### 10-1. File Location

```
소지아비로/
└── modular_lua/
    ├── SojiMotion.lua
    ├── SojiBarrage.lua
    └── SojiPattern.lua
```

The filename (without `.lua`) is the name used in `LUA:` references.

### 10-2. Calling Lua from JSON (skill coin)

```json
{
  "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0"
}
```

Breaking down the string:

```
Modular / TIMING:ChangeMotion / LUA:SojiMotion / LUAMAIN:s1_frame0
  │              │                   │                   │
  │              │                   │                   └─ Function to call
  │              │                   └─ modular_lua/SojiMotion.lua
  │              └─ When to call it (immediately before animation)
  └─ Uses ModularSkillScripts system
```

### 10-3. Calling Lua from a Passive (round trigger)

```json
"requireIDList": [
  "Modular/TIMING:AfterSlots/LUA:SojiPattern/LUAMAIN:onSkillUse"
]
```

The same format, different `TIMING`. This fires once per round rather than per coin.

---

## 11. TIMING — When Does It Execute?

| TIMING | When it fires | Used in this mod |
|---|---|---|
| `ChangeMotion` | Immediately before a coin's animation plays | SojiMotion, SojiBarrage |
| `AfterSlots` | After skill slots are assigned, before any skill executes | SojiPattern (via passive) |
| `OnCoinToss` | When the coin is flipped | ⚠️ Not used — does not fire on SuperCoin |
| `BeforeAttack` | Immediately before attack resolution | — |
| `RoundStart` | At the start of the round | — |

> ⚠️ **`OnCoinToss` does not fire on SuperCoin skills.**  
> The first attempt to randomize barrage frames used `OnCoinToss`. It never executed because `SuperCoin` bypasses the flip entirely. Switching to `ChangeMotion` fixed it — `ChangeMotion` fires for all coins, flip or no flip.

---

## 12. Available Lua Functions Used

```lua
-- Change the animation playing for this coin (only at TIMING:ChangeMotion)
changemotion("S1", 0)
-- changemotion(motionName, frameIndex)

-- Generate a random integer, inclusive on both ends
local frame = random(0, 2)   -- returns 0, 1, or 2

-- Get the current round number
local r = round()

-- Get number of skill slots
local n = skillslotcount("Self")

-- Replace the skill in a slot
skillslotreplace(slotIndex, fromSkillID, toSkillID)
-- Replaces slot[slotIndex] with toSkillID, but only if it currently holds fromSkillID

-- Write a debug line to LogOutput.log
log("message", 0)
```

---

## 13. The Core Pattern: Direct Functions vs. Two-Step

Chapter 3 introduced the **main + LUAMAIN two-step** pattern: store the coin index in `main()`, then read it in a `LUAMAIN` function. Soji Abi uses a simpler approach for animation control.

**Chapter 3's two-step approach:**

```lua
function main()
    local coinIdx = getactivations()
    setldata("Self", "myKey", coinIdx)  -- store
end

function applyMotion()
    local coinIdx = getldata("Self", "myKey") or 0  -- retrieve
    changemotion("S10", coinIdx)
end
```

Used when: the frame to play **depends on a runtime value** computed at coin execution time.

**Soji Abi's direct function approach:**

```lua
function s1_frame0()
    changemotion("S1", 0)
end

function s1_frame1()
    changemotion("S1", 1)
end
```

Each coin calls the function that exactly matches its intended frame. No `main()`, no storage, no retrieval.

Used when: the frame assignment is **fixed at design time** and written directly into the JSON.

| | Two-step | Direct function |
|---|---|---|
| `main()` required | Yes | No |
| `setldata`/`getldata` | Yes | No |
| Frame decision made | At runtime in Lua | At authoring time in JSON |
| Best for | Frame depends on buff count, round number, etc. | Frame is always the same for this coin |

Soji Abi uses direct functions for all fixed-frame cases. The two-step approach is reserved for `SojiPattern`, where the pattern variant is chosen at runtime.

---

## 14. Practical Examples: Three Lua Scripts

### 14-1. SojiMotion.lua — Per-Coin Fixed Frame Control

**Full file:**

```lua
-- SojiMotion.lua
-- Directly assigns animation frames per coin via individual functions.
-- Each coin in the JSON calls the function matching its intended frame.
-- No getactivations(), no setldata/getldata required.

-- S1 motion
-- 682001010: coin[0][1][2] → S1[0], coin[3] → S1[1]
function s1_frame0()
    changemotion("S1", 0)
    log("[SojiMotion] S1[0]", 0)
end

function s1_frame1()
    changemotion("S1", 1)
    log("[SojiMotion] S1[1]", 0)
end

-- S2 motion
-- 682001020: coin[0][1] → S2[1], coin[2] → S2[2], coin[3] → S2[1], coin[4] → S2[2]
function s2_frame1()
    changemotion("S2", 1)
    log("[SojiMotion] S2[1]", 0)
end

function s2_frame2()
    changemotion("S2", 2)
    log("[SojiMotion] S2[2]", 0)
end

-- S6 and S3: used by 광역 난사 coin[7] and coin[8]
function s6_frame0()
    changemotion("S6", 0)
    log("[SojiBarrage] apply coin=7 -> S6[0]", 0)
end

function s3_frame0()
    changemotion("S3", 0)
    log("[SojiBarrage] apply coin=8 -> S3[0]", 0)
end

-- No main() defined. Direct function approach replaces the two-step pattern.
```

**Why each skill has its own frame layout:**

The S1 animation has 2 frames (frame 0 and frame 1). A 4-coin skill on S1 would normally play frame 0 for coin 0, frame 1 for coin 1, then repeat frame 1 for coins 2 and 3. To control this precisely, coins 0–2 call `s1_frame0()` and coin 3 calls `s1_frame1()`.

The S2 animation has 3 frames (0, 1, 2). 이연잔 is a 5-coin skill, so frame assignments cycle: `s2_frame1`, `s2_frame1`, `s2_frame2`, `s2_frame1`, `s2_frame2`.

**How each coin connects:**

In `soji_skills.json`, each coin of 연격 (682001010):

```json
"coinList": [
  {
    "scale": 4,
    "abilityScriptList": [
      {"scriptName": "GiveBuffOnSucceedAttack", "buffData": {"buffKeyword": "Breath", "stack": 1, ...}},
      {"scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0"}
    ]
  },
  { /* coin[1]: s1_frame0 */ },
  { /* coin[2]: s1_frame0 */ },
  {
    "scale": 7,
    "abilityScriptList": [
      {"scriptName": "GiveBuffOnSucceedAttack", "buffData": {"buffKeyword": "Breath", "stack": 2, ...}},
      {"scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame1"}
    ]
  }
]
```

**Step-by-step execution for coin[3]:**

```
coin[3] selected
  ↓
Attack resolves (GiveBuffOnSucceedAttack: Breath +2)
  ↓
TIMING:ChangeMotion fires
  → s1_frame1() called
  → changemotion("S1", 1)
  → S1 frame 1 plays (finishing slash animation)
```

### 14-2. SojiBarrage.lua — Random Frame on Every Barrage Hit

**Full file:**

```lua
-- SojiBarrage.lua
-- Picks a random S2 frame (0, 1, or 2) for each of the 7 middle barrage coins.
--
-- IMPORTANT: OnCoinToss does NOT fire on SuperCoin skills.
-- SuperCoin bypasses the coin flip, so OnCoinToss never triggers.
-- Use ChangeMotion instead — it fires for all coins regardless of flip status.

function barrageRandomS2()
    local frame = random(0, 2)
    changemotion("S2", frame)
    log("[SojiBarrage] S2[" .. frame .. "]", 0)
end
```

**The SuperCoin timing problem — and the fix:**

The first attempt wired this to `TIMING:OnCoinToss`. The function never executed.

`SuperCoin` tells the game: skip the coin flip, treat the result as Heads. Since no flip occurs, `OnCoinToss` — which fires when a flip happens — never triggers. Switching to `ChangeMotion` solved it because `ChangeMotion` fires based on animation rendering, not on the flip.

```
Incorrect: "Modular/TIMING:OnCoinToss/LUA:SojiBarrage/LUAMAIN:barrageRandomS2"
             → never fires on SuperCoin

Correct:   "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2"
             → fires for all coins, SuperCoin or not
```

**Execution flow for one barrage coin:**

```
SuperCoin     → flip skipped, coin resolves as Heads
              → OnCoinToss: never fires
Attack resolves (VibrationExplosion + Vibration buff applied)
ChangeMotion timing fires
  → barrageRandomS2() called
  → frame = random(0, 2)  →  e.g. 2
  → changemotion("S2", 2)
  → S2 frame 2 plays
```

### 14-3. SojiPattern.lua — 3-Cycle Pattern Controller

**Full file:**

```lua
-- SojiPattern.lua
-- Replaces all 7 skill slots every round using a 3-cycle rotation.
-- Triggered by the custom passive (SojiPassive.json) at TIMING:AfterSlots.

local CYCLE_PATTERNS = {
    -- Cycle 1 (rounds 1, 4, 7...): 광역 난사 always in slot 0
    {
        {682001200, 682001030, 682001110, 682001100, 682001140, 682001060, 682001020},
        {682001200, 682001080, 682001040, 682001110, 682001100, 682001020, 682001060},
        {682001200, 682001030, 682001080, 682001110, 682001040, 682001100, 682001020}
    },
    -- Cycle 2 (rounds 2, 5, 8...): Breath accumulation
    {
        {682001020, 682001010, 682001110, 682001130, 682001100, 682001060, 682001020},
        {682001140, 682001020, 682001010, 682001110, 682001080, 682001060, 682001020}
    },
    -- Cycle 3 (rounds 3, 6, 9...): PhantomIncision pressure
    {
        {682001110, 682001040, 682001080, 682001030, 682001100, 682001060, 682001020},
        {682001040, 682001110, 682001030, 682001030, 682001140, 682001020, 682001060}
    }
}

local ALL_SKILLS = {
    682001010, 682001020, 682001030, 682001040,
    682001060, 682001080, 682001100, 682001110,
    682001130, 682001140, 682001160, 682001170,
    682001180, 682001190, 682001200
}

local function ForceSlot(slotIdx, targetSkillID)
    for _, sid in ipairs(ALL_SKILLS) do
        skillslotreplace(slotIdx, sid, targetSkillID)
    end
end

local function ApplyCyclePattern(cycleIdx)
    local patterns   = CYCLE_PATTERNS[cycleIdx]
    local chosen     = patterns[random(1, #patterns)]
    local totalSlots = skillslotcount("Self")

    for i = 1, math.min(totalSlots, #chosen) do
        ForceSlot(i - 1, chosen[i])
    end

    log("[SojiPattern] Round=" .. round() .. " Cycle=" .. cycleIdx
        .. " Slot0=" .. chosen[1] .. " Slots=" .. totalSlots, 0)
end

function onSkillUse()
    local cycleIdx = (round() - 1) % 3 + 1
    ApplyCyclePattern(cycleIdx)
end
```

**Breaking down `CYCLE_PATTERNS`:**

```lua
local CYCLE_PATTERNS = {
    -- Cycle 1: 3 variants, all start with 682001200
    {
        {682001200, 682001030, ...},   -- variant A
        {682001200, 682001080, ...},   -- variant B
        {682001200, 682001030, ...}    -- variant C
    },
    ...
}
```

Each round, one variant from the matching cycle is selected at random. This means the same cycle index feels different each occurrence. Note: `682001020` (이연잔) appears in every single cycle to ensure Breath always builds regardless of which variant is chosen.

**Breaking down `ForceSlot`:**

```lua
local function ForceSlot(slotIdx, targetSkillID)
    for _, sid in ipairs(ALL_SKILLS) do
        skillslotreplace(slotIdx, sid, targetSkillID)
    end
end
```

`skillslotreplace(slot, fromSkill, toSkill)` replaces the skill in `slot` **only if its current occupant equals `fromSkill`**. The script does not know what skill the `patternList` assigned to each slot, so it tries replacing from every possible skill ID. Exactly one attempt will match. All others are no-ops. This is the only way to "force" a slot without knowing its current content.

**Breaking down `onSkillUse` — the cycle index formula:**

```lua
local cycleIdx = (round() - 1) % 3 + 1
```

```
Round 1: (1-1) % 3 + 1 = 1  → Cycle 1
Round 2: (2-1) % 3 + 1 = 2  → Cycle 2
Round 3: (3-1) % 3 + 1 = 3  → Cycle 3
Round 4: (4-1) % 3 + 1 = 1  → Cycle 1  ← repeats
Round 5: (5-1) % 3 + 1 = 2  → Cycle 2
...
```

**The full flow every round:**

```
Round starts
  → patternList assigns skills to slots (default JSON behavior)
AfterSlots timing fires
  → passive 6820901 triggers onSkillUse()
  → cycleIdx = (round - 1) % 3 + 1
  → random variant chosen from CYCLE_PATTERNS[cycleIdx]
  → ForceSlot(0..6) replaces all 7 slots with chosen pattern
Skills execute — using the Lua-assigned loadout, not patternList
```

---

## 15. Debugging Lua Scripts

### 15-1. Inspecting Values with log()

```lua
function onSkillUse()
    local cycleIdx = (round() - 1) % 3 + 1
    log("[SojiPattern] Round=" .. round() .. " Cycle=" .. cycleIdx, 0)
    ApplyCyclePattern(cycleIdx)
end
```

Open `BepInEx/LogOutput.log` during or after a fight:

```
[Info   :ModularSkillScripts] [SojiPattern] Round=1 Cycle=1 Slot0=682001200 Slots=7
[Info   :ModularSkillScripts] [SojiMotion] S1[0]
[Info   :ModularSkillScripts] [SojiBarrage] S2[1]
```

Use this to verify:
- The correct cycle fires each round
- The correct pattern variant was selected
- Animation frames are resolving as expected

### 15-2. Common Lua Errors

| Symptom | Cause | Fix |
|---|---|---|
| Function never executes | `TIMING:OnCoinToss` on a SuperCoin skill | Change to `TIMING:ChangeMotion` |
| `skillslotreplace` has no effect | `from` skill ID doesn't match what's in the slot | Use `ForceSlot` — iterate all skill IDs |
| `[Error: ModularSkillScripts]` | Lua syntax error | Check for missing `end`, unmatched `"`, misplaced `,` |
| Function not found | Filename or function name mismatch | `LUA:SojiMotion` must exactly match `SojiMotion.lua` |
| `nil` value error | `getldata` returned nil, used without check | Add `or defaultValue` after any `getldata` call |
| Animation doesn't change | `changemotion` called at wrong timing | Confirm `TIMING:ChangeMotion` is set |

### 15-3. Quick Lua Syntax Reference

```lua
-- Variables
local x = 10
local name = "soji"

-- Conditionals
if x > 5 then
    -- ...
elseif x == 3 then
    -- ...
else
    -- ...
end

-- Loops
for i = 1, 7 do
    ForceSlot(i - 1, chosen[i])  -- note: slot is 0-indexed, i is 1-indexed
end

-- Tables (arrays, 1-indexed in Lua)
local t = {10, 20, 30}
local first = t[1]   -- 10
local len   = #t     -- 3

-- Tables (dictionaries)
local map = {[682001010] = "S1", [682001020] = "S2"}
local val = map[682001010]  -- "S1"

-- nil check
if val == nil then
    -- value is absent
end

-- String concatenation
local msg = "Round: " .. round()  -- use ".." operator
```

---

## Full Chapter Summary

| Section | Chapter 2 equivalent | Content |
|---|---|---|
| 1 | Before You Begin | Boss concept, two-keyword design, ID scheme |
| 2 | Mod Folder Structure | All 12 files across 7 directories |
| 3 | Registering a Battle Stage | `encounter.json` + `subchapterui.json` |
| 4 | Creating a Custom Boss Unit | `soji.json` + `soji_part.json`, reference data extraction |
| 5 | Creating Skills | All 15 skills; `GiveBuffOnUse stack:0`; SuperCoin+Unbreakable; 9-coin Vibration chain |
| 6 | Creating a Custom Passive | `SojiPassive.json`; `requireIDList`; `AfterSlots` timing |
| 7 | Pattern Design | `patternList` as fallback; 3-cycle philosophy |
| 8 | Testing and Debugging | Applying the mod; reading logs; common mistakes |
| 9 | Why Lua? | Chapter 3 equivalent |
| 10 | File Structure and Wiring | Chapter 3 equivalent |
| 11 | TIMING | Chapter 3 equivalent; `OnCoinToss` SuperCoin caveat |
| 12 | Available Functions | Chapter 3 equivalent |
| 13 | Core Pattern | Direct functions vs. two-step; when to use each |
| 14 | Practical Examples | `SojiMotion.lua`, `SojiBarrage.lua`, `SojiPattern.lua` |
| 15 | Debugging | Chapter 3 equivalent |

---

*For the underlying framework concepts that this chapter applies, see [Chapter 2](./chapter2_EN.md) and [Chapter 3](./chapter3_EN.md).*
