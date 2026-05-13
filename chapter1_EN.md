# Chapter 1 — Finding Your Boss Data
> **A mandatory step before you start modding**

---

## Table of Contents

1. [Understanding the dumpedData Folder](#1-understanding-the-dumpeddata-folder)
2. [Finding Your Target Character](#2-finding-your-target-character)
3. [Finding Skill Information](#3-finding-skill-information)
4. [Referencing Existing Abnormality Bosses](#4-referencing-existing-abnormality-bosses)
5. [ID Collision — The Most Dangerous Mistake](#5-id-collision--the-most-dangerous-mistake)

---

## 1. Understanding the dumpedData Folder

Lethe extracts game data into human-readable JSON files.  
This folder is **the reference manual for all modding**.

```
BepInEx/plugins/Lethe/dumpedData/
├── identities/          ← Character (Identity) data
├── skills/              ← Skill data (one file per skill ID)
├── encounters/          ← Existing battle stage data
└── limbus_data/
    ├── abnormality-unit/  ← Abnormality (boss) unit data
    ├── abnormality-part/  ← Abnormality part (body) data
    ├── skill/             ← Skill data (bundled files)
    ├── buff/              ← Buff data
    └── passive/           ← Passive data
```

### Folder Overview

| Folder | Contents | When to Use |
|--------|----------|-------------|
| `identities/` | Character appearance ID, skill IDs, passive list | Finding a character's appearance value |
| `skills/` | Detailed skill data by ID (motions, coins, effects) | Looking up motion names and coin structure |
| `limbus_data/abnormality-unit/` | Pattern, stats, and structure of real abnormality bosses | Referencing boss data structure wholesale |
| `encounters/` | Existing battle stage configuration | Reference for writing encounter.json |

> **Rule of thumb: Before creating anything, search dumpedData for something similar first.**  
> Referencing existing data causes far fewer mistakes than writing from scratch.

---

## 2. Finding Your Target Character

### 2-1. Browsing the identities Folder

The `dumpedData/identities/` folder uses character ID numbers as file names.  
Since there are hundreds of files, use your text editor's **global search** feature.

**VS Code global search shortcut:** `Ctrl + Shift + F`

Use the character's English name or a distinctive keyword as your search term.

```
Search path: dumpedData/identities/
Search terms: "Vespa", "vespa", "VESPA"
```

### 2-2. Information to Extract from the Character File

For Vespa, the data lives in `dumpedData/identities/105317.json`.

```json
{
  "id": 105317,
  "nameID": 400006,
  "appearance": "400006_VespaAppearance",
  "attributeType": "AMBER",
  "unitKeywordList": ["SMALL", "UNIQUE_COLOR", "VESPA", "FIXER"],
  "associationList": ["LIMBUS_COMPANY", "LIMBUS_COMPANY_LCD"],
  "passiveSet": {
    "passiveIdList": [40000901, 40000602, 40000902, 9999998, 9999999]
  },
  "attributeList": [
    {"skillId": 400006010},
    {"skillId": 400006020},
    {"skillId": 400006030},
    {"skillId": 400006040},
    {"skillId": 400006050}
  ]
}
```

**Checklist — what to note from this file:**

| Item | Field | Vespa Example | Purpose |
|------|-------|---------------|---------|
| Appearance ID | `appearance` | `"400006_VespaAppearance"` | Makes your boss look like this character |
| Name ID | `nameID` | `400006` | Can be reused for the boss's display name |
| Skill ID list | `attributeList[].skillId` | `400006010`, `400006020`... | For looking up skill motion names |
| Keywords | `unitKeywordList` | `["VESPA", "FIXER"...]` | Can be reused on your boss unit |
| Passive list | `passiveSet.passiveIdList` | `[40000901, 40000602...]` | Can be reused on your boss |

> **`appearance` is the key field.**  
> Paste this value into your custom boss unit's `appearance` field and the boss will use that character's model and animations.

---

## 3. Finding Skill Information

Once you have the character's skill IDs, look up the detailed data for each skill.

### 3-1. Opening Skill Files

The `dumpedData/skills/` folder uses skill IDs as file names.

```
dumpedData/skills/
├── 400006010.json   ← Vespa Skill 1
├── 400006020.json   ← Vespa Skill 2
├── 400006030.json   ← Vespa Skill 3
...
```

### 3-2. Information to Extract from a Skill File

Opening `dumpedData/skills/400006010.json`:

```json
{
  "id": 400006010,
  "skillData": [
    {
      "skillMotion": "S1",
      "attributeType": "AMBER",
      "atkType": "SLASH",
      "defType": "ATTACK",
      "targetNum": 1,
      "defaultValue": 20,
      "coinList": [
        {
          "operatorType": "ADD",
          "scale": 2,
          "abilityScriptList": [ ... ]
        },
        {
          "operatorType": "ADD",
          "scale": 2,
          "abilityScriptList": [ ... ]
        }
      ]
    }
  ]
}
```

**Checklist — what to note from a skill file:**

| Item | Field | Vespa S1 Example | Purpose |
|------|-------|-----------------|---------|
| Motion name | `skillMotion` | `"S1"` | Reuse this motion in your custom skill |
| Coin count | Length of `coinList` array | `2` | The natural coin count for this motion |
| Attack type | `atkType` | `"SLASH"` | Reference for skill design |

### 3-3. Mapping Out Vespa's Full Motion List

Open all of a character's skill files and organize the motion names:

| Skill File | skillMotion | Natural Coin Count |
|------------|------------|---------------------|
| `400006010.json` | `S1` | 2 coins |
| `400006020.json` | `S2` | 2 coins |
| `400006030.json` | `S3` | 1 coin |
| `400006040.json` | `S4` | 1 coin |
| `400006050.json` | `S5` | 1 coin |
| `400006060.json` | `S6` | 2 coins |
| `400006070.json` | `Default` | 1 coin |

> **What is the "natural coin count"?**  
> It refers to the number of animation frames the motion has.  
> If your coin count exceeds this number, **the last frame repeats**.  
> For example, a 5-coin skill using the `S1` motion plays coin[0]–coin[1] normally; coin[2]–coin[4] all repeat the same frame as coin[1].  
> To fix this, see the `changemotion()` guide in Chapter 3.

---

## 4. Referencing Existing Abnormality Bosses

The most efficient way to build a custom boss is to **copy the structure of a similar existing boss**.

### 4-1. Browsing the abnormality-unit Folder

```
dumpedData/limbus_data/abnormality-unit/
```

The JSON files here are grouped by numeric index. Use global search to find the boss you want.

```
Search terms: "VespaAppearance", "Vespa", "8001"
```

### 4-2. What to Reference in a Boss File

From an existing boss file:

```json
{
  "list": [
    {
      "id": existingBossID,
      "appearance": "appearanceID",
      "classType": "ALEPH",
      "hp": { "defaultStat": 90, "incrementByLevel": 30 },
      "minSpeedList": [4],
      "maxSpeedList": [7],
      "patternID": "PickByPattern_Abnormality_UptoActionSlotCnt",
      "abnormalityPartList": [partID],
      "patternList": [ ... ]
    }
  ]
}
```

Reference points:
- **HP values**: Understand balance by looking at the ratio of `defaultStat` to `incrementByLevel`
- **Speed range**: Find a boss of similar difficulty and use those numbers
- **Pattern count**: How many turns does a full cycle take?

### 4-3. Referencing the abnormality-part Folder

```
dumpedData/limbus_data/abnormality-part/
```

When designing resistances, compare existing boss values:

```json
"resistInfo": {
  "atkResistList": [
    {"type": "SLASH",     "value": 0.5},  ← Weak
    {"type": "PENETRATE", "value": 1.0},  ← Normal
    {"type": "HIT",       "value": 2.0}   ← Resistant
  ]
}
```

---

## 5. ID Collision — The Most Dangerous Mistake

### 5-1. How Lethe Handles IDs

When the game starts, Lethe reads files from `custom_limbus_data/` and **merges** them into the base game data.  
If **an ID already exists, the original data is overwritten silently.**

```
Original game data:  skill ID 400006010 → Vespa Basic Attack
Your mod:            skill ID 400006010 → your custom skill  ← Collision!

Result: Every Vespa Identity in the game now uses your skill instead
```

This happens **silently, with no error**. The game runs normally until a specific situation triggers the wrong behavior, making the cause very hard to trace.

### 5-2. Symptoms of a Collision

| Symptom | Possible Cause |
|---------|---------------|
| An existing character's skill behaves strangely | Skill ID collision |
| An existing boss's behavior pattern changes | Unit ID collision |
| The game crashes in a specific battle | Part ID collision causes a broken reference |
| A character's passive disappears | Passive ID collision |

### 5-3. Safe ID Ranges

Most of the game's existing IDs use low number ranges.  
Using the following ranges for custom content greatly reduces collision risk.

| Item | Ranges to Avoid | Recommended Range |
|------|----------------|-------------------|
| encounter ID | 1 – 100,000 | **600,000+** |
| Unit ID | 1 – 9,999 | **6,000,000+** |
| Part ID | Same as unit ID | **Unit ID + 100** |
| Skill ID | 1 – 9,999,999 | **Unit ID × 10 + number** |
| Buff ID | 1 – 9,999 | **Verify manually** |

> **The Yellow Harpoon mod's ID scheme:**
> ```
> encounter: 681001
> unit:      6810001
> part:      6810101   (unit + 100)
> skills:    681001010, 681001020, 681001030...  (in steps of 10)
> ```
> Skills are spaced 10 apart so that a skill like `681001015` can be inserted between existing ones later.

### 5-4. How to Check if an ID Is Already in Use

Search dumpedData to verify whether your chosen ID is taken:

**VS Code global search example:**

```
Search path: dumpedData/
Search term:  "id": 6810001
```

If no results appear, the ID is safe. If results appear, that ID is already used — pick a different number.

> **Tip:** Mixing your initials or a date into your ID numbers can also prevent collisions with other mods.  
> Example: creator `LH` → `681` + `001` = `681001` — use a number combination that means something to you.

### 5-5. Intentional Overwrite — Advanced Use

ID collision isn't always bad. If your **goal is to modify existing game content**, you can deliberately use the original ID to overwrite it.

For example, if you want to buff Vespa's existing Identity (ID `105317`), you can create a custom skill with the same skill ID (`400006010`) to replace the original.

Keep in mind:
- The replacement affects **every piece of content in the game** that uses that ID
- Removing the mod restores the original (Lethe never modifies the original files)
- If another mod overwrites the same ID, **the mod loaded later wins**

---

## Summary: Pre-Modding Checklist

Follow this order when starting a new boss mod to minimize mistakes.

```
□ 1. Find your target character in dumpedData/identities/
□ 2. Note the appearance value          → use for the boss's model
□ 3. Note the nameID value              → can be reused for the boss's name
□ 4. Note the skillId list              → for looking up each skill's skillMotion
□ 5. Open each skill file in dumpedData/skills/ and record the motion names
□ 6. Find a similar existing boss in abnormality-unit/ and reference its structure
□ 7. Decide on your ID numbers in advance
□ 8. Search all of dumpedData to confirm no ID collisions
□ 9. Proceed to Chapter 2 to write the actual files
```

---

*Next: [Chapter 2 — Creating an Encounter Mod](./chapter2_EN.md)*
