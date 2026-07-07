# Project Miragul Maps

Project Miragul Maps is a curated EverQuest map pack for the Project Miragul server. The goal of this repository is to provide cleaner in-era map labels by removing NPCs that should not appear for the server era and by updating NPC labels to match the best available spawn coordinates from EQEmu / ProjectEQ data.

The work in this repository is focused on NPC map labels rather than changing the map geometry itself. The source maps are reviewed against EQEmu spawn data, then updated so that named NPCs, merchants, quest NPCs, and other useful static NPCs appear in the correct zones with corrected coordinates and clearer labels.

## What this project does

This project was built to solve three related problems:

1. **Remove out-of-era NPCs** from map files.
2. **Move NPC labels to their correct spawn coordinates** using EQEmu spawn data.
3. **Clean up NPC labels** so that the map files are more useful in-game for Project Miragul players.

The result is a map set that better matches the server's intended era and content availability.

## Using the maps in EverQuest

To use the curated Project Miragul maps in-game, copy the updated `.txt` map files into your EverQuest client `maps` folder.

### Install location

Your EverQuest install usually has a folder like:

```text
EverQuest\maps
```

For example:

```text
C:\Users\Games\EverQuest\maps
```

or, for a custom/private server client:

```text
C:\ProjectMiragul\maps
```

The exact location depends on where your EverQuest client is installed.

### Recommended install method

Before copying the Project Miragul map files, back up your existing maps:

```text
EverQuest\maps_backup_before_project_miragul
```

