# Agent Prompt: Game Asset Documentation from Unreal Engine Binary Files

## What You're Doing

You have a collection of Unreal Engine `.uasset` data asset files for a medieval village-builder/survival. **There are NO companion `.uexp` files** — all data lives inside the `.uasset` files themselves.

Your job: parse every data asset, extract all useful gameplay data, and produce a single well-organized `GameAssets.md`. Every asset should appear exactly once, properly categorized, with its name, description, crafting recipe (including quantities), and relevant numeric stats.

---

## Binary Parsing Guidance

You'll need to write Python scripts to parse the binary serialization format. Here are the key gotchas that took us a long time to figure out:

### Name Table
The file contains a name table — sequential `FNameEntry` records: `[int32 length][ASCII string][null byte][uint32 hash]`. Build this table first; everything else references it by index.

**Finding the name table**: The UE5 package header contains a `PackageName` FString (an `int32` length followed by that many ASCII bytes) that starts with `/Game/` or another engine path. After the PackageName string, the next fields are `PackageFlags` (uint32), `NameCount` (int32), and `NameOffset` (int32). The header layout varies slightly between UE versions — the safest approach is to scan the first ~800 bytes for a FString starting with `/`, then read the three fields immediately after. NameCount tells you how many entries to parse; NameOffset tells you where the name table starts in the file.

### Serialized Property Tags
Each property is serialized as:
```
FName Name       (8 bytes — index + instance number)
FName Type       (8 bytes)
int32 ArrayIndex (4 bytes)
int32 DataSize   (4 bytes)   ← at offset i+20 from tag start
uint8 HasGuid    (1 byte)
ValueData        (DataSize bytes)
```

**The single most important thing**: the `DataSize` field is at byte offset **i+20**, not i+16. This is a non-obvious layout detail that will silently corrupt all your value extraction if you get it wrong. Double-check this.

To extract property values, search the binary data for FName patterns matching known property names (like `Worth`, `Level`, `Damage`). Each FName is 8 bytes: `[int32 name_table_index][int32 instance_number=0]`. When you find a property name FName followed by a valid type FName (`IntProperty`, `FloatProperty`, `DoubleProperty`, etc.), you've found a property tag — read the value at offset +25 from the tag start.

**For properties that appear multiple times** (e.g., `Level` or `Damage` can appear inside nested sub-structures like attack montages): take the **last** occurrence, which is typically the root-level property, not a nested sub-property.

### Health is a StructProperty
`Health` is stored as a `StructProperty`, NOT a simple numeric type. It contains a nested `MaxValue` FloatProperty with the actual HP value. To extract it: find the `Health` + `StructProperty` tag, then search inside the struct's data range for a `MaxValue` + `FloatProperty` tag and read its float value. As a fallback, scan the struct data for reasonable float values in the 10–50000 range.

### Import Table & Crafting Resources
The import table has **40-byte entries** with the `ObjectName` FName at **offset +20** within each entry. To find the import table, look for `ImportCount` and `ImportOffset` in the header fields that follow a localization GUID FString (a 32-character hex string stored as a FString) after the name table fields.

Import references are encoded as negative integers: a value of `-3` means import index `2` (formula: `-(value + 1)`).

Crafting materials are stored in a `Resources` MapProperty. To extract quantities:
1. Find the `Resources` + `MapProperty` FName pair (16 bytes total)
2. After the tag header (+25 bytes) come the KeyType and ValueType FNames (+16 bytes), then map metadata
3. Scan forward from the tag start for a small positive integer (1–20) immediately followed by negative integers — this is the entry count followed by `[import_ref, quantity]` pairs
4. Each entry is 8 bytes: `[int32 negative_import_ref][int32 quantity]`
5. Resolve the import ref to a DA_ asset name using the import table

**The quantities from this map are the ground truth** — if you're getting different quantities from a different extraction method, trust the Resources MapProperty.

**Fallback**: If the MapProperty parsing fails for an asset, extract DA_ resource references from the name table paths (entries containing `/Items/Resource/DA_`). These give you the materials without quantities.

