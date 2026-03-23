# Pet Satchel System for EQEmu

A custom pet equipment system for EQEmu servers that allows players to pre-load gear into specialized containers. When a pet is summoned, the gear is automatically equipped on the pet.

## Overview

The Pet Satchel System adds three container items (one per pet class) that auto-equip their contents onto summoned pets. Players place armor, weapons, and other equipment into their class-specific satchel, and every time they summon a pet, those items are automatically equipped.

**Pet Classes Supported:**
- Beastlord (BST)
- Necromancer (NEC)
- Magician (MAG)

## Features

- **Auto-equip on summon** — pets are geared instantly when summoned
- **Live re-equip** — `.petequip` command applies gear to already-summoned pets without resummoning
- **Pet stats inspection** — `.petstats` shows all pet stats including equipment bonuses
- **Charm support** — charmed pets also receive satchel equipment
- **Bank storage** — satchels work from both inventory and bank slots

---

## Quick Start — Implementation Order

Follow these steps in order:

1. **[Database Setup](#database-setup)** — Create the 3 satchel container items in SQL
2. **[C++ Implementation](#c-implementation)** — Add `ApplyPetBagEquipment()`, `GetPetOriginClass()`, call sites, and header declaration
3. **[Lua Binding (optional)](#lua-binding-getmitigationac)** — Expose `GetMitigationAC()` to Lua for effective AC display in `.petstats`
4. **[Build & Deploy](#building--deploying)** — Rebuild `zone.exe` and deploy
5. **[Lua Scripts](#player-commands-lua)** — Add `.petequip` and `.petstats` to `global_player.lua` (no rebuild needed)
6. **[NPC Distribution (optional)](#npc-distribution-optional)** — Give players a way to obtain satchels

Steps 1 and 5-6 don't require a rebuild. Steps 2-3 require rebuilding `zone.exe`.

---

## Database Setup

### Pet Satchel Items

Create three container items in the `items` table. These are 20-slot containers with 100% weight reduction:

```sql
-- Beastlord Pet Satchel
INSERT INTO items (id, Name, itemtype, bagslots, bagsize, bagwr, classes, races, nodrop, norent, stacksize, lore)
VALUES (800500, 'Beastlord Pet Satchel', 54, 20, 4, 100, 65535, 65535, 0, 1, 1, 'Beastlord Pet Satchel');

-- Necromancer Pet Satchel
INSERT INTO items (id, Name, itemtype, bagslots, bagsize, bagwr, classes, races, nodrop, norent, stacksize, lore)
VALUES (800501, 'Necromancer Pet Satchel', 54, 20, 4, 100, 65535, 65535, 0, 1, 1, 'Necromancer Pet Satchel');

-- Magician Pet Satchel
INSERT INTO items (id, Name, itemtype, bagslots, bagsize, bagwr, classes, races, nodrop, norent, stacksize, lore)
VALUES (800502, 'Magician Pet Satchel', 54, 20, 4, 100, 65535, 65535, 0, 1, 1, 'Magician Pet Satchel');
```

> **Note:** `itemtype=54` is a quest container. Adjust IDs to fit your server's item ID scheme. `nodrop=0` makes them NO DROP (lore items, one per character).

---

## C++ Implementation

### Core Function: `ApplyPetBagEquipment()`

Add this to `zone/pets.cpp`. It runs automatically when a pet is summoned and equips all items from the matching satchel.

```cpp
void Client::ApplyPetBagEquipment(NPC* pet)
{
    if (!pet) return;

    // Determine which class this pet belongs to
    int origin_class = GetPetOriginClass(pet, this);

    // Map class to bag item ID (adjust IDs to match your items table)
    uint32 bag_id = 0;
    switch (origin_class) {
        case 15: bag_id = 800500; break; // BST
        case 11: bag_id = 800501; break; // NEC
        case 13: bag_id = 800502; break; // MAG
        default: return; // Not a pet class
    }

    // Search general inventory and bank for the matching satchel
    struct SlotRange { int16 begin; int16 end; int16 bag_base; };
    SlotRange ranges[] = {
        { EQ::invslot::GENERAL_BEGIN, EQ::invslot::GENERAL_END, EQ::invbag::GENERAL_BAGS_BEGIN },
        { EQ::invslot::BANK_BEGIN,    EQ::invslot::BANK_END,    EQ::invbag::BANK_BAGS_BEGIN },
    };

    for (auto& range : ranges) {
        for (int slot = range.begin; slot <= range.end; slot++) {
            const EQ::ItemInstance* inst = GetInv().GetItem(slot);
            if (!inst || !inst->GetItem()) continue;
            if (inst->GetItem()->ID != bag_id) continue;
            if (!inst->IsClassBag()) continue;

            // Found the bag — iterate its contents
            int equipped = 0;
            int bag_slots = inst->GetItem()->BagSlots;
            int16 bag_base = range.bag_base +
                ((slot - range.begin) * EQ::invbag::SLOT_COUNT);

            for (int sub = 0; sub < bag_slots; sub++) {
                const EQ::ItemInstance* sub_inst = GetInv().GetItem(bag_base + sub);
                if (!sub_inst || !sub_inst->GetItem()) continue;

                const EQ::ItemData* item_data = sub_inst->GetItem();
                if (item_data->NoPet) continue;

                // Create a lootdrop entry with equip_item = 1
                auto lde = LootdropEntriesRepository::NewNpcEntity();
                lde.equip_item = 1;
                lde.item_charges = 1;
                pet->AddLootDrop(item_data, lde, true);
                equipped++;
            }

            if (equipped > 0) {
                pet->UpdateEquipmentLight();
                pet->CalcBonuses();
                Message(Chat::Green, "Pet equipped %d item(s) from your %s.",
                        equipped, inst->GetItem()->Name);
            }
            return; // Only process first matching bag found
        }
    }
}
```

> **Note on `AddLootDrop` API:** The function signature varies between EQEmu versions. Older builds use positional parameters (`AddLootDrop(item, loottable, charges, minlevel, maxlevel, equip, ...)`). Newer builds use a `LootdropEntriesRepository::NpcEntity` struct. Check your source's `zone/loot.cpp` to see which version you have. The key requirement is setting `equip_item = true/1` so items go into equipment slots, not just the loot table.

### Call Sites

Add `ApplyPetBagEquipment()` calls in two places:

**1. Pet Summoning** (`zone/pets.cpp` in `MakePoweredPet()`):

Search for `MakePoweredPet` in `zone/pets.cpp`. Near the end of the function, find the block where the pet is added to the entity list and registered with the owner. It looks like this:

```cpp
    // ... (existing code at end of MakePoweredPet) ...
    entity_list.AddNPC(npc);    // <-- pet added to world
    // ... owner->SetPetID() or AddPet() call ...

    // === ADD THIS BLOCK ===
    if (IsClient()) {
        CastToClient()->ApplyPetBagEquipment(npc);
    }
    // === END ADDITION ===
```

> **CRITICAL ordering**: Your addition MUST go AFTER both `entity_list.AddNPC()` and the pet registration call (`SetPetID` or `AddPet`). The function calls `CalcBonuses()` which calls `GetOwner()` — if the pet isn't in the entity list and registered with the owner yet, `GetOwner()` returns null and the server kills the pet.

**2. Charm Spells** (`zone/spell_effects.cpp`):

Search for `SE_Charm` or `case SE_Charm` in `zone/spell_effects.cpp`. Inside that case block, find where the charm is fully applied (after `SetPetID` or equivalent). Add the call at the end of the charm success path:

```cpp
    // ... (existing charm application code) ...

    // === ADD THIS BLOCK ===
    if (IsClient()) {
        CastToClient()->ApplyPetBagEquipment(target->CastToNPC());
    }
    // === END ADDITION ===
```

### Header Declaration

Add the function declaration to `zone/client.h`. Search for other pet-related methods (like `GetPetID` or `SetPetID`) and add nearby:

```cpp
    void ApplyPetBagEquipment(NPC* pet);
```

### Helper: `GetPetOriginClass()`

Determines which class a pet belongs to based on its summon spell. Uses `GetSpellLevel()` — a standard EQEmu function that returns the minimum level a class needs to cast a spell (returns 255 if the class can't use it).

Add to `zone/pets.cpp` and declare in `zone/mob.h`:

```cpp
// Declaration (mob.h, outside any class):
int GetPetOriginClass(Mob* pet, Client* owner = nullptr);

// Implementation (pets.cpp):
int GetPetOriginClass(Mob* pet, Client* owner)
{
    if (!pet || !pet->IsNPC()) return 0;
    if (pet->IsCharmed()) return 0;

    uint16 spell_id = pet->CastToNPC()->GetPetSpellID();
    if (spell_id == 0) return 0;

    // If owner provided, check their class first to avoid false matches
    // on spells shared by multiple pet classes
    if (owner) {
        int primary = owner->GetClass();
        if (GetSpellLevel(spell_id, primary) < 255)
            return primary;
    }

    // Fallback: check all pet classes
    for (int c : {15, 11, 13}) {  // BST, NEC, MAG
        if (GetSpellLevel(spell_id, static_cast<uint8>(c)) < 255)
            return c;
    }
    return 0;
}
```

> **Note:** `GetSpellLevel(spell_id, class_id)` returns 255 if that class cannot cast the spell. Any value less than 255 means the class has access to it.

---

## Lua Binding: GetMitigationAC (Optional)

The `.petstats` command uses `GetMitigationAC()` to show effective (combat) AC. This method exists in C++ (`Mob::GetMitigationAC()`) but is **not exposed to Lua by default** in EQEmu. You need to add the binding:

**Why this is needed:** EQEmu's `GetAC()` returns the raw base AC value from the NPC template. The actual AC used in combat is `mitigation_ac`, calculated by `ACSum()` — which includes item bonuses, defense skill, AGI, owner's pet mitigation AAs, and spell/AA buffs. Without this binding, Lua scripts can only see base AC, making it impossible to verify that satchel equipment is actually improving pet survivability.

**`zone/lua_mob.h`** — add the declaration near other AC methods (search for `GetDisplayAC`):

```cpp
int GetMitigationAC();
```

**`zone/lua_mob.cpp`** — add the implementation:

```cpp
int Lua_Mob::GetMitigationAC() {
    Lua_Safe_Call_Int();
    return self->GetMitigationAC();
}
```

And register it in the luabind scope (search for `.def("GetDisplayAC"` and add below):

```cpp
.def("GetMitigationAC", &Lua_Mob::GetMitigationAC)
```

> **Note:** If you skip this binding, replace `npc:GetMitigationAC()` with `npc:GetAC()` in the `.petstats` Lua script. You just won't see the effective AC that includes item bonuses.

---

## Player Commands (Lua)

### `.petequip` — Apply Gear to Live Pets

Add to your `global_player.lua` `event_say` handler. This lets players apply satchel gear to already-summoned pets without resummoning:

```lua
if msg:lower() == ".petequip" then
    local PET_BAGS = {[15] = 800500, [11] = 800501, [13] = 800502}
    local total_equipped = 0

    local entity_list = eq.get_entity_list()
    local npc_list = entity_list:GetNPCList()

    for npc in npc_list.entries do
        if npc:GetOwnerID() == e.self:GetID() then
            local pet_class = GetPetClass(npc)
            local bag_id = PET_BAGS[pet_class]
            if bag_id then
                local bag_slot = FindBagSlot(e.self, bag_id)
                if bag_slot >= 0 then
                    local count = 0
                    for i = 0, 19 do
                        local sub_slot = bag_slot * 100 + i
                        local item = e.self:GetItemInSlot(sub_slot)
                        if item and item > 0 then
                            npc:AddItem(item, 1, true)  -- true = equip
                            count = count + 1
                        end
                    end
                    if count > 0 then
                        e.self:Message(15, string.format(
                            "Equipped %d item(s) onto %s", count, npc:GetCleanName()))
                        total_equipped = total_equipped + count
                    end
                end
            end
        end
    end

    if total_equipped == 0 then
        e.self:Message(13, "No pet satchels found or no pets to equip.")
    end
end
```

### `.petstats` — View Pet Statistics

Shows comprehensive stats for all active pets including equipment bonuses, combat modifiers, and effective AC.

**Base AC vs Effective AC:** The `AC` value shown is the pet's base AC from its NPC template. The `Eff` (effective) value is the actual combat AC used for mitigation, calculated by `ACSum()` which adds: item AC bonuses, defense skill, AGI bonus, owner's pet mitigation AAs, and spell/AA AC. When satchel items are equipped, you'll see the effective AC jump significantly while base AC stays the same.

> **Requires:** `GetMitigationAC()` must be exposed to Lua — see [C++ Lua Binding](#lua-binding-getmitigationac) below.

```lua
if e.message:match("^%.petstats$") then
    local npc_list = eq.get_entity_list():GetNPCList()
    local pet_count = 0
    local my_id = e.self:GetID()

    for npc in npc_list.entries do
        if npc:GetOwnerID() == my_id then
            pet_count = pet_count + 1
            e.self:Message(15, "--- Pet " .. pet_count .. ": " .. npc:GetCleanName()
                .. " (Lv " .. npc:GetLevel() .. ") ---")
            e.self:Message(15, "  HP: " .. npc:GetHP() .. " / " .. npc:GetMaxHP())
            if npc:GetMaxMana() > 0 then
                e.self:Message(15, "  Mana: " .. npc:GetMana() .. " / " .. npc:GetMaxMana())
            end
            e.self:Message(15, "  AC: " .. npc:GetAC() .. " (Eff: " .. npc:GetMitigationAC()
                .. ")  ATK: " .. npc:GetATK() .. "  MinDmg: " .. npc:GetMinDMG()
                .. "  MaxDmg: " .. npc:GetMaxDMG())
            e.self:Message(15, "  STR: " .. npc:GetSTR() .. "  STA: " .. npc:GetSTA()
                .. "  DEX: " .. npc:GetDEX() .. "  AGI: " .. npc:GetAGI())
            e.self:Message(15, "  INT: " .. npc:GetINT() .. "  WIS: " .. npc:GetWIS()
                .. "  CHA: " .. npc:GetCHA())
            e.self:Message(15, "  MR: " .. npc:GetMR() .. "  FR: " .. npc:GetFR()
                .. "  CR: " .. npc:GetCR() .. "  DR: " .. npc:GetDR()
                .. "  PR: " .. npc:GetPR())
            e.self:Message(15, "  AtkSpd: " .. string.format("%.1f", npc:GetAttackSpeed())
                .. "  AtkDly: " .. npc:GetAttackDelay()
                .. "  Acc: " .. npc:GetAccuracyRating()
                .. "  Avoid: " .. npc:GetAvoidanceRating())

            -- Modifiers section (only show non-zero values)
            local haste = npc:GetHaste()
            local item_b = npc:GetItemBonuses()
            local spell_b = npc:GetSpellBonuses()

            local mods = {}
            if haste ~= 0 then table.insert(mods, "Haste: " .. haste .. "%") end

            local ds = spell_b:DamageShield()
            if ds ~= 0 then table.insert(mods, "DmgShield: " .. ds) end

            local dbl = item_b:DoubleAttackChance() + spell_b:DoubleAttackChance()
            if dbl ~= 0 then table.insert(mods, "DblAtk: " .. dbl .. "%") end

            local tri = item_b:TripleAttackChance() + spell_b:TripleAttackChance()
            if tri ~= 0 then table.insert(mods, "TriAtk: " .. tri .. "%") end

            local flurry = item_b:FlurryChance() + spell_b:FlurryChance()
            if flurry ~= 0 then table.insert(mods, "Flurry: " .. flurry .. "%") end

            local crit = item_b:CriticalHitChance(0) + spell_b:CriticalHitChance(0)
            if crit ~= 0 then table.insert(mods, "Crit: " .. crit .. "%") end

            local parry = item_b:ParryChance() + spell_b:ParryChance()
            if parry ~= 0 then table.insert(mods, "Parry: " .. parry .. "%") end

            local dodge = item_b:DodgeChance() + spell_b:DodgeChance()
            if dodge ~= 0 then table.insert(mods, "Dodge: " .. dodge .. "%") end

            local ripo = item_b:RiposteChance() + spell_b:RiposteChance()
            if ripo ~= 0 then table.insert(mods, "Ripo: " .. ripo .. "%") end

            local hh = item_b:HundredHands() + spell_b:HundredHands()
            if hh ~= 0 then table.insert(mods, "HndHnds: " .. hh) end

            local proc = item_b:ProcChance() + spell_b:ProcChance()
            if proc ~= 0 then table.insert(mods, "ProcRate: " .. proc .. "%") end

            local dmgmod = item_b:DamageModifier(0) + spell_b:DamageModifier(0)
            if dmgmod ~= 0 then table.insert(mods, "DmgMod: " .. dmgmod .. "%") end

            if #mods > 0 then
                e.self:Message(18, "  Modifiers: " .. table.concat(mods, " | "))
            end
        end
    end

    if pet_count == 0 then
        e.self:Message(13, "You have no pets summoned.")
    else
        e.self:Message(18, pet_count .. " pet(s) found.")
    end
end
```

---

## NPC Distribution (Optional)

You can distribute satchels via an NPC. Example Perl script for a vendor NPC:

```perl
# In event_say handler
if ($text =~ /satchel/i) {
    my %PET_BAG = (15 => 800500, 11 => 800501, 13 => 800502);
    my $class = $client->GetClass();

    if (exists $PET_BAG{$class}) {
        $client->SummonItem($PET_BAG{$class});
        $client->Message(15, "Here is your pet satchel. Place equipment inside — it will auto-equip on your next pet summon.");
    } else {
        $client->Message(13, "Only Beastlords, Necromancers, and Magicians can use pet satchels.");
    }
}
```

---

## Player Flow

1. **Get satchel** — say "satchel" to the NPC vendor (or however you distribute them)
2. **Fill it** — place weapons, armor, and other gear into the satchel container
3. **Summon pet** — gear auto-equips instantly on summon
4. **Change gear** — swap items in the satchel, then use `.petequip` to re-equip live pets
5. **Check stats** — use `.petstats` to verify equipment bonuses are applied

---

## Building & Deploying

After making the C++ changes, you need to rebuild `zone.exe` (the zone server binary). The other binaries (`world.exe`, `shared_memory.exe`, etc.) are unchanged.

### Prerequisites

- **CMake** (3.18+)
- **C++ compiler**: Visual Studio 2022 (Windows) or GCC (Linux)
- EQEmu source code with CMake build configured

### Build Steps (Windows — Visual Studio)

```bash
# From your EQEmu source directory:
cmake --preset <your-preset>            # Only needed first time or after CMakeLists.txt changes
cmake --build build/<preset>/  --config Release --target zone
```

Or open the generated `.sln` in Visual Studio, set configuration to Release, right-click the `zone` project, and Build.

### Build Steps (Linux — GCC)

```bash
# From your EQEmu source directory:
mkdir -p build && cd build
cmake ..
make zone -j$(nproc)
```

### Deploy

1. **Stop the server** (or at minimum stop all zone processes)
2. Copy the new `zone.exe` (or `zone` on Linux) to your server's `bin/` directory
3. Restart the server

The Lua scripts (`.petequip`, `.petstats`) don't require a rebuild — just place them in your `quests/global/` directory and use `#reloadquest` in-game or restart.

The SQL for satchel items can be run at any time — use `#reloadstatic` in-game to pick up new items without restart.

## Tools & Requirements

- **EQEmu Server** with source access (C++ modifications required)
- **CMake** (3.18+) for build configuration
- **C++ compiler** (Visual Studio 2022 or GCC) for building zone.exe
- **MySQL/MariaDB** for item creation
- **Lua** for player commands (`.petequip`, `.petstats`)
- No client-side modifications needed

## Files Modified

| File | Changes |
|------|---------|
| `zone/pets.cpp` | `ApplyPetBagEquipment()`, `GetPetOriginClass()` |
| `zone/spell_effects.cpp` | Charm auto-equip call |
| `zone/client.h` | `ApplyPetBagEquipment()` declaration |
| `zone/lua_mob.h` | `GetMitigationAC()` declaration |
| `zone/lua_mob.cpp` | `GetMitigationAC()` implementation + luabind `.def()` |
| `quests/global/global_player.lua` | `.petequip`, `.petstats` commands |

## Known Limitations

- Items with `NoPet=1` flag are skipped during auto-equip
- Satchels are searched in inventory (slots 23-32) then bank (2000-2023)
- Charmed pets receive equipment but are classless (uses all three bags if found)
- **`.petequip` is visual only** — `CalcBonuses()` is not exposed to Lua, so `.petequip` equips items visually but does not recalculate AC, ATK, or other stats. Full stat bonuses are only applied on pet summon (C++ `ApplyPetBagEquipment`) and on zoning (pet restore). To get full stats from new gear, resummon your pet.

---

## License

This implementation is shared for the EQEmu community. Adapt freely for your server.