Then extract the [latest release](https://github.com/anotheregostar/projectmiragulmaps/releases/tag/PoP_LoY_v1.0) zip into:

```text
EverQuest\maps
```

After extraction, your folder layout should look like this:

```text
EverQuest\maps\Project Miragul Darkmode Maps
EverQuest\maps\Project Miragul Lightmode Maps
```

Inside each folder, you should see EverQuest map `.txt` files such as:

```text
poknowledge_1.txt
stonebrunt_1.txt
nexus_1.txt
```

The exact files may vary by release.

### Layered map files

EverQuest map files often use numbered layer files, such as:

```text
poknowledge.txt
poknowledge_1.txt
poknowledge_2.txt
poknowledge_3.txt
```

This project primarily updates NPC point-label layers, commonly files such as:

```text
zone_1.txt
```

When installing, keep the filenames exactly as provided. Do not rename them unless you know which layer you are intentionally replacing.

### Loading the maps in-game

After copying the files:

1. Launch EverQuest.
2. Log in to Project Miragul.
3. Enter a zone that has updated map files.
4. Open the map window.
5. If the map was already open, close and reopen it, or zone out and back in if the old labels are still cached.

The updated labels should appear as normal map points.

### If the labels do not appear

Check the following:

- The files are in the correct `EverQuest\maps` folder for the client you are actually launching.
- The filenames match the zone short name, such as `poknowledge_1.txt`.
- The map layer is enabled in the in-game map window.
- You are looking at the correct map tab/layer.
- You restarted the client or reloaded the zone after copying the files.
- The map file was not copied into a different EverQuest install by mistake.

### Reverting to old maps

To revert, close EverQuest and copy your backed-up files from:

```text
EverQuest\maps_backup_before_project_miragul
```

back into:

```text
EverQuest\maps
```

## Data sources used

The update process used the current EQEmu / ProjectEQ database as the reference source for NPC spawn information. The main database tables used were:

| Table | Purpose |
| --- | --- |
| `npc_types` | NPC name, level, merchant flag, class, race, body type, lastname / role, faction, loot table, and NPC identity. |
| `spawnentry` | Connects NPC IDs to spawn groups. |
| `spawn2` | Provides actual zone spawn points, including `x`, `y`, `z`, heading, respawn time, min expansion, and max expansion. |
| `spawngroup` | Identifies spawn group details and whether a spawn point can represent multiple possible NPCs. |
| `zone` | Provides zone short name, long name, zone ID, and zone expansion. |

The core relationship used for real spawn data was:

```text
npc_types -> spawnentry -> spawn2 -> zone
```

A separate scripted / inferred lookup was also considered for NPCs without a real `spawn2` row, using the common EQEmu convention of deriving a zone from the NPC ID range:

```sql
FLOOR(npc_types.id / 1000)
```

Those scripted or inferred NPCs were kept separate from the main spawn-coordinate workflow because they do not have reliable physical map coordinates.

## Filtering strategy

The goal was not to add every creature from EQEmu to the maps. Common monsters often have many spawn points and are not useful as permanent map labels. The update process therefore focused on NPCs that were likely to be static, named, merchant, quest, or otherwise useful.

The generated NPC listing excluded:

- NPC names beginning with `a_` or `A_`.
- NPC names beginning with `an_` or `An_`.
- NPCs with multiple spawn points within the same zone.
- NPCs whose spawn group included multiple possible NPC IDs.
- NPCs that appeared to be generic roaming or placeholder creatures rather than useful map labels.

The multiple-spawnpoint filter helped remove common monsters such as animals, guards with many copies, and other repeated mobs. The single-NPC-spawngroup filter helped avoid adding NPCs from random spawn tables where the same spawn point can produce several different NPCs.

## Merchant detection

NPCs were marked as merchants based on the `merchant_id` field in `npc_types`.

```sql
CASE
    WHEN merchant_id IS NOT NULL AND merchant_id <> 0 THEN 'Yes'
    ELSE 'No'
END AS is_merchant
```

This allowed the final review spreadsheet to distinguish merchants from other NPCs while still keeping both merchants and non-merchants in the candidate list.

## Scripted NPC detection

NPC names containing `#` were marked as scripted NPCs:

```sql
CASE
    WHEN npc_name LIKE '%#%' THEN 'Yes'
    ELSE 'No'
END AS scripted_npc
```

These NPCs were not automatically discarded, but the flag made them easier to review manually.

## Expansion handling

Both zone expansion and spawn expansion values were included in the review data.

The export included:

| Column | Meaning |
| --- | --- |
| `zone_expansion` | Expansion number assigned to the zone. |
| `zone_expansion_name` | Human-readable expansion name for the zone. |
| `min_expansion_number` | Minimum expansion where the spawn should be present. |
| `min_expansion` | Human-readable name for the minimum expansion. |
| `max_expansion_number` | Maximum expansion where the spawn should be present. |
| `max_expansion` | Human-readable name for the maximum expansion. |

When a zone was added in a specific expansion but the spawn `min_expansion` was `-1`, the export treated the spawn as beginning in the zone's own expansion. This avoided incorrectly marking Luclin, Planes of Power, or later-zone NPCs as available before their zones existed.

Example logic:

```sql
CASE
    WHEN spawn_min_expansion = -1
         AND zone_expansion IS NOT NULL
         AND zone_expansion >= 0
    THEN zone_expansion
    ELSE spawn_min_expansion
END AS min_expansion_number
```

## Expansion number reference

The SQL helper view converted expansion numbers to names. The mapping used by the export was:

| Number | Expansion |
| ---: | --- |
| 0 | Classic |
| 1 | Ruins of Kunark |
| 2 | Scars of Velious |
| 3 | Shadows of Luclin |
| 4 | Planes of Power |
| 5 | Legacy of Ykesha |
| 6 | Lost Dungeons of Norrath |
| 7 | Gates of Discord |
| 8 | Omens of War |
| 9 | Dragons of Norrath |
| 10 | Depths of Darkhollow |
| 11 | Prophecy of Ro |
| 12 | The Serpent's Spine |
| 13 | The Buried Sea |
| 14 | Secrets of Faydwer |
| 15 | Seeds of Destruction |
| 16 | Underfoot |
| 17 | House of Thule |
| 18 | Veil of Alaris |
| 19 | Rain of Fear |
| 20 | Call of the Forsaken |
| 21 | The Darkened Sea |
| 22 | The Broken Mirror |
| 23 | Empires of Kunark |
| 24 | Ring of Scale |
| 25 | The Burning Lands |
| 26 | Torment of Velious |
| 27 | Claws of Veeshan |
| 28 | Terror of Luclin |
| 29 | Night of Shadows |
| 30 | Laurion's Song |
| 31 | The Outer Brood |

## Review spreadsheet process

After the SQL views were created in HeidiSQL, the NPC report was exported to a spreadsheet. The working spreadsheet combined:

- Existing map labels.
- EQEmu NPC spawn data.
- Match status.
- Match distance.
- Map coordinates.
- EQEmu spawn coordinates.
- Expansion availability.
- Merchant status.
- NPC role / lastname.

The spreadsheet was then used to decide whether each NPC should be:

| Status | Meaning |
| --- | --- |
| Existing map label retained | The map already had a useful in-era label. |
| Coordinate match | The existing map label was close enough to an EQEmu spawn coordinate to be treated as the same NPC. |
| Possible match | The name matched or nearly matched, but manual review was needed. |
| Added from database | The NPC was missing from the map but should be added. |
| Removed as out-of-era | The NPC did not belong in the Project Miragul era. |
| Removed by cleanup term | The map label contained a configured cleanup term such as `Task_Master`, `Mercenary`, `Quests`, or `Tutorial`, even if it was also an `NPC + Map` match. |

## Coordinate matching

The update process compared existing map label coordinates against EQEmu spawn coordinates. When an NPC was already a possible match, coordinate distance was used to determine whether it should be treated as a coordinate match.

The important rule was:

- First determine whether the NPC is a possible match.
- Then check whether the match distance is within the accepted range.
- Only possible matches should be promoted to coordinate matches.

This avoided incorrectly matching unrelated NPCs just because they happened to be physically near each other on the map.

## Label cleanup rules

Several utility/task/mercenary labels are now removed when they are not useful for the Project Miragul map set. This cleanup applies to **all matched rows**, not only `Map only` rows. That means a label such as `Sheley_Courilan_(Task_Master)` is deleted even though it is a valid `NPC + Map` match, because the map label contains a configured cleanup term.

- `Task_Master`
- `Adventure_Point_Merchant`
- `Wayfarer_Camp_Port`
- `Augmentation_Distillers`
- `Adventure_Merchant`
- `Group_Adventures`
- `Trade_Skill_Quests`
- `Task_Master,Roam`
- `roam,_Temp-Halloween`
- `Melee_Augs`
- `Mercenary`
- `Tradeskill_Quests`
- `Tasks`
- `Casino`
- `Quests`
- `Crown_Currency_Merchant`
- `Items_50+`
- `Items_5-49`
- `Hot_Zone_Quests`
- `Armor_Quests`
- `Wedding_Coordinator`
- `Tradeskill_Kits`
- `Wedding_Corrdinator`
- `Living_Legacy`
- `Fellowship_Vendor`
- `Hot_Zones`
- `Special_Mercenary_Liaison`
- `Legendary_Liaison`
- `Spirit_Shrouds`
- `Missions`
- `Classic_Missions`
- `Fellowship_Registrar`
- `Guild_Banners`
- `Mercenary_Liason`
- `Mercenary_Liason,Roam`
- ``Hero`s_Forge``
- `Mercenary_Roster_Quest`
- `Tutorials`
- `Tutorial`
- `Overseer_Tetradrachms`
- `Group_Armor`
- `The_Tamrel_Trials`
- `Mine_Quest`
- `Tradeskill_Books`

NPCs with names containing `_Fabled_` were also excluded from the final generated map data because they are generally not appropriate for a normal in-era map label set.

## Map update process

The overall process was:

1. Download or import the current EQEmu / ProjectEQ database into MariaDB.
2. Create a separate reference database in HeidiSQL so the live server database was not modified.
3. Build SQL views to extract likely useful NPC spawn records.
4. Export the NPC spawn report to CSV / Excel.
5. Combine the exported NPC data with the existing map label data.
6. Identify existing labels, coordinate matches, possible matches, and map-only labels.
7. Remove out-of-era NPCs based on expansion availability.
8. Remove configured cleanup labels from both `Map only` and `NPC + Map` rows.
9. Update matched NPC labels to the correct EQEmu spawn coordinates.
10. Add missing useful NPCs from the EQEmu data.
11. Export the corrected `.txt` map files.
12. Review the maps in-game and in the map editor.


## Current script workflow

The repository uses two main Python scripts and drag-and-drop batch files.

### 1. Extract map points and build the review workbook

Drag the source map folder onto:

```text
Drag Folder Here - Extract EQ Map Points.bat
```

This runs:

```text
extract_eq_map_points.py
```

The extraction script reads all `P` point lines from the map `.txt` files, converts map coordinates to database-style coordinates for comparison, loads `EQMapEditor NPC Label List.csv`, and creates:

```text
<DraggedFolderName> Combined Map Data.xlsx
```

The extract step now keeps map-only rows in the workbook so the apply step can audit and delete unwanted labels from the real map files.

### 2. Review the workbook

Open the generated `Combined Map Data.xlsx` file and review:

- `source_status`
- `npc_matched`
- `npc_name`
- `npc_role`
- `npc_map_label`
- `map_label`
- `match_distance`
- expansion columns
- map and NPC coordinates

This workbook is the audit point before the map files are changed.

### 3. Apply the reviewed data to the maps

Drag the same source map folder onto:

```text
Drag Folder Here - Apply Combined Map Data.bat
```

This runs:

```text
apply_combined_map_data.py
```

The apply script finds the workbook named:

```text
<DraggedFolderName> Combined Map Data.xlsx
```

It then backs up the original map files, applies replacements/deletions, and writes an apply summary CSV.

## Current apply-script behaviour

The apply script now performs cleanup before normal NPC replacement. This is important because some unwanted labels can still be valid NPC matches. For example:

```text
Sheley_Courilan_(Task_Master)
```

matches the real NPC `Sheley_Courilan`, but it is still removed because the map label contains `Task_Master`.

The current apply order is:

1. If `map_label` contains a configured cleanup term, delete the matching map point line.
2. Otherwise, if the NPC is out of era, delete the matching map point line.
3. Otherwise, if the row is a valid matched NPC, replace the map point with the corrected NPC coordinate and label.
4. Write an apply summary showing what happened to each actionable row.

The apply script also protects against incorrect coordinate-only replacements:

- A map point line can only be replaced once.
- Coordinate matches must still be a reasonable name/fuzzy-name match.
- Nearby NPCs with different names are skipped for manual review instead of overwriting each other.
- Duplicate rows for the same NPC/map point keep the row with the smallest `match_distance`.

## Suggested regeneration workflow

To regenerate the NPC source data:

1. Open HeidiSQL.
2. Connect to the MariaDB server that contains the EQEmu / ProjectEQ database.
3. Select the reference database.
4. Run the SQL helper views.
5. Run the final export query.
6. Export the result grid as CSV.
7. Use the exported CSV as the NPC reference data for the map update scripts.

The final report columns used for map review were:

| Column | Purpose |
| --- | --- |
| `npc_name` | NPC name from EQEmu. |
| `scripted_npc` | `Yes` if the NPC name contains `#`. |
| `npc_role` | NPC `lastname`, often used for role text such as merchant type or title. |
| `zone_short_name` | EQEmu short zone name. |
| `zone_long_name` | Readable zone name. |
| `zone_expansion` | Numeric expansion value for the zone. |
| `zone_expansion_name` | Expansion name for the zone. |
| `spawn_x` | EQEmu spawn X coordinate. |
| `spawn_y` | EQEmu spawn Y coordinate. |
| `spawn_z` | EQEmu spawn Z coordinate. |
| `is_merchant` | `Yes` if `merchant_id` is populated. |
| `min_expansion_number` | Numeric minimum expansion value after zone fallback correction. |
| `min_expansion` | Readable minimum expansion name. |
| `max_expansion_number` | Numeric maximum expansion value. |
| `max_expansion` | Readable maximum expansion name. |

## Final export query example

The final review query used the curated one-spawn-per-zone view and filtered out common generic NPC naming patterns. It also removed spawn groups that could spawn multiple possible NPCs.

```sql
WITH single_npc_spawngroups AS (
    SELECT
        spawngroupID
    FROM spawnentry
    GROUP BY spawngroupID
    HAVING COUNT(DISTINCT npcID) = 1
)
SELECT
    npc_name,

    CASE
        WHEN npc_name LIKE '%#%' THEN 'Yes'
        ELSE 'No'
    END AS scripted_npc,

    lastname AS npc_role,

    zone_short_name,
    zone_long_name,

    zone_expansion,
    zone_expansion_name,

    ROUND(x, 2) AS spawn_x,
    ROUND(y, 2) AS spawn_y,
    ROUND(z, 2) AS spawn_z,

    CASE
        WHEN is_merchant = 1 THEN 'Yes'
        ELSE 'No'
    END AS is_merchant,

    CASE
        WHEN spawn_min_expansion = -1
             AND zone_expansion IS NOT NULL
             AND zone_expansion >= 0
        THEN zone_expansion
        ELSE spawn_min_expansion
    END AS min_expansion_number,

    CASE
        WHEN spawn_min_expansion = -1
             AND zone_expansion IS NOT NULL
             AND zone_expansion >= 0
        THEN zone_expansion_name
        ELSE spawn_min_expansion_name
    END AS min_expansion,

    spawn_max_expansion AS max_expansion_number,
    spawn_max_expansion_name AS max_expansion

FROM vw_npc_spawn_listing_one_spawn_per_zone v

JOIN single_npc_spawngroups sng
    ON sng.spawngroupID = v.spawngroupID

WHERE
    npc_name NOT LIKE 'a\_%' ESCAPE '\\'
    AND npc_name NOT LIKE 'A\_%' ESCAPE '\\'
    AND npc_name NOT LIKE 'an\_%' ESCAPE '\\'
    AND npc_name NOT LIKE 'An\_%' ESCAPE '\\'

GROUP BY
    npc_name,

    CASE
        WHEN npc_name LIKE '%#%' THEN 'Yes'
        ELSE 'No'
    END,

    lastname,

    zone_short_name,
    zone_long_name,

    zone_expansion,
    zone_expansion_name,

    ROUND(x, 2),
    ROUND(y, 2),
    ROUND(z, 2),

    CASE
        WHEN is_merchant = 1 THEN 'Yes'
        ELSE 'No'
    END,

    CASE
        WHEN spawn_min_expansion = -1
             AND zone_expansion IS NOT NULL
             AND zone_expansion >= 0
        THEN zone_expansion
        ELSE spawn_min_expansion
    END,

    CASE
        WHEN spawn_min_expansion = -1
             AND zone_expansion IS NOT NULL
             AND zone_expansion >= 0
        THEN zone_expansion_name
        ELSE spawn_min_expansion_name
    END,

    spawn_max_expansion,
    spawn_max_expansion_name

ORDER BY
    zone_expansion,
    zone_short_name,
    npc_name;
```

## Project status

This repository is a curated map-data project for Project Miragul. The map files may continue to evolve as more zones are reviewed, out-of-era labels are found, and Project Miragul's era/content rules are refined.