### What to Extract
From each asset, pull out:
- **Display name** and **description** (see Display Names section below)
- **Gameplay tags** from the name table — strings like `Item.Weapon.Sword`, `Buildings.Crafting.Blacksmith`, `World.Biome.Meadow`, etc.
- **Numeric stats**: Worth, Durability, Weight, Level, Damage, Health, Speed — whatever is present. Ignore SkillLevel and CraftingTime for items (CraftingTime only matters for buildings).
- **Crafting materials with quantities** from the Resources map
- **Crafting station** from gameplay tags matching `Buildings.Crafting.Equipment.*` or `Buildings.Crafting.Resources.*` or `Buildings.Crafting.Food.*` — the last segment of the tag is the station name
- **All references to other assets** — DA_ prefixed names in the name table encode relationships. An animal asset references what it drops (DA_RawSteak, DA_WolfFur, etc.). A job asset's crafting tags indicate what buildings/tools it needs. A building's tags encode placement rules.

### Display Names
Many asset filenames are cryptic (`DA_SwordCopper`, `DA_AU_Prince`, `DA_ClubWood`). Display names and descriptions are stored near the end of the file in a predictable pattern involving GUIDs:

**Pattern 1 — Two bracket-GUIDs**: `[HEX32]` → hex → **Name** → `[HEX32]` → hex → **Description**
**Pattern 2 — One bracket-GUID**: `[HEX32]` → hex → **Name** (or **Description** if long), then check `{Text}/Text/actual_text` blocks for the other
**Pattern 3 — {Text} blocks only**: Some assets use `{Text}\nText\nActual Text` blocks (common in foliage/farm assets)

Use the `strings` command to extract readable text, then locate bracket-GUID markers (`[32-hex-char]`) and readable text near them. Short readable text after a GUID is the name; long readable text is the description.

