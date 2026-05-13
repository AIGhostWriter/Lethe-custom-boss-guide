# Chapter 2 — Creating an Encounter Mod
> **Lethe Framework · Yellow Harpoon (Vespa Boss) as the Running Example**

*Previous: [Chapter 1 — Finding Your Boss Data](./chapter1_EN.md) | Next: [Chapter 3 — Advanced Lua](./chapter3_EN.md)*

---

## Table of Contents

1. [Before You Begin](#1-before-you-begin)
2. [Mod Folder Structure](#2-mod-folder-structure)
3. [Registering a Battle Stage](#3-registering-a-battle-stage)
4. [Creating a Custom Boss Unit](#4-creating-a-custom-boss-unit)
5. [Creating Skills](#5-creating-skills)
6. [Creating Custom Buffs](#6-creating-custom-buffs)
7. [Pattern Design](#7-pattern-design)
8. [Testing and Debugging](#8-testing-and-debugging)
9. [Going Further](#9-going-further)

---

## 1. Before You Begin

### What This Guide Builds

This guide uses the **Yellow Harpoon** mod as its running example, walking through every step needed to add a completely new boss battle stage to Limbus Company.

End result:
- A custom battle node registered in the Playground map
- A custom boss using Vespa's appearance
- 10 custom skills and a 12-turn action pattern cycle

### Prerequisites

| Item | Description |
|------|-------------|
| LetheLauncher | The launcher that includes the Lethe framework |
| Text Editor | VS Code recommended (JSON syntax highlighting) |
| Basic JSON Knowledge | Understand `{}`, `[]`, `""`, and `,` rules |

### What JSON Can Do vs. What Requires Lua

| JSON Alone | Requires Lua Script |
|------------|---------------------|
| Register a battle stage | Specify animation frames per coin |
| Configure boss stats and resistances | Conditional pattern switching |
| Set skill effects (buffs, explosions, etc.) | Complex phase transition logic |
| Design action pattern cycles | Custom scripted calculations |

Sections 1–6 of this guide use JSON only. For Lua advanced content, see Chapter 3.

---

## 2. Mod Folder Structure

### Basic Layout

In Lethe, one folder inside `mods/` = one mod.

```
BepInEx/plugins/Lethe/mods/
└── MyModName/
    ├── custom_encounters/       ← Battle stage definitions
    ├── custom_limbus_data/      ← Units, skills, buffs, and other game data
    ├── custom_limbus_locale/    ← Text (names, descriptions)
    ├── custom_sprites/          ← Skill icon images
    └── modular_lua/             ← Lua scripts (advanced)
```

### Folder Roles

| Folder | Role | Required? |
|--------|------|-----------|
| `custom_encounters/` | Register battle stages | **Required** |
| `custom_limbus_data/abnormality-unit/` | Define boss units | Required for custom boss |
| `custom_limbus_data/abnormality-part/` | Define boss parts (bodies) | Required for custom boss |
| `custom_limbus_data/skill/` | Define custom skills | Required for custom skills |
| `custom_limbus_locale/` | Name and description text | Optional |
| `modular_lua/` | Lua scripts | For advanced features |

### Yellow Harpoon's Actual Folder Structure

```
YellowHarpoon/
├── custom_encounters/
│   └── YellowHarpoon/
│       ├── encounter.json         ← Battle configuration
│       └── subchapterui.json      ← UI/map node placement
├── custom_limbus_data/
│   ├── abnormality-unit/
│   │   └── vespa.json             ← Boss unit body
│   ├── abnormality-part/
│   │   └── part.json              ← Boss part (body stats)
│   └── skill/
│       └── vespa_skills.json      ← 10 custom skills
└── modular_lua/
    └── VespaMotion.lua            ← Animation control (advanced)
```

> **Tip:** To start, you only need the 2 files inside `custom_encounters/` — enough to create a stage that spawns existing game units.

---

## 3. Registering a Battle Stage

A battle stage consists of exactly two files, both inside the same folder.

```
custom_encounters/
└── EncounterName/
    ├── encounter.json
    └── subchapterui.json
```

### 3-1. encounter.json — The Battle Configuration

This file defines all the rules for the fight.

**Yellow Harpoon's actual file:**

```json
{
    "id": 681001,
    "stageLevel": 60,
    "stageType": "Abnormality",
    "isBatonPassOn": true,
    "participantInfo": {
        "min": 1,
        "max": 12
    },
    "battleCameraInfo": {
        "defaultPosX": 0.0,
        "defaultPosY": 0.0,
        "defaultPosZ": 40.0
    },
    "waveList": [
        {
            "battleMapInfo": {
                "mapName": "Cp9_Middlefinger",
                "mapSize": 33.0
            },
            "unitList": [
                {
                    "unitID": 6810001,
                    "unitCount": 1,
                    "unitLevel": 87
                }
            ],
            "bgmList": ["Battle_Cp9_Boss_3_2"],
            "allyPositionID": 58,
            "enemyPositionID": 57
        }
    ],
    "staminaCost": 0,
    "recommendedLevel": 60,
    "turnLimit": 99,
    "rewardList": [],
    "effectiveResistCondition": 1.2
}
```

**Key field descriptions:**

| Field | Description | Yellow Harpoon Value |
|-------|-------------|----------------------|
| `id` | Unique stage ID. Use **100,000+** to be safe | `681001` |
| `stageType` | `"Abnormality"` (boss) or `"Normal"` (standard) | `"Abnormality"` |
| `isBatonPassOn` | Whether Baton Pass is allowed | `true` |
| `participantInfo` | Range of ally participants allowed | 1–12 |
| `waveList` | List of battle waves (array) | 1 wave |
| `unitID` | Enemy unit ID | `6810001` |
| `unitLevel` | Enemy level | `87` |
| `mapName` | Battle map name | `"Cp9_Middlefinger"` |
| `staminaCost` | Stamina cost (`0` = free) | `0` |
| `turnLimit` | Max turns (`99` = effectively unlimited) | `99` |

**Common mapName values:**

| mapName | Atmosphere |
|---------|-----------|
| `"SmokeWar_v1"` | Urban street |
| `"Cp7.5-1_LCE_Lab_v4"` | Laboratory |
| `"Cp9_Middlefinger"` | Middle Finger building |
| `"Cp6_Boss"` | Wide boss arena |

> **Note:** If `unitID` points to a custom unit, the unit data must exist in `custom_limbus_data/abnormality-unit/`. For existing game units, look up the ID in `dumpedData/limbus_data/enemy/`.

---

### 3-2. subchapterui.json — Map Node Placement

This file controls where the battle node appears on the stage select screen.

**Yellow Harpoon's actual file:**

```json
{
    "chapterId": 2,
    "subchapterId": 681001,
    "nodeId": 681001,
    "storyIdInTheaterData": 0,
    "isUnlockByUnlockCode": false,
    "unlockCode": 101,
    "relatedData": {
        "chapterId": 2,
        "subchapterId": 0,
        "nodeId": 0
    },
    "uiConfig": {
        "customChapterText": "Yellow Harpoon",
        "chapterTagIconType": "BATTLE",
        "region": "p",
        "mapAreaId": 101,
        "timeLine": "PLAYGROUND",
        "illustId": "story-id_E041X"
    },
    "type": "STAGE_NODE"
}
```

**Key field descriptions:**

| Field | Description |
|-------|-------------|
| `subchapterId` / `nodeId` | **Must match** the `id` in `encounter.json` |
| `isUnlockByUnlockCode` | Set to `false` to always be unlocked |
| `customChapterText` | Name displayed on the node |
| `timeLine: "PLAYGROUND"` | Places the node in the Playground area (easiest to access) |
| `region: "p"` | Playground region code |
| `mapAreaId: 101` | Playground map area ID |

> **Strongly recommended: place your node in Playground.** It is always accessible regardless of story progress or unlock conditions.

---

## 4. Creating a Custom Boss Unit

Skip this section if you're using an existing game unit. For a custom boss, you need two files.

```
custom_limbus_data/
├── abnormality-unit/
│   └── vespa.json        ← Full unit configuration
└── abnormality-part/
    └── part.json         ← Body (part) stats
```

### 4-1. abnormality-unit — The Unit Body

**Yellow Harpoon's actual file (key sections):**

```json
{
  "list": [
    {
      "id": 6810001,
      "nameID": 400006,
      "appearance": "400006_VespaAppearance",
      "classType": "ALEPH",
      "attributeType": "CRIMSON",
      "startActionSlotNum": 6,
      "maxActionSlotNum": 6,
      "hp": {
        "defaultStat": 90,
        "incrementByLevel": 37
      },
      "unitKeywordList": ["SMALL", "UNIQUE_COLOR", "VESPA", "FIXER"],
      "associationList": ["LIMBUS_COMPANY"],
      "patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
      "abnormalityPartList": [6810101],
      "initBuffList": [
        {
          "level": 1,
          "list": [
            {
              "buffKeyword": "YellowHarpoonSin",
              "stack": 1,
              "turn": 99
            }
          ]
        }
      ],
      "hasMp": true,
      "panicType": 1074,
      "lowMorale": -20,
      "panic": -45,
      "passiveSet": {
        "passiveIdList": [40000901, 40000602, 40000902, 9999998, 9999999]
      },
      "attributeList": [
        {"skillId": 681001010, "number": 0},
        {"skillId": 681001020, "number": 0},
        {"skillId": 681001030, "number": 0},
        {"skillId": 681001050, "number": 0},
        {"skillId": 681001060, "number": 0},
        {"skillId": 681001070, "number": 0},
        {"skillId": 681001090, "number": 0},
        {"skillId": 681001100, "number": 0},
        {"skillId": 681001110, "number": 0},
        {"skillId": 681001120, "number": 0}
      ],
      "patternList": [ ... ]
    }
  ]
}
```

**Key field descriptions:**

| Field | Description | Yellow Harpoon Value |
|-------|-------------|----------------------|
| `id` | Unique unit ID | `6810001` |
| `nameID` | Name text ID (can reuse an existing character's nameID) | `400006` (Vespa) |
| `appearance` | Character appearance ID to use | `"400006_VespaAppearance"` |
| `classType` | Grade (`ZAYIN`, `TETH`, `HE`, `WAW`, `ALEPH`) | `"ALEPH"` |
| `attributeType` | Attribute (`CRIMSON`, `AMBER`, `SHAMROCK`, etc.) | `"CRIMSON"` |
| `startActionSlotNum` | Number of skills used per turn | `6` |
| `maxActionSlotNum` | Maximum slot count | `6` |
| `abnormalityPartList` | List of part IDs this unit owns | `[6810101]` |

> ⚠️ **Critical trap: Never put `999` in `startActionSlotNum`.**  
> The engine tries to process this literally and malfunctions. Always write an explicit number. Yellow Harpoon uses `6`.

**`initBuffList` — Buffs applied at battle start:**

```json
"initBuffList": [
  {
    "level": 1,
    "list": [
      {
        "buffKeyword": "YellowHarpoonSin",
        "stack": 1,
        "turn": 99
      }
    ]
  }
]
```

> ⚠️ **Always set `turn` to `99`.**  
> Setting `turn: 0` silently prevents the buff from being applied at all. This is known Lethe behavior.

**`attributeList` — Skills available to this unit:**

```json
"attributeList": [
  {"skillId": 681001010, "number": 0},
  {"skillId": 681001020, "number": 0}
]
```

Only skills registered here can be referenced in `patternList`. Skill IDs must match those defined in `custom_limbus_data/skill/`.

---

### 4-2. abnormality-part — Part (Body) Stats

This defines the stats of the "body" that actually receives attacks. Every unit must have at least one part.

**Yellow Harpoon's actual file:**

```json
{
  "list": [
    {
      "id": 6810101,
      "nameID": 400006,
      "hp": {
        "defaultStat": 90,
        "incrementByLevel": 35
      },
      "minSpeedList": [5],
      "maxSpeedList": [8],
      "partType": "BODY",
      "isDestroyable": "false",
      "spreadMpEffectToAbnormality": "true",
      "resistInfo": {
        "atkResistList": [
          {"type": "SLASH",      "value": 0.5},
          {"type": "PENETRATE",  "value": 1.0},
          {"type": "HIT",        "value": 1.0}
        ],
        "attributeResistList": [
          {"type": "CRIMSON",  "value": 1.0},
          {"type": "SCARLET",  "value": 1.0},
          {"type": "AMBER",    "value": 1.0},
          {"type": "SHAMROCK", "value": 1.0},
          {"type": "AZURE",    "value": 1.0},
          {"type": "INDIGO",   "value": 1.0},
          {"type": "VIOLET",   "value": 1.0},
          {"type": "WHITE",    "value": 0.0},
          {"type": "BLACK",    "value": 0.0}
        ]
      }
    }
  ]
}
```

**Resistance value reference:**

| value | Meaning |
|-------|---------|
| `0.5` | Weak (50% extra damage taken) |
| `1.0` | Normal |
| `2.0` | Resistant (50% damage reduction) |
| `0.0` | Immune |

> **The `id` here must match the number listed in the unit's `abnormalityPartList`.**  
> In Yellow Harpoon, the unit has `abnormalityPartList: [6810101]` and the part has `id: 6810101` — they must align.

---

## 5. Creating Skills

Skills are defined as a single JSON file under `custom_limbus_data/skill/`. One file can contain multiple skills.

```json
{
  "list": [
    { /* Skill 1 */ },
    { /* Skill 2 */ },
    ...
  ]
}
```

### 5-1. Understanding the Coin System

Every attack skill in Limbus Company is made up of **coins**. 1 coin = 1 hit.

```
When a 3-coin skill executes:
  coin[0] flipped → heads = attack lands → effect triggers
  coin[1] flipped → heads = attack lands → effect triggers
  coin[2] flipped → heads = attack lands → effect triggers
```

Each coin can have its own independent effects.

### 5-2. Basic Skill Structure

**4-coin Basic Slash example (681001010):**

```json
{
  "id": 681001010,
  "textID": 681001010,
  "skillType": "SKILL",
  "skillTier": 1,
  "skillData": [{
    "iconID": 40000601,
    "skillTargetType": "RANDOM",
    "canDuel": true,
    "canChangeTarget": true,
    "attributeType": "AMBER",
    "atkType": "SLASH",
    "defType": "ATTACK",
    "skillMotion": "S1",
    "targetNum": 1,
    "defaultValue": 20,
    "skillLevelCorrection": 2,
    "abilityScriptList": [
      {"scriptName": "PrimeTargetByLevelSort"},
      {"scriptName": "EmptyBody"},
      {"scriptName": "AttackDmgMultiplierByNegativeBuffCountLimit_100", "value": 0.5}
    ],
    "coinList": [
      {"operatorType": "ADD", "scale": 2, "abilityScriptList": [
        {"scriptName": "GiveBuffOnSucceedAttack",
         "buffData": {"buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 0}}
      ]},
      {"operatorType": "ADD", "scale": 2, "abilityScriptList": [
        {"scriptName": "GiveBuffOnSucceedAttack",
         "buffData": {"buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 0}}
      ]},
      {"operatorType": "ADD", "scale": 2, "abilityScriptList": [
        {"scriptName": "GiveBuffOnSucceedAttack",
         "buffData": {"buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 0}}
      ]},
      {"operatorType": "ADD", "scale": 2, "abilityScriptList": [
        {"scriptName": "GiveBuffOnSucceedAttack",
         "buffData": {"buffKeyword": "Laceration", "target": "Target", "stack": 0, "turn": 2}}
      ]}
    ]
  }]
}
```

**Skill base field descriptions:**

| Field | Description | Example |
|-------|-------------|---------|
| `id` | Unique skill ID | `681001010` |
| `skillTier` | Skill grade (1 = basic, 2 = intermediate, 3 = advanced) | `1` |
| `attributeType` | Skill attribute color | `"AMBER"` |
| `atkType` | Attack type (`SLASH`, `PENETRATE`, `HIT`) | `"SLASH"` |
| `defType` | Defense type (`ATTACK`, `GUARD`, `EVADE`, `COUNTER`) | `"ATTACK"` |
| `skillMotion` | Animation ID to play | `"S1"` |
| `targetNum` | Number of targets | `1` (single), `3` (AoE) |
| `defaultValue` | Base attack power | `20` |
| `skillLevelCorrection` | Attack power increase per level | `2` |
| `importanceLevel` | AI skill selection weight (higher = more frequent) | `1`–`10` |

### 5-3. Coin Field Descriptions

```json
{
  "operatorType": "ADD",
  "scale": 2,
  "grade": 2,
  "color": "GREY",
  "abilityScriptList": [ ... ]
}
```

| Field | Description |
|-------|-------------|
| `operatorType` | Almost always `"ADD"` |
| `scale` | Attack power multiplier for this coin |
| `grade: 2` + `color: "GREY"` | Marks this coin as Unbreakable |
| `abilityScriptList` | List of effects this coin triggers |

**Unbreakable Coin vs. SuperCoin — Two Different Concepts:**

When two units clash with attack skills in Limbus Company, a **Clash** occurs.  
The loser's coin is **destroyed** and the attack is cancelled. That is the default rule.

| Concept | JSON Setting | Effect |
|---------|-------------|--------|
| **Unbreakable Coin** | `grade: 2` + `color: "GREY"` | The coin is not destroyed even when losing a Clash. After the opponent attacks, this coin still fires |
| **SuperCoin** | `"scriptName": "SuperCoin"` | The coin flip always results in Heads. The power bonus is always applied |

The two effects are independent and are typically used together.

```json
{
  "operatorType": "ADD",
  "grade": 2,
  "color": "GREY",
  "scale": 2,
  "abilityScriptList": [
    {"scriptName": "SuperCoin"}
  ]
}
```

The example above applies both effects: the coin survives losing a Clash, and the flip always lands Heads.

### 5-4. Commonly Used abilityScripts

**Scripts applied to the whole skill (top-level `abilityScriptList`):**

```json
"abilityScriptList": [
  {"scriptName": "PrimeTargetByLevelSort"},
  {"scriptName": "EmptyBody"}
]
```

| scriptName | Effect |
|------------|--------|
| `PrimeTargetByLevelSort` | Prioritizes higher-level targets |
| `EmptyBody` | Standard abnormality attack body |
| `SetCoinResultToHeadBySkill` | Forces all coins to Heads based on skill power |
| `SupportiveDefense` | Support function for defensive skills |

**Scripts applied per coin (inside `coinList` → `abilityScriptList`):**

```json
{"scriptName": "GiveBuffOnSucceedAttack",
 "buffData": {"buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 0}}
```

| scriptName | Effect |
|------------|--------|
| `SuperCoin` | Coin flip always Heads (power bonus always applied) |
| `GiveBuffOnSucceedAttack` | Apply a buff on successful attack |
| `GiveBuffOnUse` | Apply a buff when the skill is used |
| `VibrationExplosionOnSucceedAttackAndLoseVibrationTurnByCheckSystemAbility` | Tremor explosion (`value` sets the count) |
| `ActivateTargetLacerationOnSucceedAttack` | Activate Laceration on target |
| `BreakTargetOnSucceedAttack` | Break the target |

**`buffData` fields:**

```json
"buffData": {
  "buffKeyword": "Vibration",   ← Buff type
  "target": "Target",           ← Target ("Target", "Self", "AllyExceptSelf")
  "stack": 5,                   ← Stacks to apply
  "turn": 0                     ← Duration in turns (0 = permanent)
}
```

### 5-5. Practical Example: A 10-Coin Major Skill (681001100)

Abnormality boss skills often work differently from player Identities in the Clash system.  
When a boss needs to reliably land powerful consecutive hits, it's standard to make all coins Unbreakable + SuperCoin.

```json
{
  "id": 681001100,
  "skillTier": 3,
  "skillData": [{
    "skillMotion": "S10",
    "targetNum": 1,
    "defaultValue": 16,
    "skillLevelCorrection": 7,
    "abilityScriptList": [
      {"scriptName": "PrimeTargetByLevelSort"},
      {"scriptName": "EmptyBody"},
      {"scriptName": "SetCoinResultToHeadBySkill"},
      {"scriptName": "AttackDmgUpByRatioViaBuffCheckDevideByStack",
       "buffData": {"buffKeyword": "Laceration", "buffOwner": "Target", "stack": 10, "value": 0.2}},
      {"scriptName": "SkipThisRoundOnEndAttack"}
    ],
    "coinList": [
      {"operatorType": "ADD", "grade": 2, "color": "GREY", "scale": 2, "abilityScriptList": [
        {"scriptName": "SuperCoin"},
        {"scriptName": "ActivateTargetLacerationOnSucceedAttack", "value": 1}
      ]},
      {"operatorType": "ADD", "grade": 2, "color": "GREY", "scale": 2, "abilityScriptList": [
        {"scriptName": "SuperCoin"},
        {"scriptName": "ActivateTargetLacerationOnSucceedAttack", "value": 1}
      ]},
      /* ... coin[2]–coin[8] same structure ... */
      {"operatorType": "ADD", "grade": 2, "color": "GREY", "scale": 2, "abilityScriptList": [
        {"scriptName": "SuperCoin"},
        {"scriptName": "BreakTargetOnSucceedAttack"}
      ]}
    ]
  }]
}
```

> **Animation note:** When building a 10-coin skill with `skillMotion: "S10"`, if the animation only has 5 frames, coins coin[5]–coin[9] will all repeat the last frame. To fix this, see the `changemotion()` guide in Chapter 3.

---

## 6. Creating Custom Buffs

Rather than limiting yourself to the game's built-in buffs (Tremor, Laceration, etc.), you can create entirely new custom buffs.  
A custom buff requires **three types of files**.

```
MyMod/
├── custom_buffs/
│   └── MyBuff.txt                        ← Buff ID registration (can be empty)
├── custom_limbus_data/
│   └── buff/
│       └── mybuff.json                   ← Buff properties and behavior
└── custom_limbus_locale/
    └── KR/                               ← Change folder name to match your language (KR, EN, etc.)
        ├── bufList/
        │   └── mybuff_locale.json        ← Buff name
        └── keywordList/
            └── mybuff_keyword.json       ← Buff tooltip description
```

### 6-1. custom_buffs/*.txt — Buff ID Registration

Create a `.txt` file inside `custom_buffs/` with the same name as your buff ID.  
**The file can be empty.** Its existence alone registers that ID as a valid buff.

```
custom_buffs/
└── YellowHarpoonSin.txt    ← The existence of this file makes "YellowHarpoonSin" a valid buff ID
```

> **Note:** The game must be restarted for a new buff ID to be recognized.  
> Using the same name as an existing game buff will overwrite it (see Chapter 1 on ID collision).

### 6-2. custom_limbus_data/buff/*.json — Buff Property Definition

This file defines the buff's properties and behaviors (effects).

**Basic template:**

```json
{
  "list": [
    {
      "id": "YellowHarpoonSin",
      "iconId": "Nail",
      "buffClass": "NonvolatileBuff",
      "buffType": "Positive",
      "canBeDespelled": false,
      "list": [],
      "categoryKeywordList": []
    }
  ]
}
```

**Key field descriptions:**

| Field | Description |
|-------|-------------|
| `id` | Unique buff ID. **Must exactly match** the `.txt` filename in `custom_buffs/` |
| `iconId` | Icon ID to display. You can use an existing buff's ID to borrow its icon |
| `buffClass` | Structural form of the buff (see table below) |
| `buffType` | `"Positive"`, `"Negative"`, or `"Neutral"` — affects scripts like "remove random debuff" |
| `canBeDespelled` | If `false`, cannot be forcibly removed |
| `list` | Array of behavior scripts that define what the buff actually does |
| `categoryKeywordList` | Which status condition family this buff belongs to (`"COMBUSTION"`, `"LACERATION"`, etc.) |

**`buffClass` types:**

| buffClass | Description |
|-----------|-------------|
| `NonvolatileBuff` | Has potency only. Does not expire automatically at turn end |
| `VolatileBuff` | Has potency only. Expires automatically at turn end |
| `CountableBuff` | Has both potency and count |
| `sinBuff` | Similar to `CountableBuff`. Used by the 7 major status conditions (Tremor, Laceration, etc.) |

**Optional additional fields:**

```json
{
  "id": "YellowHarpoonSin",
  "maxStack": 20,
  "maxTurn": 10,
  "attributeType": "AMBER",
  "destroyableOnZero": true,
  "destroyableOnZeroTurn": true,
  "destroyableOnRetreat": false
}
```

| Field | Description | Default |
|-------|-------------|---------|
| `maxStack` | Maximum potency | 99 |
| `maxTurn` | Maximum count | 99 |
| `attributeType` | Attribute used by `sinBuff` (e.g., `AMBER`, `AZURE`) | — |
| `destroyableOnZero` | Automatically removed when potency reaches 0 | — |
| `destroyableOnZeroTurn` | Automatically removed when count reaches 0 | — |
| `destroyableOnRetreat` | Removed when the unit retreats | — |

### 6-3. list — The Buff's Actual Behavior Scripts

The `list` array inside the buff JSON defines **what the buff actually does**.  
Check `dumpedData/limbus_data/buff/` for a wide range of patterns from existing buffs.

**Behavior script example — apply Laceration at round start:**

```json
"list": [
  {
    "ability": "GiveBuffOnRoundStart",
    "value": 1,
    "buffData": {
      "buffKeyword": "Laceration",
      "target": "Self",
      "stack": 1,
      "turn": -1,
      "activeRound": 0
    }
  },
  {
    "ability": "LoseStackOnRoundEndByRatio",
    "value": 0.5
  }
]
```

This replicates the behavior of the built-in `Nail` buff:
- `GiveBuffOnRoundStart`: applies 1 stack of Laceration at round start
- `LoseStackOnRoundEndByRatio`: reduces potency by 50% at round end

Find more `ability` script types by browsing existing buffs in `dumpedData/limbus_data/buff/`.

### 6-4. Custom Buff Icons

Place a `buffIcon_{ID}.png` file in the `custom_sprites/` folder to use a custom icon.

```
custom_sprites/
└── buffIcon_YellowHarpoonSin.png
```

Then in the buff JSON:
```json
"iconId": "YellowHarpoonSin"
```

### 6-5. Buff Locale (Name and Tooltip Text)

**bufList — Buff name:**

```
custom_limbus_locale/KR/bufList/mybuff_locale.json
```

```json
{
  "dataList": [
    {
      "id": "YellowHarpoonSin",
      "name": "Sin of the Yellow Harpoon"
    }
  ]
}
```

**keywordList — Buff tooltip description:**

```
custom_limbus_locale/KR/keywordList/mybuff_keyword.json
```

```json
{
  "dataList": [
    {
      "id": "YellowHarpoonSin",
      "desc": "Imbued with the energy of the Yellow Harpoon. Certain skills are empowered while this is held."
    }
  ]
}
```

> **Note:** The locale folder name (`KR/`) should match the language you are writing for. Use `KR` for Korean or `EN` for English.

### 6-6. Applying a Custom Buff in a Skill

Custom buffs are applied exactly the same way as built-in buffs.

```json
{"scriptName": "GiveBuffOnSucceedAttack",
 "buffData": {
   "buffKeyword": "YellowHarpoonSin",
   "target": "Target",
   "stack": 3,
   "turn": 2
 }}
```

To apply it as a starting buff on the unit, add it to `initBuffList` in `abnormality-unit` (see Section 4-1).

---

## 7. Pattern Design

Patterns determine **which skills the boss uses, in what order, on each turn**.

### 7-1. patternList Structure

The `patternList` in `abnormality-unit` is an array that cycles through one entry per turn.

```json
"patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
"patternList": [
  { /* Turn 1 pattern */ },
  { /* Turn 2 pattern */ },
  { /* Turn 3 pattern */ },
  ...
]
```

When `patternID` is `PickByPattern_Abnormality_UptoActionSlotCnt`:
- `patternList[0]` → used on turn 1
- `patternList[1]` → used on turn 2
- After `patternList[N-1]` → wraps back to `patternList[0]`

### 7-2. Structure of a Single Pattern

One pattern entry is the list of skill slots used that turn.

```json
{
  "slotList": [
    {"skillParentList": [{"skillChildList": [{"skillID": 681001060, "chance": 1}], "chance": 1}]},
    {"skillParentList": [{"skillChildList": [{"skillID": 681001090, "chance": 1}], "chance": 1}]},
    {"skillParentList": [{"skillChildList": [{"skillID": 681001010, "chance": 1}], "chance": 1}]},
    {"skillParentList": [{"skillChildList": [{"skillID": 681001100, "chance": 1}], "chance": 1}]},
    {"skillParentList": [{"skillChildList": [{"skillID": 681001030, "chance": 1}], "chance": 1}]},
    {"skillParentList": [{"skillChildList": [{"skillID": 681001110, "chance": 1}], "chance": 1}]}
  ]
}
```

The length of `slotList` = the number of skills used that turn = should match `startActionSlotNum`.

A single slot written compactly:

```json
{"skillParentList": [{"skillChildList": [{"skillID": 681001060, "chance": 1}], "chance": 1}]}
```

### 7-3. Yellow Harpoon's Pattern Design Philosophy

Yellow Harpoon alternates **Phase 1 (P1) skills** and **Phase 2 (P2) skills** across slots each turn.

```
P1 skills: 010, 020, 030, 050, 060, 070
P2 skills: 090, 100, 110, 120
```

Example slot assignments per turn:

```
Turn 1: [P1:060] [P2:090] [P1:010] [P2:100] [P1:030] [P2:110]
Turn 2: [P1:060] [P2:070] [P1:020] [P2:120] [P1:050] [P2:090]
...
```

Mixing attack and defensive/utility skills keeps boss behavior from becoming monotonous.

**Full 12-pattern cycle structure (excerpt):**

```json
"patternList": [
  {
    "slotList": [
      {"skillParentList": [{"skillChildList": [{"skillID": 681001060, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001090, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001010, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001100, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001030, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001110, "chance": 1}], "chance": 1}]}
    ]
  },
  {
    "slotList": [
      {"skillParentList": [{"skillChildList": [{"skillID": 681001060, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001070, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001020, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001120, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001050, "chance": 1}], "chance": 1}]},
      {"skillParentList": [{"skillChildList": [{"skillID": 681001090, "chance": 1}], "chance": 1}]}
    ]
  }
  /* ... 12 patterns total ... */
]
```

### 7-4. Pattern Design Tips

- **Match slot count to `startActionSlotNum`.** Extra slots beyond that number are ignored.
- **Put skills you always want to appear in the same slot position across all patterns.** In Yellow Harpoon, `681001060` (a counter skill) is in slot 0 of almost every pattern.
- **Skills with high `importanceLevel` get priority in out-of-pattern selection.** Combined with a pattern, this reinforces the intended behavior even more strongly.

---

## 8. Testing and Debugging

### 8-1. Applying Your Mod

1. Copy the completed mod folder to:
   ```
   BepInEx/plugins/Lethe/mods/MyModName/
   ```
2. Launch the game using LetheLauncher.
3. Go to the stage select screen and look for the **Playground** area.
4. If your custom battle node appears, it worked.

> File changes made while the game is running are not reflected. Always **restart the game** after making changes.

### 8-2. Checking Errors with the Log

When something goes wrong, open:

```
BepInEx/LogOutput.log
```

Lethe-related errors begin with `[Lethe]` or `[Warning: Lethe]`.

```
[Info   : Lethe] loading file from \YellowHarpoon\custom_limbus_data\skill\vespa_skills.json
[Warning: Lethe] Skill ID 681001010 not found in attributeList
```

### 8-3. Common Mistakes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Stage node doesn't appear | `subchapterId`/`nodeId` doesn't match `encounter.json`'s `id` | Make all three values identical |
| Boss uses only 1–2 skills | `startActionSlotNum` set to `999` | Change to the desired number (e.g., `6`) |
| Starting buff not applied | `initBuffList` has `turn: 0` | Change to `turn: 99` |
| Skill never appears in patterns | Skill ID not registered in `attributeList` | Add it to `attributeList` |
| Error about missing part | `abnormalityPartList` ID doesn't match part `id` | Align the IDs |
| JSON parse error | Missing comma, bracket mismatch | Validate JSON in VS Code |

---

## 9. Going Further

Once the basic encounter is working, you can add these features:

### 9-1. Adding Custom Names

Add a JSON file in `custom_limbus_locale/KR/enemyList/` to assign a custom display name to your boss.

```json
{
  "dataList": [
    {
      "id": 6810001,
      "name": "Vespa, Yellow Harpoon"
    }
  ]
}
```

### 9-2. Multi-Wave Configuration

Add additional objects to `waveList` in `encounter.json`. From wave 2 onwards, leave `mapName` as an empty string to keep the previous map.

```json
"waveList": [
  {
    "battleMapInfo": {"mapName": "Cp9_Middlefinger", "mapSize": 33.0},
    "unitList": [{"unitID": 6810001, "unitCount": 1, "unitLevel": 87}],
    "bgmList": ["Battle_Cp9_Boss_3_2"]
  },
  {
    "battleMapInfo": {"mapName": "", "mapSize": 33.0},
    "unitList": [{"unitID": 6810001, "unitCount": 2, "unitLevel": 87}],
    "bgmList": ["Battle_Cp9_Boss_3_2"]
  }
]
```

### 9-3. Custom Skill Icons

Place a `skill_[skillID].png` file in `custom_sprites/` to replace the skill's icon.

```
custom_sprites/
└── skill_681001010.png
```

### 9-4. Advanced Lua Features

If you need any of the following, see the dedicated Lua guide in Chapter 3:

- Specifying animation frames per coin (`changemotion`)
- Phase transitions based on HP threshold
- Replacing skill slots under specific conditions
- Custom buff value calculations

---

## Appendix: Recommended ID Scheme

To prevent collisions, custom IDs should follow these guidelines:

| Item | Recommended Range | Yellow Harpoon Example |
|------|------------------|------------------------|
| encounter ID | 600,000+ | `681001` |
| Unit ID | 6,000,000+ | `6810001` |
| Part ID | Unit ID + 100 | `6810101` |
| Skill ID | Unit ID × 10 + number | `681001010`, `681001020` |

Space skill numbers in increments of 10 (`010`, `020`, `030`, ...) so you can insert new skills between existing ones later.
