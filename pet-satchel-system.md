# Pet Satchel System for EQEmu

A custom pet equipment system for EQEmu servers that allows players to pre-load gear into specialized containers. When a pet is summoned, the gear is automatically equipped on the pet.

## Overview

The Pet Satchel System adds three container items (one per pet class) that auto-equip their contents onto summoned pets. It integrates with the multi-pet system and supports crystal class multi-classing.

**Pet Classes Supported:**
- Beastlord (BST)
- Necromancer (NEC)
- Magician (MAG)

## Features

- **Auto-equip on summon** - pets are geared instantly when summoned
- **Live re-equip** - `.petequip` command applies gear to already-summoned pets
- **Pet stats inspection** - `.petstats` shows all pet stats including equipment effects
- **Multi-pet aware** - each pet gets gear from its class-specific satchel
- **Charm support** - charmed pets also receive satchel equipment
- **Crystal class compatible** - works with multi-class crystal systems

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

    // Map class to bag item ID
    uint32 bag_id = 0;
    switch (origin_class) {
        case 15: bag_id = 800500; break; // BST
        case 11: bag_id = 800501; break; // NEC
        case 13: bag_id = 800502; break; // MAG
        default: return; // Not a pet class
    }

    // Search inventory slots 23-32 (general) and 2000-2023 (bank) for the bag
    const EQ::ItemInstance* bag = nullptr;
    int bag_slot = -1;

    // Check general inventory
    for (int slot = EQ::invslot::GENERAL_BEGIN; slot <= EQ::invslot::GENERAL_END; slot++) {
        const EQ::ItemInstance* item = GetInv().GetItem(slot);
        if (item && item->GetID() == bag_id) {
            bag = item;
            bag_slot = slot;
            break;
        }
    }

    // Check bank if not found
    if (!bag) {
        for (int slot = EQ::invslot::BANK_BEGIN; slot <= EQ::invslot::BANK_END; slot++) {
            const EQ::ItemInstance* item = GetInv().GetItem(slot);
            if (item && item->GetID() == bag_id) {
                bag = item;
                bag_slot = slot;
                break;
            }
        }
    }

    if (!bag) return;

    // Equip each item from the bag onto the pet
    int equipped = 0;
    for (int i = 0; i < bag->GetItem()->BagSlots && i < 20; i++) {
        const EQ::ItemInstance* sub = bag->GetItem(i);
        if (!sub || !sub->GetItem()) continue;

        // Skip NoPet items
        if (sub->GetItem()->NoPet) continue;

        pet->AddLootDrop(
            sub->GetItem(),
            pet->GetLoottable(),  // loot table reference
            sub->GetCharges(),    // charges
            1,                    // min (unused)
            127,                  // max (unused)
            true,                 // equip_item = true (auto-equip)
            false                 // not a quest item
        );
        equipped++;
    }

    if (equipped > 0) {
        pet->UpdateEquipmentLight();
        pet->CalcBonuses();
        Message(Chat::Yellow, "Equipped %d item(s) from %s onto %s.",
                equipped, bag->GetItem()->Name, pet->GetCleanName());
    }
}
```

### Call Sites

Add `ApplyPetBagEquipment()` calls in two places:

**1. Pet Summoning** (`zone/pets.cpp` in `MakePoweredPet()`):
```cpp
// AFTER AddNPC() and owner->AddPet() — CalcBonuses needs the pet registered
if (IsClient()) {
    CastToClient()->ApplyPetBagEquipment(npc);
}
```

> **CRITICAL**: The `AddNPC()` and `AddPet()` calls MUST happen BEFORE `ApplyPetBagEquipment()`. The function calls `CalcBonuses()` which calls `GetOwner()` which checks `IsMyPet()`. If the pet isn't registered in the owner's pet list yet, it gets killed instantly.

**2. Charm Spells** (`zone/spell_effects.cpp` in the Charm case):
```cpp
// After charm is applied and pet is registered
if (IsClient()) {
    CastToClient()->ApplyPetBagEquipment(target->CastToNPC());
}
```

### Helper: `GetPetOriginClass()`

Determines which class a pet belongs to (needed for multi-class scenarios):

```cpp
int GetPetOriginClass(Mob* pet, Client* owner)
{
    if (!pet || !pet->IsNPC()) return 0;
    if (pet->IsCharmedPet()) return 0; // Charmed pets are classless

    uint16 spell_id = pet->IsNPC() ? pet->CastToNPC()->GetPetSpellID() : 0;
    if (spell_id == 0) return 0;

    // Check which of the owner's active classes can cast this pet spell
    // Priority: primary > virtual > crystal classes
    if (owner) {
        int primary = owner->GetClass();
        // Check if primary class can use this spell
        if (IsValidPetSpellForClass(spell_id, primary))
            return primary;
        // Check virtual/crystal classes similarly...
    }

    // Fallback: check all pet classes
    for (int c : {15, 11, 13}) { // BST, NEC, MAG
        if (IsValidPetSpellForClass(spell_id, c))
            return c;
    }
    return 0;
}
```

---

## Lua Commands

### `.petequip` - Live Pet Equipment

Add to your `global_player.lua` `event_say` handler:

```lua
if msg:lower() == ".petequip" then
    -- Map class to bag item ID
    local PET_BAGS = {[15] = 800500, [11] = 800501, [13] = 800502}
    local total_equipped = 0

    -- Find all pets owned by this player
    local entity_list = eq.get_entity_list()
    local npc_list = entity_list:GetNPCList()

    for npc in npc_list.entries do
        if npc:GetOwnerID() == e.self:GetID() then
            local pet_class = GetPetClass(npc)  -- determine pet's class
            local bag_id = PET_BAGS[pet_class]
            if bag_id then
                -- Find bag in inventory (slots 23-32) or bank (2000-2023)
                local bag_slot = FindBagSlot(e.self, bag_id)
                if bag_slot >= 0 then
                    local count = 0
                    for i = 0, 19 do
                        local sub_slot = bag_slot * 100 + i  -- sub-item offset
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