**Validation**: Reject names that are system strings (PrimaryAssetType, ItemEquipable, BlueprintGeneratedClass, etc.), hex GUIDs, or contain special characters (@, >, <, =, {, etc.). If no valid name is found, derive one from the filename by stripping `DA_`, `DA_BLD_`, `DA_FarmPlant_` prefixes and inserting spaces at camelCase boundaries.

---

## Filtering: What to Include and Exclude

Not every `.uasset` file is a gameplay data asset. You must be selective:

### Include
- Player-usable items (weapons, tools, armor, consumables, resources, seeds, food)
- Enemies that the player fights (hostile NPCs with combat stats)
- Animals (wildlife and livestock — bears, wolves, deer, chickens, cows, pigs, sheep, etc.)
- Biome region definitions (the major world zones)
- Player-assignable villager jobs
- Player-buildable structures (walls, floors, furniture, crafting stations, etc.)
- Crops (farm plants growing in the ground)
- World resources (harvestable foliage objects — trees, boulders, bushes)

### Exclude by Other Signals
- **NPC-only clothing/cosmetics**: assets for villager or NPC outfits (things like "VillagerShirt", "DesertVillageLeaderHat", etc.) that are not equippable by the player. A strong signal is `Level: 99` — this typically marks internal/NPC-only items. If an armor piece has Level 99 and a name suggesting it's for an NPC, exclude it.
- **Internal biome sub-layers**: the game has many internal biome subdivisions used for terrain generation (dirt layers, rocky ground variants, lake shores, river beds, mine veins, POI building zones, cemetery sub-areas, test maps). These are NOT the gameplay biomes. Only include the top-level biome regions (meadow, forest, desert, jungle, mountain, etc.).
- **Vegetation spawner/cluster configs**: internal data controlling how groups of trees or bushes spawn procedurally (things like "Forest Trees Cluster", "Meadow Bushes Cluster", "Grass Medium", "Default Empty Vegetation"). These are NOT the harvestable world resources. Only include the actual individual harvestable objects (the tree, the boulder, the bush).
- **Duplicate entries**: if the same logical asset appears multiple times (e.g., same enemy with the same data under different paths), only include it once. Deduplicate by display name within each category.

---

## The 9 Categories

Classify every asset into exactly one of these categories (except Vendors, which is static data — see below). Use the folder structure and gameplay tags to figure out where each asset belongs. The categories are:

### Items
The broadest category — anything a player can hold in their inventory. Weapons, tools, armor, consumables, accessories, and all inventory resources (ores, ingots, gems, leather, fur, fabric, seeds, wood, food, etc.). If it exists in the player's inventory rather than being placed in the world or built, it's an Item.

For each item show: Name, Description, Level, Worth, Durability, Weight, crafting recipe with quantities, and crafting station. **Do NOT show Crafting Time for items** — it clutters the output and isn't player-facing for inventory items.

### Enemies
Hostile NPCs. Extract combat stats: Level, Health, and Damage. Also show which biomes they are found in (from `World.Biome.*` tags). **Do not include crafting materials or "Crafted using" for enemies.** Do not include drops — enemy loot tables are managed separately.

- **Drops** — extracted from `LootOverrides` data in the asset. Look for DA_ references to items like `DA_RawSteak`, `DA_WolfFur`, `DA_DeerFur`, `DA_Feather`, `DA_RawMeatShank`, `DA_BoarFur`, `DA_Wool`, `DA_Egg`, `DA_Milk` etc. in the name table. Resolve these to human-readable names and show them as `- Drops: Raw Steak, Wolf Fur`

**Every enemy entry must have data beyond just a name.** If you're seeing enemies with only a name and nothing else (no description, no level, no health), your extraction is incomplete — go back and extract more from those assets.

### Animals
Wildlife and domesticated animals. Group by species. For each animal, extract and show:
- **Name** and **Description**
- **Level**, **Health** (from StructProperty → MaxValue), and **Damage**
- **Drops** — extracted from `LootOverrides` data in the asset. Look for DA_ references to items like `DA_RawSteak`, `DA_WolfFur`, `DA_DeerFur`, `DA_Feather`, `DA_RawMeatShank`, `DA_BoarFur`, `DA_Wool`, `DA_Egg`, `DA_Milk` etc. in the name table. Resolve these to human-readable names and show them as `- Drops: Raw Steak, Wolf Fur`

**Do NOT show crafting materials or crafting stations for animals.**

**This is a real category with real entries.** The game has bears, wolves, deer, boars, chickens, cows, pigs, sheep, foxes, rabbits, and more. If your Animals section is empty or missing, you haven't found the animal data assets — look harder. They exist in `MGF_Animals/DataAssets/` alongside the main game assets.

### Biomes
Biome region definitions describing the world zones (meadow, forest, desert, jungle, mountain, etc.). These are found in `BiomeVegetation` folders — extract the name and description from each.

**Only include the top-level biome regions.** Do NOT include internal biome sub-layers like dirt/rocky ground variants, lake/river zones, mine veins, cemetery sub-areas, or building zones within POIs. Those are engine internals, not gameplay biomes.

### Jobs
Villager job roles (lumberjack, miner, blacksmith, etc.). Show what buildings and tools each job uses — extracted from gameplay tags matching `Buildings.Crafting.*`. Display these as `- Uses: Blacksmith, Granite Smithy, Grindstone` (NOT "Crafted at").

Only include player-assignable villager jobs — not internal sub-roles or NPC-only jobs.

### Buildings
Everything the player can build. Identify buildings by having `BLD` in their asset name. This is a big category — walls, floors, foundations, roofs, furniture, crafting stations, storage, decorations, fences, mechanical systems, etc.

For buildings show: Name, Description, crafting materials (from resource references), and Placement rules (from `Buildings.Place.*` tags like InsideBuilding, OutsideBuilding, MustBeSnapped). **Do not show `Crafted at` for buildings** — buildings are not crafted at stations. **Do not show `Level` for buildings** — ignore `CharacterLevel`.

**Important**: there should be ONE buildings category. Don't split "villager buildings" into a separate section — they're all just buildings.

### Crops
The growing farm plants themselves — the data assets that define a plant growing in a farm plot (e.g., the wheat plant, the pumpkin plant, the berry bush). Not the seeds you carry in your inventory to plant them, and not the harvested produce — just the plant while it's in the ground. Found in `DAFarmPlants` folders.

### World Resources
Physical objects placed in the game world that players walk up to and harvest — trees you chop down, boulders you mine, bushes you pick, fiber plants you pull, fallen logs you break apart. These are found in `FoliageDataAssets` folders.

For world resources, show any DA_ resource references as `- Yields: Timber` (NOT "Crafted using" — players don't craft these, they harvest them). Also show Durability where present.

**Only include individual harvestable objects** — not vegetation cluster spawner configs, grass types, or procedural generation data.

### Vendors
This is **not** extracted from `.uasset` files. Vendor data comes from the project's `TradingDeveloperSettings` config (typically in a `DefaultGame.ini` or similar config file under `[/Script/Solidriver.TradingDeveloperSettings]`). This data will be provided separately.

For each vendor, show:
- `Required Level:` (from `VendorToLevelMapping`) — NOT "Unlock Level" or "Occupation Tag"
- `Sells:` — human-readable list using **only the last segment** of each tag. For example, `Item.Consumable.Baked` becomes just `Baked`. `Buildings.Crafting.Equipment.Workbench` becomes `Workbench`.
- `Excludes:` — same tag simplification (last segment only)
- `Item Probability:` — only if less than 100%
- `Extension Rarity Chances:` — Legendary/Epic/Rare/Uncommon percentages, only if any are non-zero

**Do NOT include**: global vendor settings (default money, money scaling, sell price multiplier), occupation tags, or full tag paths.

---

## Deduplicating Variants

You'll find assets with letter or number suffixes — like `DA_TreeA`, `DA_TreeB`, `DA_TreeE` or `DA_Meadow_Bush01` through `DA_Meadow_Bush05`. When multiple assets share identical gameplay data and only differ in which 3D mesh they use, collapse them into a single entry with a clean name (e.g., just "Meadow Tree" instead of showing A/B/E separately). This applies to foliage, boulders, logs, and decorative building variants.

**However**, not all numbered suffixes are visual variants. Some represent distinct gameplay tiers — for example, building pieces with numbered suffixes where each number is a different material tier that will eventually have different names and properties. The rule: compare the actual data. If properties are identical, collapse. If they differ (or represent intentionally separate tiers), keep them as individual entries.

For World Resources, deduplicate by display name AND properties — assets with the same name and identical stats are visual variants and should be collapsed.

For all other categories, deduplicate by display name — if two assets parse to the same display name, keep only the first.

---

## Output Format

Produce a single `GameAssets.md` file. No metadata, no preamble — straight to the content.

The structure is:

```markdown
# Game Assets

## Category Name

- Name: Raw Extracted Name
- Description: The item's description text
- Level: 4
- Worth: 42
- Durability: 160
- Weight: 2
- Crafted using Copper Ingot x2, Deer Leather x1, Pinewood x1
- Crafted at: Granite Smithy
### Interpreted Display Name
```

### Category-Specific Formatting

**Items:**
```
### Copper Bite
- Name: Copper Bite
- Description: A simple copper sword...
- Level: 4
- Worth: 42
- Durability: 160
- Weight: 2
- Crafted using Copper Ingot x2, Deer Leather x1, Pine Timber x1
- Crafted at: Granite Smithy
```
No Crafting Time. Stats order: Level, Worth, Durability, Weight.

**Enemies:**
```
### Troll
- Name: Troll
- Description: Huge and unyielding...
- Level: 8
- Health: 1500
- Damage: 50
- Drops: Coin
- Found in: DarkForest
```
No crafting info. Stats order: Level, Health, Damage. Show biome from `World.Biome.*` tags.

**Animals:**
```
### Bear Brown
- Name: Bear Brown
- Description: Powerful and unpredictable...
- Level: 6
- Health: 500
- Damage: 60
- Drops: Raw Steak
```
No crafting info. Stats order: Level, Health, Damage. Drops from LootOverrides DA_ refs.

**Jobs:**
```
### Blacksmith
- Name: Blacksmith
- Description: Uses an anvil to craft equipment, tools and weapons
- Uses: Blacksmith, Granite Smithy, Grindstone
```
Use `Uses:` NOT `Crafted at:`. Buildings/tools from `Buildings.Crafting.*` tags.

**Buildings:**
```
### Furnace
- Name: Furnace
- Description: Melt all kinds of ores to make ingots
- Crafted using Clay Brick, Copper Ingot
- Placement: InsideBuilding, MustBeSnapped, OutsideBuilding
```
No Crafted at. No Level. Show Placement from `Buildings.Place.*` tags.

**World Resources:**
```
### Oak Tree
- Name: Oak Tree
- Description: A tall tree with rough bark...
- Durability: 200
- Yields: Timber
```
Use `Yields:` NOT `Crafted using`. Shows what harvesting the object produces.

**Biomes:**
```
### Desert
- Name: Desert
- Description: A land of endless sand and burning light...
```

**Crops:**
```
### Wheat Plant
- Name: Wheat Plant
- Description: A growing stand of wheat...
```

**Vendors:**
```
### Blacksmith
- Required Level: 1
- Sells: Blacksmith, Granite Smithy, Weapon
- Excludes: Ingot, Ore, Accessory, Projectile, Workbench
- Extension Rarity Chances: Legendary: 5%, Epic: 10%, Rare: 20%, Uncommon: 30%
```
No occupation tags. No global settings. Tags use only their last segment.

### General Formatting Rules
- Each asset gets its own `###` heading with the best display name available
- Always include a `- Name:` bullet with the raw extracted name
- Only include bullets for data that actually exists — skip empty fields
- Sort entries alphabetically within each category
- For numeric values: display whole floats as integers (`2.0` → `2`), round others to 2 decimal places (no trailing zeros — `1.8` not `1.80`)
- When showing crafting materials, resolve internal DA_ asset names to their proper display names in Title Case and always include the quantity when available (e.g., `Copper Ingot x2`, `Pine Timber x1`)
- CraftingTime values (for buildings only) are stored in seconds — convert to human-readable: `9m`, `40m`, `1h 30m`, etc.

---

## Quality Check

Before you're done, review the output for:
- Are any entries missing or duplicated?
- Are visual-only variants collapsed, while gameplay-distinct tiers kept separate?
- Does each asset sit in the category that matches what it actually *is* in the game?
- Do crafting recipes show actual quantities (x2, x3, etc.) where they exist?
- Are Worth values showing up for items that have them (a good chunk — 150+ entries should have Worth)?
- Do display names look like proper game names (not internal asset IDs, GUIDs, or garbage strings)?
- **No NPC-only items**: check for Level 99 entries or villager/NPC clothing that slipped through
- **No internal engine data**: check that biomes doesn't have terrain sub-layers, world resources doesn't have vegetation cluster configs, jobs doesn't have internal sub-roles
- **Animals exist and have Health**: every animal should have at least Level and Health (extracted from StructProperty). Most hostile animals should also have Damage. All animals should have Drops.
- **Enemies have Health**: extracted from the Health StructProperty → MaxValue. Every enemy should show Level, Health, and Damage.
- **No Crafting Time on Items**: Items should NOT have a Crafting Time line
- **No Crafted using on Enemies or Animals**: combat entities don't have crafting recipes
- **Jobs use "Uses:" not "Crafted at:"**: jobs list the buildings/tools they use
- **World Resources use "Yields:" not "Crafted using"**: harvestable objects yield materials
- **Buildings have NO "Crafted at:" line**: buildings show their crafting recipe as `Crafted using` but not which station they're made at
- **Vendors have no occupation tags and no global settings**: just Required Level, Sells (short tags), Excludes, probability, rarity
- **Descriptions are correct**: spot-check that descriptions match their assets — a wrong description (like a shoe description on a food item) means the binary parsing mapped data incorrectly