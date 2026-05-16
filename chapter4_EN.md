# Chapter 4 — Complete Sample: Soji Abi (소지아비)

> **This chapter documents an entire boss mod from scratch — every design decision, file, and mistake, exactly as it happened.**

*Previous: [Chapter 3 — Advanced Lua](./chapter3_EN.md)*

---

## Table of Contents

1. [Planning Stage](#1-planning-stage)
2. [Data Extraction from dumpedData](#2-data-extraction-from-dumpeddata)
3. [Full Folder Tree](#3-full-folder-tree)
4. [Passives](#4-passives)
5. [Skills — Deep Dive](#5-skills--deep-dive)
6. [Advanced Lua — Three Scripts](#6-advanced-lua--three-scripts)
7. [Common Mistakes Made During Development](#7-common-mistakes-made-during-development)

---

## 1. Planning Stage

Before a single file is created, spend time designing the boss on paper. Vague plans lead to mid-development redesigns, wasted files, and ID collisions.

### 1-1. Boss Identity

The boss is **Soji Abi (소지아비)** — a sword-fighter who accumulates momentum with every strike and unleashes it in a massive multi-hit barrage. Every mechanic in the kit exists to support a single fantasy: a boss that feels more dangerous the longer the fight goes on.

| Property | Decision | Reason |
|---|---|---|
| **Model** | LeiHeng (`1322_Pinky_Father2pAppearance`) | Sword animations fit a flowing, rhythmic fighting style |
| **Attribute** | INDIGO | Calm-but-relentless feel; distinct from typical CRIMSON boss color |
| **Slots per turn** | 7 | Enough pressure without losing pattern variety |
| **Skill count** | 15 | Sufficient for a 3-cycle rotation without repetition |
| **Stage level** | 77 | Upper mid-game difficulty |

### 1-2. Core Mechanic Design: Two Keywords

The whole kit revolves around exactly two custom keywords:

| Keyword | Korean | Type | Applied To | Purpose |
|---|---|---|---|---|
| `Breath` | 호흡 | Stacking buff | Self | Power resource. Accumulates every hit, empowers later skills |
| `PhantomIncision` | 잔상 | Stacking debuff | Target | Marks the target for follow-up damage |

This creates a two-phase rhythm:

```
Phase A — Breath accumulation  (연격, 이연잔, 잔상베기)
         ↓
Phase B — Payoff attacks using accumulated Breath
         ↓
Climax  — 광역 난사: 9-coin barrage that detonates Vibration on every hit
```

### 1-3. Skill Budget: 15 Skills in Three Roles

| Role | Skills | IDs |
|---|---|---|
| Breath builders | 연격, 이연잔 | 682001010, 682001020 |
| Afterimage openers | 잔상베기, 잔상 해방 | 682001030, 682001060 |
| Heavy attacks | 폭압격, 사냥, 처형, 공진 | 682001040, 682001080, 682001100, 682001110 |
| Chain skills | 이연속참, 필연쇄 | 682001130, 682001140 |
| Debuff/utility | 호흡 강탈, 정신 압박, 취약 유도, 취약 강타 | 682001160–682001190 |
| Climax | 광역 난사 | 682001200 |

### 1-4. Pattern Philosophy: Three Cycles

Rather than a simple linear pattern, the rotation is divided into **three repeating cycles**:

| Cycle | Round offset | Focus | Signature skill |
|---|---|---|---|
| Cycle 1 | 1, 4, 7, 10… | 광역 난사 + heavy pressure | 682001200 always in slot 0 |
| Cycle 2 | 2, 5, 8, 11… | Breath accumulation | 682001010/020/130 front-loaded |
| Cycle 3 | 3, 6, 9, 12… | PhantomIncision pressure | 682001030 repeated |

Within each cycle, 2–3 specific pattern variants are defined and one is randomly selected each turn. This means the same cycle never feels identical twice.

This cycle system is implemented entirely in Lua and cannot be done in JSON alone. See [Section 6-3](#6-3-sojipatternlua--the-3-cycle-pattern-controller).

### 1-5. ID Scheme

All IDs follow the `682` block to avoid conflicts:

```
encounter:   682001
unit:        6820001
part:        6820101   (unit + 100)
passive:     6820901   (unit + 900)
skills:      682001010, 682001020, … 682001200   (steps of 10)
```

Verify each ID is unused by searching `dumpedData/` with VS Code global search before writing any files.

---

## 2. Data Extraction from dumpedData

### 2-1. Finding the Appearance String

The boss uses LeiHeng's model. Search `dumpedData/identities/` for `"1322"`:

```
VS Code: Ctrl+Shift+F
Search path: dumpedData/identities/
Search term: "1322"
```

This locates the reference character file. The key value is:

```json
"appearance": "1322_Pinky_Father2pAppearance"
```

Copy this string exactly into the boss unit's `appearance` field.

### 2-2. Extracting Passive IDs from the Reference Character

Open the reference character file and find `passiveSet`:

```json
"passiveSet": {
  "passiveIdList": [132219, 132224, 132220, 132222, 132223, 132214, 132230, 9999998]
}
```

These are base-game passives that provide clash bonuses and morale behaviors. **Copy this list directly** into your custom boss unit — no additional files are needed because these IDs already exist in the base game.

The universal abnormality passive `9999998` should always be included. One custom ID (`6820901`) is appended at the end and requires its own file. See [Section 4](#4-passives).

### 2-3. Extracting Skill Motion Names

For each skill the reference character has, open the corresponding file in `dumpedData/skills/`:

```
dumpedData/skills/
├── (skill id 1).json   → skillMotion: "S1", coinList length: 2
├── (skill id 2).json   → skillMotion: "S2", coinList length: 3
├── (skill id 3).json   → skillMotion: "S3", coinList length: 1
├── (skill id 6).json   → skillMotion: "S6", coinList length: 1
```

Each file contains:

```json
{
  "skillData": [{
    "skillMotion": "S1",
    "coinList": [
      { /* coin 0 */ },
      { /* coin 1 */ }
    ]
  }]
}
```

**Record the natural coin count for each motion.** This is the number of unique animation frames the motion has. If you build a skill with more coins than frames, excess coins repeat the last frame — which looks unnatural. The Lua animation scripts in this mod fix this. See [Section 6-1](#6-1-sojimotionlua--per-coin-direct-frame-control).

Soji Abi's motion usage:

| Motion | Natural frames | Used in skills |
|---|---|---|
| `S1` | 2 | 682001010 (4 coins → frame 0×3, frame 1×1) |
| `S2` | 3 | 682001020 (5 coins → rotating frames), 682001200 barrage |
| `S3` | 1 | 682001030, 682001200 coin[8] |
| `S6` | 1 | 682001200 coin[7] |

### 2-4. Referencing a Similar Existing Boss

Before writing the abnormality-unit file, find a structurally similar boss in:

```
dumpedData/limbus_data/abnormality-unit/
```

This provides correct values for fields like `mentalConditionInfo`, `panicType`, `specialDuelRank`, and `patternID` without guessing.

---

## 3. Full Folder Tree

The completed mod contains **12 files** across 7 directories:

```
BepInEx/plugins/Lethe/mods/
└── 소지아비로/
    ├── custom_encounters/
    │   └── 소지아비로/
    │       ├── encounter.json              ← Stage registration
    │       └── subchapterui.json           ← Map node (Playground)
    ├── custom_limbus_data/
    │   ├── abnormality-unit/
    │   │   └── soji.json                   ← Boss: stats, passives, patternList
    │   ├── abnormality-part/
    │   │   └── soji_part.json              ← Boss body: HP, resistances
    │   ├── passive/
    │   │   └── SojiPassive.json            ← Custom passive (Lua trigger)
    │   └── skill/
    │       └── soji_skills.json            ← All 15 skills
    ├── custom_limbus_locale/
    │   └── KR/
    │       ├── enemyList/
    │       │   └── soji_locale.json        ← Boss display name
    │       └── skillList/
    │           └── soji_skills_locale.json ← All 15 skill descriptions
    └── modular_lua/
        ├── SojiMotion.lua                  ← Per-coin animation frame control
        ├── SojiBarrage.lua                 ← Barrage random frame selection
        └── SojiPattern.lua                 ← 3-cycle pattern controller
```

### 3-1. encounter.json

```json
{
    "id": 682001,
    "stageLevel": 77,
    "stageType": "Abnormality",
    "isBatonPassOn": true,
    "participantInfo": { "min": 1, "max": 12 },
    "battleCameraInfo": {
        "defaultPosX": 0.0, "defaultPosY": 0.0, "defaultPosZ": 40.0
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

**Field breakdown:**

| Field | Value | Notes |
|---|---|---|
| `id` | `682001` | Must match `subchapterId`/`nodeId` in subchapterui.json |
| `stageType` | `"Abnormality"` | Boss-type encounter |
| `mapName` | `"Cp9_HouseSpidersRoooftop"` | Spider House rooftop — fits the character's lore |
| `bgmList` | Two tracks | The second track activates at a point in the fight |
| `allyPositionID` / `enemyPositionID` | `175` / `164` | Controls spawn positions; find values from existing encounter data |
| `effectiveResistCondition` | `1.2` | EGO passive activation resist threshold |

### 3-2. subchapterui.json

```json
{
    "chapterId": 2,
    "subchapterId": 682001,
    "nodeId": 682001,
    "isUnlockByUnlockCode": false,
    "unlockCode": 101,
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

`subchapterId` and `nodeId` **must both equal** `encounter.json`'s `id`. Setting `timeLine: "PLAYGROUND"` places the node in the Playground area, which is always accessible.

### 3-3. soji_part.json — Boss Body Resistances

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
          {"type": "SLASH",     "value": 1.2},
          {"type": "PENETRATE", "value": 1.0},
          {"type": "HIT",       "value": 1.0}
        ],
        "attributeResistList": [
          {"type": "CRIMSON",  "value": 1.2},
          {"type": "SCARLET",  "value": 1.0},
          {"type": "AMBER",    "value": 0.8},
          {"type": "SHAMROCK", "value": 0.8},
          {"type": "AZURE",    "value": 1.2},
          {"type": "INDIGO",   "value": 0.8},
          {"type": "VIOLET",   "value": 1.0},
          {"type": "WHITE",    "value": 2.0},
          {"type": "BLACK",    "value": 2.0}
        ]
      },
      "passiveSet": { "passiveIdList": [132205] }
    }
  ]
}
```

The part's `id` (`6820101`) must match the number in the unit's `abnormalityPartList`. The resistance profile reflects a swordsman who is ironically *weak* to slash attacks (×1.2 incoming = takes more damage) but resistant to AMBER and SHAMROCK (×0.8). The `WHITE`/`BLACK` immunities (×2.0) are common for humanoid bosses.

### 3-4. Locale Files

**enemyList/soji_locale.json** — Boss display name:

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

The top-level key is the **unit ID as a string**. Do not use a `dataList` array — that format is for buffs, not enemy names.

**skillList/soji_skills_locale.json** — Skill descriptions:

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
          {"desc": "[적중시] 호흡 1 획득", "summary": ""},
          {"desc": "[적중시] 호흡 1 획득", "summary": ""},
          {"desc": "[적중시] 호흡 1 획득", "summary": ""},
          {"desc": "[적중시] 호흡 2 획득", "summary": ""}
        ]
      }]
    }]
  }
}
```

The `coindescs` array must have **exactly as many entries as the skill has coins**. 연격 has 4 coins → 4 `coindescs`. 광역 난사 has 9 coins → 9 `coindescs`.

---

## 4. Passives

Boss passives work differently from player identity passives. Understanding this distinction is essential.

### 4-1. Boss Passives vs. Identity Passives

| | Boss (abnormality-unit) | Identity (personality) |
|---|---|---|
| **Passive file location** | `custom_limbus_data/passive/` | `custom_limbus_data/personality-passive/` |
| **Referenced in** | `passiveSet.passiveIdList` inside unit JSON | `battlePassiveList` / `supporterPassiveList` |
| **Passive definition format** | `id` + `requireIDList` | `personalityID` + `passiveIDList` |
| **Custom Lua trigger** | Via `requireIDList` Modular strings | Not directly supported |

### 4-2. The passiveSet in soji.json

```json
"passiveSet": {
  "passiveIdList": [
    132219,   ← base-game passive (clash bonus)
    132224,   ← base-game passive (morale behavior)
    132220,
    132222,
    132223,
    132214,
    132230,
    9999998,  ← universal abnormality passive — always include
    6820901   ← ★ custom passive: the Lua pattern trigger
  ]
}
```

IDs `132219`–`132230` come directly from the reference character's `passiveSet` found in dumpedData. They work without any custom files because they already exist in the base game.

`9999998` is a universal passive present on virtually every abnormality boss. Always include it.

`6820901` is the only custom ID. It requires a file in `custom_limbus_data/passive/`.

### 4-3. SojiPassive.json — Lua Trigger Passive

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

This passive has no stat effects. Its entire purpose is to **fire a Lua function once per turn**.

Dissecting the `requireIDList` entry:

```
Modular / TIMING:AfterSlots / LUA:SojiPattern / LUAMAIN:onSkillUse
  │              │                  │                    │
  │              │                  │                    └─ Function to call
  │              │                  └─ modular_lua/SojiPattern.lua
  │              └─ Fires after slots are assigned, before skills execute
  └─ Declares use of the ModularSkillScripts system
```

`TIMING:AfterSlots` is the critical choice. It fires **after the game assigns skills to slots for the round, but before those skills begin executing**. This is the only valid window for `skillslotreplace()`:

- **Too early** (before `AfterSlots`): slot assignments haven't happened yet, so you would be overwriting empty slots
- **Correct** (`AfterSlots`): slots are assigned, not yet executed — `skillslotreplace()` can override them
- **Too late** (during skill execution): skills are already mid-animation and slot replacement has no effect

---

## 5. Skills — Deep Dive

All 15 skills live in `custom_limbus_data/skill/soji_skills.json` as a single `list` array.

### 5-1. Full Skill Reference

| ID | Name | Tier | Coins | Sin | Key mechanic |
|---|---|---|---|---|---|
| 682001010 | 연격 | 1 | 4 | VIOLET | Breath ×1/1/1/2; AoE 5 targets |
| 682001020 | 이연잔 | 1 | 5 | VIOLET | Breath ×1 all 5; most frequent in patterns |
| 682001030 | 잔상베기 | 2 | 1 | INDIGO | SuperCoin; Breath +5 if one-sided; PhantomIncision |
| 682001040 | 폭압격 | 2 | — | INDIGO | Forced clash; target count reduced on loss |
| 682001060 | 잔상 해방 | 2 | — | INDIGO | PhantomIncision burst |
| 682001080 | 사냥 | 2 | — | INDIGO | Hunt |
| 682001100 | 처형 | 3 | — | INDIGO | Heavy execution |
| 682001110 | 공진 | 2 | — | INDIGO | Vibration application |
| 682001130 | 이연속참 | 2 | — | VIOLET | Breath + chain |
| 682001140 | 필연쇄 | 2 | — | INDIGO | Chain skill |
| 682001160 | 호흡 강탈 | 2 | — | INDIGO | Breath drain |
| 682001170 | 정신 압박 | 2 | — | INDIGO | Mental pressure |
| 682001180 | 취약 유도 | 2 | — | INDIGO | Weakness induction |
| 682001190 | 취약 강타 | 2 | — | INDIGO | Weakness strike |
| 682001200 | 광역 난사 | 3 | 9 | VIOLET | VibrationExplosion + Vibration 5/5 on all 9 coins |

### 5-2. Anatomy of a Basic Skill — 연격 (682001010)

The full JSON for 연격:

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
    "range": 8,
    "arearange": 5,
    "skillMotion": "S1",
    "viewType": "ENCOUNTER",
    "parryingCloseType": "NEAR",
    "abilityScriptList": [
      { "scriptName": "EmptyBody" },
      {
        "scriptName": "GiveBuffOnUse",
        "buffData": {
          "buffKeyword": "Breath",
          "target": "Self",
          "stack": 0,
          "turn": 2,
          "activeRound": 0
        }
      }
    ],
    "coinList": [
      {
        "operatorType": "ADD",
        "scale": 4,
        "abilityScriptList": [
          {
            "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 1, "turn": 0 }
          },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0" }
        ]
      },
      { /* coin[1] — same as coin[0] */ },
      { /* coin[2] — same as coin[0] */ },
      {
        "operatorType": "ADD",
        "scale": 7,
        "abilityScriptList": [
          {
            "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 2, "turn": 0 }
          },
          { "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame1" }
        ]
      }
    ]
  }]
}
```

**Breaking down `abilityScriptList` (top-level):**

```json
{ "scriptName": "EmptyBody" }
```
Required for every abnormality skill. Declares this as a boss-type attack body. Every boss skill must include this. Omitting it causes the skill to behave incorrectly in clash resolution.

```json
{
  "scriptName": "GiveBuffOnUse",
  "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 0, "turn": 2 }
}
```

This fires when the skill is *selected*, before any coins resolve.

`stack: 0` does **not** add 0 stacks. It means: **extend the Breath buff's duration to 2 turns without changing the current stack count**. This is how Breath persists across rounds — every time 연격 is used, the existing Breath gets 2 more turns, preventing it from expiring.

> `GiveBuffOnUse` + `stack: 0` + `turn: N` = duration refresh only. Use it to keep a persistent buff from expiring.

**Breaking down the coins:**

| Coin | `scale` | Effect |
|---|---|---|
| coin[0] | 4 | Breath +1, play S1 frame 0 |
| coin[1] | 4 | Breath +1, play S1 frame 0 |
| coin[2] | 4 | Breath +1, play S1 frame 0 |
| coin[3] | 7 | Breath +2, play S1 frame 1 |

The final coin has `scale: 7` (vs 4 for the others) — it deals significantly more damage on the last hit. The Lua reference `LUAMAIN:s1_frame1` switches to the finishing animation frame. See [Section 6-1](#6-1-sojimotionlua--per-coin-direct-frame-control).

**`targetNum: 5` and `viewType: "ENCOUNTER"`**

연격 hits up to 5 targets. `viewType: "ENCOUNTER"` zooms the camera out to show all units (full encounter view) rather than focusing on a single clash target. Use `"ENCOUNTER"` for AoE skills, `"BATTLE"` for single-target.

### 5-3. Layered Conditionals — 잔상베기 (682001030)

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
          {
            "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "PhantomIncision", "target": "Target", "stack": 1, "turn": 0 }
          },
          {
            "scriptName": "GiveBuffOnSucceedAttack",
            "buffData": { "buffKeyword": "PhantomIncision", "target": "Target", "stack": 1, "turn": 0 }
          }
        ]
      }
    ]
  }]
}
```

Three mechanics are layered here:

**① `GiveBuffBeforeAttackIfOnesideAttack` — free Breath if unopposed**

```json
"scriptName": "GiveBuffBeforeAttackIfOnesideAttack",
"buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 5, "turn": 0 }
```

Fires before the attack, but **only if there is no clash** (the player left this slot uncontested). The boss gains 5 Breath for free before damage is calculated, meaning that Breath is already active for any Breath-scaling mechanics on this skill.

If the player does contest the slot, this script never fires — the Breath bonus is denied.

**② `TagetNumAdderOnLoseDuel` — disrupted spread**

```json
"scriptName": "TagetNumAdderOnLoseDuel", "value": -2
```

If the boss *loses* the duel, the target count is reduced by 2. A player who contests and wins limits the afterimage slash to fewer targets. This rewards active defending.

**③ The single coin: Unbreakable + SuperCoin + Critical**

```
grade: 2 + color: "GREY"    = Unbreakable — fires even if the player wins the clash
SuperCoin                   = always Heads — power bonus always applies  
CriticalDmgUpByJsonValue    = +30% critical hit damage
```

These three combine: even if the player wins the clash and takes the hit, this coin still fires, always lands at full power, and deals bonus critical damage.

**Two `GiveBuffOnSucceedAttack: PhantomIncision` on one coin:**

Both scripts are in the same coin. They resolve sequentially: the target gains 1 PhantomIncision from the first, then 1 more from the second — 2 total from a single hit. Using two script entries rather than `stack: 2` is intentional; it allows each stack to be tracked and checked individually by downstream scripts.

### 5-4. Power Scaling Reference

| Parameter | Effect |
|---|---|
| `defaultValue` | Base damage floor before coins |
| `skillLevelCorrection` | Additional power added per unit level |
| Coin `scale` | Damage multiplier for that specific coin |
| `skillTier` | Affects the skill's clash power weighting |

For a custom boss at level 77 in sandbox play:

| Stat | Reasonable range |
|---|---|
| `defaultValue` | 8–18 |
| `skillLevelCorrection` | 0–5 |
| Coin `scale` | 3–9 |

### 5-5. The 9-Coin Barrage — 광역 난사 (682001200)

All 9 coins are SuperCoin + Unbreakable. Coins 0–6 use a random animation frame. Coins 7 and 8 use fixed frames.

**Coin 0 (representative — coins 1–6 are identical):**

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
    {
      "scriptName": "GiveBuffOnSucceedAttack",
      "buffData": { "buffKeyword": "Vibration", "target": "Target", "stack": 5, "turn": 5 }
    }
  ]
}
```

**Script execution order per coin:**

```
① SuperCoin             → flip = always Heads; attack resolves
② barrageRandomS2       → at ChangeMotion timing: pick S2[0/1/2] at random
③ VibrationExplosion    → detonate current Vibration stacks on target
④ GiveBuffOnSucceedAttack: Vibration 5/5  → reload 5 stacks for the next coin
```

The explosion (③) fires before the new stacks are loaded (④). This is correct — you detonate what's there, then refill for the next coin.

**The damage chain across all 9 coins:**

```
Coin 0: Explosion (0 stacks if none prior) → Vibration 5 applied
Coin 1: Explosion (5 stacks)               → Vibration 5 applied
Coin 2: Explosion (5 stacks)               → Vibration 5 applied
...
Coin 6: Explosion (5 stacks)               → Vibration 5 applied
Coin 7: Explosion (5 stacks), S6[0]        → Vibration 5 applied
Coin 8: Explosion (5 stacks), S3[0]        → Vibration 5 applied + Breath 3 to Self
```

**Coin 7 and Coin 8 use fixed frames:**

```json
// Coin 7
{ "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s6_frame0" }

// Coin 8 — also grants Breath to the boss
{ "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s3_frame0" },
{ "scriptName": "GiveBuffOnSucceedAttack",
  "buffData": { "buffKeyword": "Breath", "target": "Self", "stack": 3, "turn": 0 } }
```

The final coin switches to the S3 finishing animation (a visually distinct closing blow) and rewards the boss with 3 Breath, signaling the end of the barrage sequence.

---

## 6. Advanced Lua — Three Scripts

The mod contains three Lua files. Each solves a different problem that JSON alone cannot handle.

### 6-1. SojiMotion.lua — Per-Coin Direct Frame Control

**Problem:** The S1 animation has 2 frames. A 4-coin skill using S1 would play frame 0 for coins 0–1 normally, but coins 2–3 would both repeat frame 1 (the last available frame). Desired: coins 0–2 play frame 0, coin 3 plays frame 1.

**Solution — direct per-coin functions:**

```lua
-- SojiMotion.lua
-- Each coin calls the function that matches its intended frame.
-- No getactivations(), no setldata/getldata needed.

-- S1: 682001010 coins[0][1][2] → frame 0, coin[3] → frame 1
function s1_frame0()
    changemotion("S1", 0)
    log("[SojiMotion] S1[0]", 0)
end

function s1_frame1()
    changemotion("S1", 1)
    log("[SojiMotion] S1[1]", 0)
end

-- S2: 682001020 uses a rotating pattern: frame1, frame1, frame2, frame1, frame2
function s2_frame1()
    changemotion("S2", 1)
    log("[SojiMotion] S2[1]", 0)
end

function s2_frame2()
    changemotion("S2", 2)
    log("[SojiMotion] S2[2]", 0)
end

-- S6 and S3: used by 광역 난사 for the final two coins
function s6_frame0()
    changemotion("S6", 0)
    log("[SojiBarrage] apply coin=7 -> S6[0]", 0)
end

function s3_frame0()
    changemotion("S3", 0)
    log("[SojiBarrage] apply coin=8 -> S3[0]", 0)
end

-- No main() defined — getactivations() approach abandoned for direct functions
```

In the JSON, each coin calls the function that matches its intended frame:

```json
"coinList": [
  {
    "scale": 4,
    "abilityScriptList": [
      {"scriptName": "GiveBuffOnSucceedAttack", "buffData": {"buffKeyword": "Breath", ...}},
      {"scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame0"}
    ]
  },
  { /* coin[1]: s1_frame0 */ },
  { /* coin[2]: s1_frame0 */ },
  {
    "scale": 7,
    "abilityScriptList": [
      {"scriptName": "GiveBuffOnSucceedAttack", "buffData": {"buffKeyword": "Breath", "stack": 2 ...}},
      {"scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiMotion/LUAMAIN:s1_frame1"}
    ]
  }
]
```

**Why not use the `getactivations()` approach from Chapter 3?**

Chapter 3's VespaMotion pattern stores the coin index in `main()` and reads it in `LUAMAIN`. That works, but:
- Requires `main()` + `setldata` + `getldata` — three moving parts
- Risk of timing desync if `main()` fires unexpectedly
- The frame assignment is buried in Lua logic, not visible from the JSON

With direct functions:
- Each coin explicitly names its frame in the JSON
- No storage needed
- Zero risk of desync

Use the `getactivations()` approach when frame assignment depends on **runtime state** (e.g., "frame 2 if Breath > 5"). Use direct functions when assignment is **fixed at design time**.

### 6-2. SojiBarrage.lua — Random Frame Per Hit

**Problem:** Barrage coins 0–6 should play a random S2 frame each time for a visually chaotic effect.

**Why `OnCoinToss` failed:**

The first attempt used `TIMING:OnCoinToss` — it seemed like the right timing for per-coin randomization. It never fired. The reason: **`SuperCoin` bypasses the coin flip entirely**, so `OnCoinToss` never triggers on SuperCoins. The solution is `ChangeMotion`, which fires regardless of whether a flip occurred:

```lua
-- SojiBarrage.lua

function barrageRandomS2()
    local frame = random(0, 2)         -- 0, 1, or 2 with equal probability
    changemotion("S2", frame)
    log("[SojiBarrage] S2[" .. frame .. "]", 0)
end
```

In each of coins 0–6 of the barrage skill:

```json
{ "scriptName": "Modular/TIMING:ChangeMotion/LUA:SojiBarrage/LUAMAIN:barrageRandomS2" }
```

**Execution flow for one barrage coin:**

```
SuperCoin fires     → flip skipped, attack resolves
                    → OnCoinToss never fires (no flip occurred)
VibrationExplosion  → detonates Vibration stacks
GiveBuffOnSucceedAttack: Vibration → reloads 5 stacks
ChangeMotion timing → barrageRandomS2() called
  frame = random(0,2) = e.g. 1
  changemotion("S2", 1)  ✓
```

The animation plays last, after damage resolution.

> **Key rule:** `TIMING:OnCoinToss` never fires on `SuperCoin` skills. Use `TIMING:ChangeMotion` for any animation logic on SuperCoin skills.

### 6-3. SojiPattern.lua — The 3-Cycle Pattern Controller

This is the most complex script. It dynamically replaces all 7 skill slots each round based on which cycle the current round belongs to, with random selection from multiple variants per cycle.

**The full script:**

```lua
-- SojiPattern.lua
-- 3-Cycle pattern controller (7 slots)
-- Triggered each round via passive requireIDList → AfterSlots

local CYCLE_PATTERNS = {
    -- Cycle 1 (rounds 1, 4, 7, ...): 광역 난사 always in slot 0
    {
        {682001200, 682001030, 682001110, 682001100, 682001140, 682001060, 682001020},
        {682001200, 682001080, 682001040, 682001110, 682001100, 682001020, 682001060},
        {682001200, 682001030, 682001080, 682001110, 682001040, 682001100, 682001020}
    },
    -- Cycle 2 (rounds 2, 5, 8, ...): Breath accumulation
    {
        {682001020, 682001010, 682001110, 682001130, 682001100, 682001060, 682001020},
        {682001140, 682001020, 682001010, 682001110, 682001080, 682001060, 682001020}
    },
    -- Cycle 3 (rounds 3, 6, 9, ...): PhantomIncision pressure
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
    local currentRound = round()
    local cycleIdx = (currentRound - 1) % 3 + 1
    ApplyCyclePattern(cycleIdx)
end
```

**Breaking down `CYCLE_PATTERNS`:**

The outer table has 3 entries (one per cycle). Each cycle contains 2–3 arrays, each array being a full 7-slot skill assignment. For example, Cycle 1 has 3 variants — all three guarantee 682001200 (광역 난사) in slot 0, but differ in the other 6 slots.

Note: 682001020 (이연잔) appears in almost every pattern across all cycles. This is intentional — it is the "anchor" skill that ensures Breath always accumulates regardless of which variant is selected.

**Breaking down `ForceSlot`:**

```lua
local function ForceSlot(slotIdx, targetSkillID)
    for _, sid in ipairs(ALL_SKILLS) do
        skillslotreplace(slotIdx, sid, targetSkillID)
    end
end
```

`skillslotreplace(slot, fromSkill, toSkill)` replaces the skill in `slot` **only if its current occupant is `fromSkill`**. Because the script doesn't know which skill `patternList` assigned to any given slot this round, it tries replacing from every possible skill ID. Exactly one call will match; all others are no-ops. This brute-force approach works around the limitation that `skillslotreplace` cannot say "replace whatever is in slot N."

**Breaking down the cycle index calculation:**

```lua
local cycleIdx = (currentRound - 1) % 3 + 1
```

```
Round 1: (1-1) % 3 + 1 = 0 % 3 + 1 = 1  → Cycle 1
Round 2: (2-1) % 3 + 1 = 1 % 3 + 1 = 2  → Cycle 2
Round 3: (3-1) % 3 + 1 = 2 % 3 + 1 = 3  → Cycle 3
Round 4: (4-1) % 3 + 1 = 0 % 3 + 1 = 1  → Cycle 1  ← repeats
```

**How the passive connects this to the game loop:**

```json
// SojiPassive.json
"requireIDList": ["Modular/TIMING:AfterSlots/LUA:SojiPattern/LUAMAIN:onSkillUse"]
```

Each round:
1. Game assigns skills to slots from `patternList` (as normal)
2. `AfterSlots` fires
3. Passive triggers `onSkillUse()`
4. `onSkillUse` calculates cycle, picks a random variant, force-replaces all 7 slots
5. Skills execute with the Lua-assigned loadout — `patternList` assignments are fully overridden

The `patternList` in `soji.json` must still be present with valid skill IDs (the engine requires it to exist). All patterns are set with `682001200` in slot 0 as a safe fallback, since Lua overwrites every slot before execution anyway.

**Reading the log output during testing:**

```
[SojiPattern] Round=1 Cycle=1 Slot0=682001200 Slots=7
[SojiPattern] Round=2 Cycle=2 Slot0=682001020 Slots=7
[SojiPattern] Round=3 Cycle=3 Slot0=682001110 Slots=7
[SojiPattern] Round=4 Cycle=1 Slot0=682001200 Slots=7
```

Slot0 on Cycle 1 is always 682001200 (광역 난사) — confirming the barrage is guaranteed every Cycle 1 round.

---

## 7. Common Mistakes Made During Development

These are real errors that occurred during development, documented exactly as they happened.

### 7-1. `OnCoinToss` Never Fires on SuperCoins

**What happened:** The random barrage frame selector was attached to `TIMING:OnCoinToss`, assuming it would fire once per coin flip.

**Result:** The function never executed. Barrage played the same frame every time.

**Root cause:** `SuperCoin` bypasses the coin flip. No flip = `OnCoinToss` never fires. This applies to every SuperCoin in the game.

**Fix:** Switch from `TIMING:OnCoinToss` to `TIMING:ChangeMotion`. Animation control fires at `ChangeMotion` for all coins, including SuperCoins.

### 7-2. `skillslotreplace` Silently Failing

**What happened:** Calling `skillslotreplace(0, 682001200, 682001200)` to set slot 0 to the barrage skill. Slot 0 sometimes remained with the old skill from `patternList`.

**Root cause:** `skillslotreplace` requires the `from` argument to exactly match what's currently in the slot. If `patternList` assigned 682001010 to slot 0 that round, `skillslotreplace(0, 682001200, X)` does nothing because 682001200 is not in slot 0.

**Fix:** Iterate through all skill IDs and attempt replacement from each. One will match:

```lua
for _, sid in ipairs(ALL_SKILLS) do
    skillslotreplace(slotIdx, sid, targetSkillID)
end
```

### 7-3. Locale `coindescs` Count Must Match Exactly

**Symptom:** Skill tooltip shows blank or misaligned coin descriptions.

**Cause:** Locale had a different number of `coindescs` entries than the actual coin count in the skill JSON.

**Rule:** Count `coinList` entries → set exactly that many `coindescs`. Check every skill individually. For 광역 난사 (9 coins), 9 `coindescs` are required.

### 7-4. `GiveBuffOnUse` with `stack: 0` Does Not Add Stacks

**What it does:** Extends the buff's duration. Does **not** increase stack count. Used intentionally in this mod to keep Breath alive across turns without inflating stacks.

**Common confusion:** Expecting `stack: 0` to remove the buff or do nothing. It specifically means "maintain/refresh the existing buff without changing its stack count."

---

## Appendix: Script & Technique Reference

| Technique | File | Key code |
|---|---|---|
| Duration refresh without stacking | `soji_skills.json` | `GiveBuffOnUse, stack: 0, turn: N` |
| Pre-attack conditional buff | `soji_skills.json` | `GiveBuffBeforeAttackIfOnesideAttack` |
| Target count reduction on loss | `soji_skills.json` | `TagetNumAdderOnLoseDuel, value: -2` |
| Unbreakable coin | `soji_skills.json` | `grade: 2, color: "GREY"` |
| Always-heads coin | `soji_skills.json` | `SuperCoin` in `abilityScriptList` |
| Critical damage bonus | `soji_skills.json` | `CriticalDmgUpByJsonValue, value: 0.3` |
| Vibration chain | `soji_skills.json` | `VibrationExplosionOnSucceedAttack` then `GiveBuffOnSucceedAttack: Vibration` |
| Per-coin fixed frame | `SojiMotion.lua` | Direct function `s1_frame0()`, `s1_frame1()` etc. |
| Per-coin random frame | `SojiBarrage.lua` | `barrageRandomS2()` with `random(0, 2)` |
| 3-cycle pattern rotation | `SojiPattern.lua` | `onSkillUse()` → `ApplyCyclePattern()` |
| Force-replace any slot content | `SojiPattern.lua` | `ForceSlot()` iterates all skill IDs |
| Passive as round trigger | `SojiPassive.json` | `requireIDList` with `AfterSlots` timing |

---

*For TIMING keyword reference and the full `changemotion`/`skillslotreplace` function list, see [Chapter 3 — Advanced Lua](./chapter3_EN.md).*