### `.petstats` - Pet Statistics Display

Shows comprehensive stats for all active pets:

```lua
if msg:lower() == ".petstats" then
    local entity_list = eq.get_entity_list()
    local npc_list = entity_list:GetNPCList()
    local found = false

    for npc in npc_list.entries do
        if npc:GetOwnerID() == e.self:GetID() and not npc:GetBodyType() == 21 then
            found = true
            local name = npc:GetCleanName()
            e.self:Message(15, "--- " .. name .. " ---")
            e.self:Message(15, string.format(
                "Level: %d | HP: %d/%d | Mana: %d/%d",
                npc:GetLevel(), npc:GetHP(), npc:GetMaxHP(),
                npc:GetMana(), npc:GetMaxMana()))
            e.self:Message(15, string.format(
                "AC: %d | ATK: %d | DMG: %d-%d",
                npc:GetAC(), npc:GetATK(),
                npc:GetMinDMG(), npc:GetMaxDMG()))
            e.self:Message(15, string.format(
                "STR:%d STA:%d DEX:%d AGI:%d INT:%d WIS:%d CHA:%d",
                npc:GetSTR(), npc:GetSTA(), npc:GetDEX(), npc:GetAGI(),
                npc:GetINT(), npc:GetWIS(), npc:GetCHA()))
            e.self:Message(15, string.format(
                "MR:%d FR:%d CR:%d DR:%d PR:%d",
                npc:GetMR(), npc:GetFR(), npc:GetCR(),
                npc:GetDR(), npc:GetPR()))
            e.self:Message(15, string.format(
                "Attack Speed: %.1f | Delay: %d",
                npc:GetAttackSpeed(), npc:GetAttackDelay()))
        end
    end

    if not found then
        e.self:Message(13, "You have no active pets.")
    end
end
```

### `.pethelp` - Pet Command Reference

Can be implemented as either a GM command (C++) or Lua. Shows available `/pet` commands:

