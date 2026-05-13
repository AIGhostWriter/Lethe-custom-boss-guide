# Chapter 3 — Advanced Lua: ModularSkillScripts
> **Solving What JSON Cannot Do**

*Previous: [Chapter 2 — Creating an Encounter Mod](./chapter2_EN.md)*

---

## Table of Contents

1. [Why Lua?](#1-why-lua)
2. [File Structure and Wiring](#2-file-structure-and-wiring)
3. [TIMING — When Does It Execute?](#3-timing--when-does-it-execute)
4. [Available Lua Functions](#4-available-lua-functions)
5. [The Core Pattern: main + LUAMAIN Two-Step](#5-the-core-pattern-main--luamain-two-step)
6. [Practical Example: Controlling Animation Frames](#6-practical-example-controlling-animation-frames)
7. [Debugging Lua Scripts](#7-debugging-lua-scripts)
8. [Going Further](#8-going-further)

---

## 1. Why Lua?

JSON is **declarative**. You can define "this skill has this effect," but you cannot express conditional branching like "behave differently depending on the current state."

**Things JSON cannot do:**

| Desired Behavior | Reason |
|-----------------|--------|
| Play different animations per coin index | JSON has no way to read the coin index |
| Use a different skill when HP is below 50% | JSON has no conditional branching |
| Change behavior based on the skill used last turn | JSON has no state storage |
| Calculate values based on buff stack count | JSON has no runtime computation |

Lethe supports these behaviors through the **ModularSkillScripts** system.  
This system connects Lua scripts to specific execution timings on skill coins.

---

## 2. File Structure and Wiring

### 2-1. File Location

Lua scripts are stored as `.lua` files inside the `modular_lua/` folder in your mod directory.

```
MyMod/
└── modular_lua/
    └── VespaMotion.lua     ← The filename becomes the script name
```

### 2-2. Calling a Lua Function from JSON

Add the following format to a skill's `abilityScriptList` to connect a Lua function.

```json
{
  "scriptName": "Modular/TIMING:ChangeMotion/LUA:VespaMotion/LUAMAIN:applyVespaMotion"
}
```

Breaking down the string:

```
Modular / TIMING:ChangeMotion / LUA:VespaMotion / LUAMAIN:applyVespaMotion
  │              │                    │                      │
  │              │                    │                      └─ Function to call at ChangeMotion timing
  │              │                    └─ References modular_lua/VespaMotion.lua
  │              └─ The timing at which the LUAMAIN function executes
  └─ Declares use of the ModularSkillScripts system
```

- `Modular/` — declares that this scriptName uses the ModularSkillScripts system
- `TIMING:ChangeMotion` — the timing at which the `LUAMAIN` function runs
- `LUA:VespaMotion` — references the file `modular_lua/VespaMotion.lua`
- `LUAMAIN:applyVespaMotion` — the function to call at that timing

> **The `main()` function runs automatically at the default timing.**  
> Even without a `LUAMAIN` designation, the `main()` function inside the Lua file is called automatically every time a coin executes.

### 2-3. A Real Skill JSON Example

```json
{
  "operatorType": "ADD",
  "grade": 2,
  "color": "GREY",
  "scale": 2,
  "abilityScriptList": [
    {"scriptName": "SuperCoin"},
    {"scriptName": "ActivateTargetLacerationOnSucceedAttack", "value": 1},
    {"scriptName": "Modular/TIMING:ChangeMotion/LUA:VespaMotion/LUAMAIN:applyVespaMotion"}
  ]
}
```

Regular scriptNames and Modular scriptNames can be freely mixed in the same `abilityScriptList`.

---

## 3. TIMING — When Does It Execute?

The `TIMING:` keyword determines **when** the function specified by `LUAMAIN` runs.  
It is separate from the "default timing" at which `main()` runs.

### Key TIMING Values

| TIMING Keyword | Execution Point |
|---------------|----------------|
| `ChangeMotion` | Immediately before the coin's animation plays |
| `BeforeAttack` | Immediately before the attack is resolved |
| `WhenUse` | When the skill is selected/used (before any coin executes) |
| `OnCoinToss` | When the coin is flipped (Heads/Tails determination) |
| `RoundStart` | At the start of the round |

### Execution Order of main() and LUAMAIN

When a single coin executes, the timing order is:

```
[Coin selected]
    ↓
main() executes        ← Default timing. Read and store data here
    ↓
[TIMING:OnCoinToss]   ← Heads/Tails determination
    ↓
[TIMING:BeforeAttack] ← Immediately before attack resolution
    ↓
[TIMING:ChangeMotion] ← Immediately before animation. Call changemotion() here
    ↓
[Animation plays]
    ↓
[Attack effects applied]
```

> **Why separate main() from LUAMAIN?**  
> `changemotion()` can only be called at the `ChangeMotion` timing.  
> But `getactivations()` (the current coin index) works more reliably at the default timing (`main()`).  
> So the pattern is: read and save the coin index in `main()`, then retrieve it in `LUAMAIN` to act on it.

---

## 4. Available Lua Functions

### 4-1. Reading Current State

```lua
-- Returns the current coin index (0-based)
local coinIdx = getactivations()

-- Returns the ID of the currently executing skill
local skillId = getskillid()

-- Returns the current round (turn) number
local turn = round()

-- Returns the unit count for ally/enemy
local count = unitcount("Target")  -- "Self", "Target", "SubTarget", etc.
```

### 4-2. Reading / Writing Buffs

```lua
-- Read buff stack count
local stack = getbuff("Self", "Vibration", "stack")
-- or
local stack = bufcheck("Self", "Vibration", "stack")

-- Apply a buff
buff("Self", "Haste", 2, 3, 0)
-- buff(target, buffKeyword, stack, turn, activeRound)

-- Remove a buff
destroybuff("Self", "Vibration", 0)
```

### 4-3. Storing / Reading Data

These functions store and retrieve data that persists for the duration of combat.  
They are essential for passing values between the coin execution timing and the ChangeMotion timing.

```lua
-- Store a value (persists until combat ends)
setldata("Self", "myKey", 42)
setldata("Self", "myKey", "strings work too")
setldata("Self", "toDelete", nil)  -- storing nil = deletes the entry

-- Read a value (returns nil if not found)
local val = getldata("Self", "myKey")

-- nil check example
local val = getldata("Self", "myKey") or 0  -- defaults to 0 if not found
```

### 4-4. Animation Control

```lua
-- Change motion (only valid at TIMING:ChangeMotion)
changemotion("S10", 2)
-- changemotion(motionName, frameIndex)
-- motionName: "S1", "S2", ... "S10", "Default", etc.
-- frameIndex: 0-based. Which frame of that motion to play
```

### 4-5. Skill Slot Control

```lua
-- Replace the skill in a specific slot with a different skill
skillslotreplace(slotIndex, currentSkillID, replacementSkillID)

-- Get the current number of skill slots
local slots = skillslotcount("Self")
```

### 4-6. Other Useful Functions

```lua
-- Write to the log (recorded in BepInEx/LogOutput.log)
log("debug message", 0)

-- Generate a random integer (inclusive on both ends)
local r = random(0, 3)  -- one of: 0, 1, 2, 3

-- Call a function from another Lua file
require("OtherFileName")()
```

---

## 5. The Core Pattern: main + LUAMAIN Two-Step

This is the most commonly used pattern in ModularSkillScripts.  
"Store data at the default timing; retrieve and use it at a specific timing."

### Pattern Structure

```lua
-- [Step 1] main() — automatically called at coin execution timing
-- Role: read the current state (coin index, buffs, etc.) and save it
function main()
    local coinIdx = getactivations()
    local skillId = getskillid()
    setldata("Self", "saveKey_" .. skillId, coinIdx)
end

-- [Step 2] LUAMAIN function — called at the timing specified by TIMING:
-- Role: read the saved data and perform the actual action
function applyMotion()
    local skillId = getskillid()
    local coinIdx = getldata("Self", "saveKey_" .. skillId) or 0
    
    -- Actual action
    changemotion("S10", coinIdx)
end
```

```json
In the skill JSON:
{
  "scriptName": "Modular/TIMING:ChangeMotion/LUA:MyScript/LUAMAIN:applyMotion"
}
```

### Why Include skillId in the Key Name?

When multiple skills share the same Lua file, each skill needs its own storage space.  
Including `skillId` in the key name prevents each skill's data from interfering with another's.

```lua
setldata("Self", "vmotion_" .. 681001100, 3)  -- data for skill 100
setldata("Self", "vmotion_" .. 681001120, 7)  -- data for skill 120
-- These are different keys, so no conflict
```

---

## 6. Practical Example: Controlling Animation Frames

### 6-1. The Problem

Building a 10-coin skill with `skillMotion: "S10"`:
- If the S10 animation only has 5 frames (0–4)
- coin[5]–coin[9] all repeat the last frame (frame 4)

Desired behavior: from coin[5] onwards, alternate between frames 2 and 3

```
coin[0] → S10[0]   coin[5] → S10[2] ← manually assigned
coin[1] → S10[1]   coin[6] → S10[3] ← manually assigned
coin[2] → S10[2]   coin[7] → S10[2] ← manually assigned
coin[3] → S10[3]   coin[8] → S10[3] ← manually assigned
coin[4] → S10[4]   coin[9] → S10[2] ← manually assigned
```

### 6-2. Full VespaMotion.lua Code

```lua
-- VespaMotion.lua
-- Controls animation frames per coin for Vespa's 10-coin skills

-- Mapping of skill ID → motion name to control
local SKILL_MOTION = {
    [681001100] = "S10",   -- Harpoon Barrage
    [681001120] = "S9",    -- Crowd Harpoon
}

-- Up to this many coins play in order (0–4)
local BASE_FRAME_COUNT = 5

-- Frames to loop for coins beyond BASE_FRAME_COUNT
local LOOP_FRAMES = {2, 3}

-- [Step 1] Save current coin index at coin execution timing
function storeActivation()
    local coinIdx = getactivations()
    local skillId = getskillid()
    setldata("Self", "vmotion_" .. skillId, coinIdx)
end

-- [Step 2] Determine frame from saved index at ChangeMotion timing
function applyVespaMotion()
    local skillId = getskillid()
    local motionType = SKILL_MOTION[skillId]
    if not motionType then return end  -- not a controlled skill, skip

    -- Read saved coin index (fallback to getactivations() if missing)
    local coinIdx = getldata("Self", "vmotion_" .. skillId)
    if coinIdx == nil then
        coinIdx = getactivations()
    end

    local frameIdx
    if coinIdx < BASE_FRAME_COUNT then
        -- Coins 0–4: play in order
        frameIdx = coinIdx
    else
        -- Coin 5+: cycle through LOOP_FRAMES
        -- Lua arrays are 1-indexed, so +1 is required
        local loopPos = (coinIdx - BASE_FRAME_COUNT) % #LOOP_FRAMES
        frameIdx = LOOP_FRAMES[loopPos + 1]
    end

    changemotion(motionType, frameIdx)
end

-- main() is automatically called on every coin execution
function main()
    storeActivation()
end
```

### 6-3. Connecting to the Skill JSON

In `vespa_skills.json`, add the following line to **every coin** in skills 681001100 and 681001120.

```json
"coinList": [
  {
    "operatorType": "ADD",
    "grade": 2,
    "color": "GREY",
    "scale": 2,
    "abilityScriptList": [
      {"scriptName": "SuperCoin"},
      {"scriptName": "ActivateTargetLacerationOnSucceedAttack", "value": 1},
      {"scriptName": "Modular/TIMING:ChangeMotion/LUA:VespaMotion/LUAMAIN:applyVespaMotion"}
    ]
  },
  ... (remaining coins, same structure)
]
```

> **Watch out:** There must be no trailing comma after the last item in `abilityScriptList`. This is the most common JSON format error.

### 6-4. Step-by-Step Execution Flow

```
coin[7] executes
  │
  ├─ main() called
  │     coinIdx = getactivations()  → 7
  │     setldata("Self", "vmotion_681001100", 7)
  │
  ├─ [Attack resolved]
  │
  └─ ChangeMotion timing
        applyVespaMotion() called
        coinIdx = getldata("Self", "vmotion_681001100")  → 7
        7 >= 5, so use LOOP_FRAMES
        loopPos = (7 - 5) % 2 = 0
        frameIdx = LOOP_FRAMES[0 + 1] = LOOP_FRAMES[1] = 2
        changemotion("S10", 2)  → plays frame 2 of S10 ✓
```

### 6-5. Customization Guide

| What to Change | Variable to Modify | Example |
|---------------|-------------------|---------|
| When looping begins | `BASE_FRAME_COUNT` | Set to `3` → looping starts at coin[3] |
| Which frames to loop | `LOOP_FRAMES` | `{1, 2, 3}` → loops 1→2→3→1→2→3 |
| Apply to other skills | `SKILL_MOTION` table | Add skill ID and motion name pairs |

---

## 7. Debugging Lua Scripts

### 7-1. Inspecting Values with log()

```lua
function applyVespaMotion()
    local coinIdx = getldata("Self", "vmotion_" .. getskillid()) or -1
    log("coinIdx = " .. coinIdx, 0)  -- prints to LogOutput.log
    
    changemotion("S10", coinIdx)
end
```

Open `BepInEx/LogOutput.log` and you'll see:

```
[Info   :ModularSkillScripts] coinIdx = 7
```

### 7-2. Common Lua Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `[Error: ModularSkillScripts]` | Lua syntax error | Check for missing `end`, mismatched quotes |
| Function never executes | File name and `LUA:` name don't match | Verify `LUA:VespaMotion` ↔ `VespaMotion.lua` |
| Function never executes | Function name and `LUAMAIN:` don't match | Check spelling of function name |
| `nil` value error | `getldata` return value not checked | Use `or defaultValue` pattern |
| Animation doesn't change | `changemotion` called at wrong timing | Confirm `TIMING:ChangeMotion` is set |

### 7-3. Quick Lua Syntax Reference

```lua
-- Variables
local x = 10
local name = "vespa"

-- Conditionals
if x > 5 then
    -- ...
elseif x == 3 then
    -- ...
else
    -- ...
end

-- Loops
for i = 0, 9 do
    log("i = " .. i, 0)
end

-- Tables (arrays, 1-indexed)
local t = {10, 20, 30}
local first = t[1]   -- 10  (Note: Lua starts at 1, not 0)
local len = #t       -- 3

-- Tables (dictionaries)
local map = {[681001100] = "S10", [681001120] = "S9"}
local val = map[681001100]  -- "S10"

-- nil check
if val == nil then
    -- value is absent
end

-- String concatenation
local msg = "coin: " .. coinIdx  -- use ".." operator
```

---

## 8. Going Further

### 8-1. Reading Other Mods' Lua Code

Other mods in the `mods/` folder also have Lua scripts in `modular_lua/`.  
These are excellent references for learning different patterns:

| Mod | File | What You Can Learn |
|-----|------|--------------------|
| `TheReloadLeiHeng` | `skillpattern.lua` | Swapping skill slots based on buff conditions |
| `TheReloadLeiHeng` | `bp_reload.lua` | Conditional bullet reload logic |

### 8-2. Splitting into Multiple Lua Files

When scripts become complex, split them into separate files and connect them with `require`.

```lua
-- In VespaPhase2.lua, call a function from VespaMotion.lua
require("VespaMotion")

function phase2Motion()
    applyVespaMotion()  -- calls the function from VespaMotion.lua
end
```

### 8-3. Things to Try Next

- **Phase transition on HP ratio**: save phase state with `setldata`, call `skillslotreplace` based on condition
- **Tracking skills used**: record "did skill X get used this combat?" with `setldata`
- **Conditional animation**: only apply `changemotion` when a specific buff has N or more stacks

---

## Full Guide Summary

| Chapter | Content | Key Files |
|---------|---------|-----------|
| Chapter 1 | Analyzing game data, planning IDs, avoiding collisions | Browsing `dumpedData/` |
| Chapter 2 | Writing encounter, unit, part, skill, and pattern JSON | `encounter.json`, `vespa.json`, `vespa_skills.json` |
| Chapter 3 | Controlling runtime behavior with Lua, specifying animation frames | `VespaMotion.lua` |
