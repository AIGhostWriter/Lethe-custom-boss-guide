# Chapter 4 — Full Sample: Soji Abi (소지아비)

> A complete boss mod walkthrough. Every file shown is the actual file used.

*Previous: [Chapter 3 — Advanced Lua](./chapter3_EN.md)*

---

## Table of Contents

1. [Concept and ID Scheme](#1-concept-and-id-scheme)
2. [Mod Folder Structure](#2-mod-folder-structure)
3. [Registering the Encounter](#3-registering-the-encounter)
4. [Boss Unit](#4-boss-unit)
5. [Skills](#5-skills)
6. [Custom Passive](#6-custom-passive)
7. [Pattern Design](#7-pattern-design)
8. [Lua — 광역 난사 (Area Barrage)](#8-lua--광역-난사-area-barrage)

---

## 1. Concept and ID Scheme

Soji Abi is a 7-slot sword-fighter boss built around two keywords:

| Keyword | Applied to | Role |
|---|---|---|
| `Breath` (호흡) | Self | Power resource — accumulates every hit |
| `PhantomIncision` (잔상) | Target | Mark for follow-up damage |

The climax skill **광역 난사** (682001200) fires 9 coins in a row, each exploding and reloading Vibration stacks.

**ID scheme:**

```
encounter:   682001
unit:        6820001
part:        6820101
passive:     6820901
skills:      682001010 … 682001200   (steps of 10)
```

---

## 2. Mod Folder Structure

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

---

## 3. Registering the Encounter

### encounter.json

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

| Field | Notes |
|---|---|
| `id` | Must match `subchapterId` and `nodeId` in `subchapterui.json` |
| `stageType` | `"Abnormality"` for a boss fight |
| `mapName` | Spider House rooftop — picked from existing map names in `dumpedData/encounters/` |
| `bgmList` | Two tracks — the second activates mid-fight |
| `allyPositionID` / `enemyPositionID` | Spawn positions; values copied from a similar existing encounter |
| `staminaCost: 0` | Free to enter — standard for sandbox mods |

### subchapterui.json

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

`subchapterId` and `nodeId` must both equal `encounter.json`'s `id`. `timeLine: "PLAYGROUND"` places the node in the Playground area, which is always accessible.

---

## 4. Boss Unit

### abnormality-unit — soji.json (key sections)

```json
{
  "list": [{
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
  }]
}
```

| Field | Notes |
|---|---|
| `appearance` | Copied from `dumpedData/identities/` for character 1322 |
| `unitScriptID` | AI script — copied from a reference boss using the same rig |
| `startActionSlotNum` | Skills per turn. Never set to `999` — always an explicit number |
| `abnormalityPartList` | Must match the `id` in `soji_part.json` |
| `attributeList` | Every skill the boss can use must be listed here. `"number": 0` is correct for boss units |
| `passiveSet` | IDs `132219`–`132230` and `9999998` are copied from the reference character. `6820901` is the custom passive |
| `mentalConditionInfo` | Controls morale gain/loss on clash win/loss — copied from a similar existing boss |

> The passive IDs `132219`–`132230` and `9999998` are base-game passives. They require no custom files — they already exist in the game data.

### abnormality-part — soji_part.json

```json
{
  "list": [{
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
  }]
}
```

The part `id` (`6820101`) must match `abnormalityPartList` in `soji.json`. Resistance values: `< 1.0` = weak, `1.0` = normal, `> 1.0` = resistant.

### Locale — soji_locale.json

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

The top-level key is the unit ID as a string. This format is specific to `enemyList` — do not use the `dataList` array format used for buffs.

---

## 5. Skills

All 15 skills are in `custom_limbus_data/skill/soji_skills.json`.

### 5-1. Skill List

| ID | Name | Coins | Sin | Key mechanic |
|---|---|---|---|---|
| 682001010 | 연격 | 4 | VIOLET | Breath per hit; AoE 5 targets |
| 682001020 | 이연잔 | 5 | VIOLET | Breath per hit; anchor skill in every pattern |
| 682001030 | 잔상베기 | 1 | INDIGO | SuperCoin; Breath +5 if one-sided; PhantomIncision |
| 682001040 | 폭압격 | — | INDIGO | Forced clash; target count reacts to duel result |
| 682001060 | 잔상 해방 | — | INDIGO | PhantomIncision burst |
| 682001080 | 사냥 | — | INDIGO | — |
| 682001100 | 처형 | — | INDIGO | Heavy execution |
| 682001110 | 공진 | — | INDIGO | Vibration |
| 682001130 | 이연속참 | — | VIOLET | Breath + chain |
| 682001140 | 필연쇄 | — | INDIGO | — |
| 682001160 | 호흡 강탈 | — | INDIGO | Breath drain |
| 682001170 | 정신 압박 | — | INDIGO | — |
| 682001180 | 취약 유도 | — | INDIGO | — |
| 682001190 | 취약 강타 | — | INDIGO | — |
| 682001200 | 광역 난사 | 9 | VIOLET | VibrationExplosion + Vibration 5/5 on all 9 coins |

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
      { /* coin[1] — identical to coin[0] */ },
      { /* coin[2] — identical to coin[0] */ },
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

**Top-level `abilityScriptList`** — fires when the skill is selected, before any coins:

- `EmptyBody` — required on every boss skill. Declares a boss-type attack body.
- `GiveBuffOnUse` with `stack: 0, turn: 2` — `stack: 0` does **not** add stacks. It refreshes the Breath buff's remaining duration to 2 turns without changing the current count. This keeps Breath alive across rounds.

**Coins:**

| Coin | `scale` | Effect |
|---|---|---|
| [0] [1] [2] | 4 | Breath +1; plays S1 frame 0 |
| [3] | 7 | Breath +2; plays S1 frame 1 (finishing animation) |

`viewType: "ENCOUNTER"` zooms the camera out to show all units — use this for AoE skills. Use `"BATTLE"` for single-target clashes.

### 5-3. Conditional Scripts — 잔상베기 (682001030)

```json
{
  "id": 682001030,
  "skillTier": 2,
  "skillData": [{
    "attributeType": "INDIGO",
    "defaultValue": 14,
    "skillLevelCorrection": 3,
    "skillMotion": "S3",
    "viewType": "ENCOUNTER",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      {
        "scriptName": "GiveBuffBeforeAttackIfOnesideAttack",
        "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 5, "turn": 0 }
      },
      { "scriptName": "TagetNumAdderOnLoseDuel", "value": -2 }
    ],
    "coinList": [
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
    ]
  }]
}
```

**Top-level scripts:**

| Script | When it fires | Effect |
|---|---|---|
| `GiveBuffBeforeAttackIfOnesideAttack` | Before attack, only if no clash | Grants Breath +5. Fires only when the player left this slot uncontested |
| `TagetNumAdderOnLoseDuel` `value: -2` | After losing a duel | Target count −2. Rewards the player for contesting |

**The single coin:**

```
grade: 2 + color: "GREY"        → Unbreakable — fires even when the player wins the clash
SuperCoin                        → Coin flip always Heads — full power always applies
CriticalDmgUpByJsonValue 0.3     → +30% critical hit damage
GiveBuffOnSucceedAttack ×2       → 2 separate entries = 2 PhantomIncision stacks applied sequentially
```

### 5-4. abilityScript Reference

**Top-level (fires on skill selection):**

| Script | Effect |
|---|---|
| `EmptyBody` | Required on every boss skill |
| `GiveBuffOnUse` | Apply buff on selection. `stack: 0` = duration refresh only |
| `GiveBuffBeforeAttackIfOnesideAttack` | Buff granted before attack if no clash occurs |
| `TagetNumAdderOnLoseDuel` | Adjusts target count when boss loses a duel |

**Per-coin (fires on each coin):**

| Script | Effect |
|---|---|
| `SuperCoin` | Coin flip always Heads |
| `GiveBuffOnSucceedAttack` | Apply buff on successful hit |
| `VibrationExplosionOnSucceedAttack` | Detonate current Vibration stacks on target |
| `CriticalDmgUpByJsonValue` | Multiply critical damage by `1 + value` |

### 5-5. Skill Locale — soji_skills_locale.json

```json
{
  "682001030": {
    "id": "682001030",
    "levelList": [{
      "level": 1,
      "name": "잔상베기",
      "desc": "[일방적 공격 시] 호흡 5 획득. [결투 패배 시] 공격 대상 수 -2.",
      "coinlist": [{
        "coindescs": [
          { "desc": "[파괴불가] 치명타 피해 +30%, [적중시] 잔상 1 부여, [결투 승리 시] 잔상 1 추가 부여", "summary": "" }
        ]
      }]
    }]
  }
}
```

`coindescs` must have **exactly as many entries as the skill has coins**. 잔상베기 has 1 coin → 1 entry. 광역 난사 has 9 coins → 9 entries.

---

## 6. Custom Passive

`custom_limbus_data/passive/SojiPassive.json`:

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

This passive has no stat effects. Its only role is to fire the Lua pattern controller once per round.

**`requireIDList` string breakdown:**

```
Modular / TIMING:AfterSlots / LUA:SojiPattern / LUAMAIN:onSkillUse
  │              │                  │                    │
  │              │                  │                    └─ Function to call
  │              │                  └─ modular_lua/SojiPattern.lua
  │              └─ Fires after slots are assigned, before skills execute
  └─ ModularSkillScripts system
```

`AfterSlots` is the only timing where `skillslotreplace()` is effective. Skills are assigned but not yet executing — slots can still be overridden.

The passive is activated by listing its ID in `passiveSet` inside `soji.json`:

```json
"passiveSet": {
  "passiveIdList": [132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998, 6820901]
}
```

---

## 7. Pattern Design

### patternList Structure

Each entry in `patternList` defines one round of skills (7 slots):

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
  }
  /* 5 more patterns ... */
]
```

A single slot written compactly:

```json
{"skillParentList": [{"skillChildList": [{"skillID": 682001200, "chance": 1}], "chance": 1}]}
```

The nested structure supports weighted random selection per slot. `chance: 1` makes it deterministic.

### Lua Override

The `patternList` in this mod acts as a **fallback only**. The Lua script `SojiPattern.lua` overrides all 7 slots every round at `AfterSlots` timing using a 3-cycle rotation:

| Cycle | Rounds | Focus | Slot 0 |
|---|---|---|---|
| Cycle 1 | 1, 4, 7… | Area barrage + heavy pressure | 682001200 always |
| Cycle 2 | 2, 5, 8… | Breath accumulation | 682001020 / 682001140 |
| Cycle 3 | 3, 6, 9… | PhantomIncision pressure | 682001110 / 682001040 |

All `patternList` entries have 682001200 in slot 0 as a safe default in case the Lua override does not run.

---

## 8. Lua — 광역 난사 (Area Barrage)

### 8-1. Why Lua Is Needed Here

`광역 난사` (682001200) has 9 coins. The S2 animation has 3 frames. Coins 0–6 should play a random frame each hit. JSON has no `random()` function — Lua is required.

### 8-2. SojiBarrage.lua

```lua
-- SojiBarrage.lua
-- Picks a random S2 frame (0, 1, or 2) for each hit on coins 0–6.
-- Called at TIMING:ChangeMotion — fires for all coins including SuperCoins.

function barrageRandomS2()
    local frame = random(0, 2)
    changemotion("S2", frame)
    log("[SojiBarrage] S2[" .. frame .. "]", 0)
end
```

> `changemotion(motionName, frameIndex)` — changes the animation frame playing for this coin. Only valid at `TIMING:ChangeMotion`.  
> `random(0, 2)` — returns 0, 1, or 2 with equal probability (inclusive on both ends).

### 8-3. Wiring Lua to Each Coin

**Coins 0–6** call `barrageRandomS2` for a random S2 frame:

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

**Coin 7** switches to a fixed S6 frame:

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

**Coin 8** (final blow) — S3 frame, plus Breath reward for the boss:

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

### 8-4. The Vibration Chain

Every coin runs the same sequence:

```
① SuperCoin                      → flip bypassed, attack resolves
② ChangeMotion (Lua)             → animation frame assigned
③ VibrationExplosionOnSucceedAttack → detonate current Vibration stacks on target
④ GiveBuffOnSucceedAttack: Vibration 5/5 → reload 5 stacks for the next coin
```

Explosion (③) fires before reload (④). The chain across all 9 coins:

```
Coin 0: Explosion (0 stacks) → Vibration 5 applied
Coin 1: Explosion (5 stacks) → Vibration 5 applied
Coin 2: Explosion (5 stacks) → Vibration 5 applied
...
Coin 7: Explosion (5 stacks), S6[0] → Vibration 5 applied
Coin 8: Explosion (5 stacks), S3[0] → Vibration 5 applied + Breath 3 to self
```

From coin 1 onward, every explosion fires at maximum stacks.

### 8-5. TIMING Note

All 9 coins are SuperCoins. SuperCoin bypasses the coin flip, so `TIMING:OnCoinToss` never fires on this skill. Animation logic must use `TIMING:ChangeMotion`, which fires for all coins regardless of flip status.

---

*For the full Lua function reference and TIMING keyword list, see [Chapter 3](./chapter3_EN.md).*