```
--- Multi-Pet Commands (affect ALL pets) ---
/pet attack       - All pets attack your target
/pet back off     - All pets stop attacking
/pet follow me    - All pets follow you
/pet guard here   - All pets guard current spot
/pet guard me     - All pets guard near you
/pet sit down     - All pets sit (regen)
/pet stand up     - All pets stand
/pet hold         - All pets hold (don't auto-aggro)
/pet taunt        - Toggle taunt on all pets
/pet no cast      - Toggle casting on all pets
/pet get lost     - Dismiss FOCUSED pet only

Max 1 pet per active class, 3 total.
```

---

## Multi-Pet System Integration

The Pet Satchel System works with the multi-pet architecture. Key structural changes:

### mob.h - Pet ID Tracking

```cpp
// Replace single pet ID with a vector
std::vector<uint16> petids;       // All active pet entity IDs
uint16 focused_pet_id = 0;        // Currently focused pet

// Key methods
bool AddPet(uint16 pet_id);
void RemovePet(uint16 pet_id);
std::vector<Mob*> GetAllPets();
bool IsMyPet(Mob* mob);
void ValidatePetList();
```

### Pet Limits

```cpp
// In ruletypes.h
RULE_INT(Pets, AbsolutePetLimit, 3, "Maximum total pets per player")
```

- One pet per active class (BST pet + NEC pet + MAG pet)
- Familiars don't count toward the limit
- Charm counts toward the limit (classless)
- Suspend Minion is disabled (multi-pet replaces it)

### Pet Command Broadcasting

All `/pet` commands in `client_packet.cpp` are broadcast to ALL pets:

```cpp
// In OPPetCommand handler:
auto all_pets = GetAllPets();
for (auto* pet : all_pets) {
    // Apply command to each pet
    pet->Say("command...");
}
// Exception: /pet get lost only affects focused pet
```

### Pet Buff Broadcasting

ST_Pet (targettype 14) spells auto-apply to ALL owned pets:

```cpp
// In spell_effects.cpp, when casting a pet-target spell:
auto all_pets = GetAllPets();
for (auto* pet : all_pets) {
    SpellFinished(spell_id, pet);  // Cast on each pet
}
```

### Pet Persistence (Zoning)

Pets are saved before zone cleanup and restored on zone-in:

- DB tables: `character_pet_info`, `character_pet_buffs`, `character_pet_inventory`
- `pet` column uses 0-2 for up to 3 pets
- Saved in `DoZoneSuccess()` BEFORE cleanup (flag `m_pets_saved_for_zone` prevents destructor overwrite)

---

## Tools & Requirements

- **EQEmu Server** with source access (C++ modifications required)
- **CMake + Visual Studio** (or GCC) for building zone.exe
- **MySQL/MariaDB** for item creation
- **Lua** for quest-side commands (`.petequip`, `.petstats`)
- No client-side modifications needed

## Files Modified

| File | Changes |
|------|---------|
| `zone/pets.cpp` | `ApplyPetBagEquipment()`, `GetPetOriginClass()`, multi-pet logic |
| `zone/mob.h` | `petids` vector, `focused_pet_id`, pet management methods |
| `zone/mob.cpp` | Pet list management implementation |
| `zone/spell_effects.cpp` | Charm auto-equip, pet buff broadcasting |
| `zone/client_packet.cpp` | Pet command broadcasting to all pets |
| `zone/client.h` | Method declarations |
| `zone/client.cpp` | Pet persistence, zoning support |
| `zone/zoning.cpp` | Save/restore multi-pet state |
| `zone/zonedb.cpp` | Multi-pet DB read/write (pet column 0-2) |
| `zone/ruletypes.h` | `Pets:AbsolutePetLimit` rule |
| `quests/global/global_player.lua` | `.petequip`, `.petstats` commands |
| `quests/nexus/Spell_Weaver.pl` | Satchel acquisition NPC |

---

## Known Limitations

- RoF2 client pet window focus swap doesn't work mid-session (architectural limitation)
- Items with `NoPet=1` flag are skipped during auto-equip
- Pet satchels search inventory slots 23-32 and bank slots 2000-2023 only
- Crystal class spell clones (IDs 43000-44999) require special handling in `GetPetOriginClass()`

---

## License

This implementation is shared for the EQEmu community. Adapt freely for your server.
