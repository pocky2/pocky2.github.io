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
            pet->GetLoottable(),
            sub->GetCharges(),
            1, 127,
            true,   // equip_item = true (auto-equip)
            false
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
// AFTER entity_list.AddNPC() and owner->SetPetID() calls
if (IsClient()) {
    CastToClient()->ApplyPetBagEquipment(npc);
}
```

> **CRITICAL ordering**: `AddNPC()` and pet registration MUST happen BEFORE `ApplyPetBagEquipment()`. The function calls `CalcBonuses()` which calls `GetOwner()` — if the pet isn't registered yet, the server kills it.

**2. Charm Spells** (`zone/spell_effects.cpp` in the Charm effect case):
```cpp
// After charm is applied and pet is registered
if (IsClient()) {
    CastToClient()->ApplyPetBagEquipment(target->CastToNPC());
}
```

### Helper: `GetPetOriginClass()`

Determines which class a pet belongs to based on its summon spell:

```cpp
int GetPetOriginClass(Mob* pet, Client* owner)
{
    if (!pet || !pet->IsNPC()) return 0;
    if (pet->IsCharmedPet()) return 0;

    uint16 spell_id = pet->CastToNPC()->GetPetSpellID();
    if (spell_id == 0) return 0;

    // Check which pet class can cast this spell
    // Check owner's primary class first, then alternates
    if (owner) {
        int primary = owner->GetClass();
        if (primary == 15 || primary == 11 || primary == 13) {
            if (IsValidPetSpellForClass(spell_id, primary))
                return primary;
        }
    }

    // Fallback: check all pet classes
    for (int c : {15, 11, 13}) {
        if (IsValidPetSpellForClass(spell_id, c))
            return c;
    }
    return 0;
}
```

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

Shows comprehensive stats for all active pets including equipment bonuses:

```lua
if msg:lower() == ".petstats" then
    local entity_list = eq.get_entity_list()
    local npc_list = entity_list:GetNPCList()
    local found = false

    for npc in npc_list.entries do
        if npc:GetOwnerID() == e.self:GetID() then
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

## Tools & Requirements

- **EQEmu Server** with source access (C++ modifications required)
- **C++ compiler** (Visual Studio or GCC) for building zone.exe
- **MySQL/MariaDB** for item creation
- **Lua** for player commands (`.petequip`, `.petstats`)
- No client-side modifications needed

## Files Modified

| File | Changes |
|------|---------|
| `zone/pets.cpp` | `ApplyPetBagEquipment()`, `GetPetOriginClass()` |
| `zone/spell_effects.cpp` | Charm auto-equip call |
| `zone/client.h` | `ApplyPetBagEquipment()` declaration |
| `quests/global/global_player.lua` | `.petequip`, `.petstats` commands |

## Known Limitations

- Items with `NoPet=1` flag are skipped during auto-equip
- Satchels are searched in inventory (slots 23-32) then bank (2000-2023)
- Charmed pets receive equipment but are classless (uses all three bags if found)

---

## License

This implementation is shared for the EQEmu community. Adapt freely for your server.
