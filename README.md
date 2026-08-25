# Noorland Project Info

Design and status reference for the NoorlandMC server and the NoorlandStudios plugin suite.

> This document has two jobs: describe **what we're building** (design intent) and record
> **what actually exists right now** (implementation state). Those drifted badly apart, so
> every gameplay section below is now marked.

| Marker | Meaning |
|:------:|---------|
| ✅ | Implemented and in the codebase |
| 🟡 | Partially implemented — some of it is live, some is still design |
| 📋 | Designed only — no code backing it yet |

Anything marked 📋 is a **decision still owned by design**, not a bug. Anything where the
design text and the code disagree has been rewritten to match the code, with the old intent
kept as a note where it still matters. Open conflicts are collected in
[Open Questions & Design Drift](#open-questions--design-drift).

---

## Contents

- [Platform Baseline](#platform-baseline)
- [Architecture](#architecture)
- **Core Stack**
  - [Ranks & Perms](#ranks--perms)
  - [Jobs & Promotions](#jobs--promotions)
  - [Quests](#quests)
  - [NPCs](#npcs)
  - [Towns, Camps & Nations](#towns-camps--nations)
  - [Economy, Market & Auction House](#economy-market--auction-house)
  - [Items, Sets & Blueprints](#items-sets--blueprints)
  - [Geodes, Drops & Salvage](#geodes-drops--salvage)
  - [Fishing](#fishing)
  - [Chat & Social](#chat--social)
  - [Events](#events)
  - [Resource Pack & Bedrock](#resource-pack--bedrock)
  - [Moderation & Safety](#moderation--safety)
- [Donator Store](#donator-store)
- [Structure](#structure)
- [Monetization (Through content)](#monetization-through-content)
- [Open Questions & Design Drift](#open-questions--design-drift)

---

# Platform Baseline

| Thing | Value |
|-------|-------|
| Server | Paper `1.21.11` |
| Java | 21 (toolchain pinned across the suite) |
| Folia | Every Noor plugin ships `folia-supported: true` |
| Group | `com.noorlandmc` |
| Core version | NoorCore `1.1.0` |
| Consumed as | Shadow/shaded artifact — jitpack `com.github.NoorlandStudios:NoorCore`, or `publishToMavenLocal` for suite development |

**Scheduling rule.** Nothing in the suite uses `Bukkit.getScheduler()` or `BukkitRunnable` —
they throw on Folia. NoorCore shades a single relocated copy of
[FoliaLib](https://github.com/TechnicallyCoded/FoliaLib) (`0.5.2`, relocated to
`com.noorlandmc.NoorCore.lib.folialib`) and exposes it via `NoorCore.getFoliaLib()`.
Downstream plugins reuse that one instance rather than bundling their own.

```java
// Entity-region thread (movement, teleports, kicks, inventory edits)
NoorCore.getFoliaLib().getScheduler().runAtEntity(player, () -> player.kick(reason));
// Location/region thread (block edits, world changes)
NoorCore.getFoliaLib().getScheduler().runAtLocation(loc, () -> block.setType(type));
// Off-thread work
NoorCore.getFoliaLib().getScheduler().runAsync(() -> heavyComputation());
```

### Third-party dependencies in play

| Plugin | Used by | Hard/soft |
|--------|---------|:---------:|
| Vault | NoorEconomy, NoorQuests | hard |
| LuckPerms | NoorCrates, NoorChatExtras, NoorQuests, NoorRanks, NoorAdmin | soft |
| PlaceholderAPI | NoorTowns, NoorCrates, NoorChatExtras, NoorEvents | soft |
| EssentialsX (+ Economy) | NoorEconomy (balance provider), NoorFriends, NoorTowns, NoorAdmin | soft |
| Geyser + Floodgate | NoorPack, NoorItems, NoorGeodes, NoorFish (`loadbefore`) | soft |
| WorldEdit / FastAsyncWorldEdit | NoorTowns (schematics), NoorNPCs | soft |
| WorldGuard | NoorNPCs | soft |
| dynmap / squaremap / BlueMap | NoorTowns, NoorFamilies | soft |
| LiteBans | NoorAdmin, NoorChatExtras | soft |
| SilkSpawners | NoorCrates, NoorPickup, NoorJobs (Miner perk) | soft |
| QuickShop-Hikari | NoorJobs | soft |

---

# Architecture

Everything routes through **NoorCore**. Satellite plugins never depend on each other's
internals — they talk through service bridges registered in NoorCore's registry, so a
plugin can be absent without breaking its consumers.

```
                          ┌───────────────────────┐
                          │       NoorCore        │
                          │  ServiceRegistry/API  │
                          │  EventBus             │
                          │  Feature flags        │
                          │  AntiCheat trackers   │
                          │  Menu framework       │
                          │  Shaded FoliaLib      │
                          └───────────┬───────────┘
                                      │ service bridges
   ┌──────────────┬───────────────┬───┴────────┬───────────────┬──────────────┐
   │              │               │            │               │              │
NoorItems     NoorTowns      NoorEconomy   NoorJobs       NoorNPCs      NoorPack
NoorGeodes    NoorQuests     NoorCrates    NoorFish       NoorEvents    NoorChatExtras
NoorRanks     NoorFriends    NoorFamilies  NoorAdmin      NoorPickup    NoorAdvancements
```

### NoorCore subsystems ✅

| Package | What it holds |
|---------|---------------|
| `api` | `API`, `Service`, `ServiceRegistry` — bridge lookup/registration |
| `bridge` | 19 cross-plugin interfaces (below), plus 2 in `economy` |
| `dto` | Bukkit-free snapshots passed across bridges |
| `economy` | Transaction/audit contracts — `EconomyTransactionServiceBridge`, `EconomyAuditUiBridge`, receipts, audit records, integrity status |
| `events` | `EventBus`, `NoorEvent` |
| `featureflags` | `FeatureFlag`, `FlagRegistry`, `FlagEntry`, `FeatureFlagChangedEvent` |
| `anticheat` | Combat, Interaction, Inventory, Movement and PingAnomaly trackers; `FlagSystem`, `FlagType`, `StaffAlertLevel` |
| `menu` | Shared GUI framework — `Menu`, `GUIMenu`, `PagedMenu`, `CenteredMenu`, `MenuButton`, live menu editor, icon resolver, RTP menu |

All cross-thread state in NoorCore (registry, event bus, anticheat trackers, menu viewers)
is backed by concurrent collections so it is safe under Folia's region model.

### Service bridges ✅

| Bridge | Implemented by | Purpose |
|--------|----------------|---------|
| `TownServiceBridge` | NoorTowns | Town/camp lookups, personal camp storage read/write |
| `NpcTownPersistenceBridge` | NoorTowns | Durable, idempotent, retry-safe NPC↔town operations |
| `ItemServiceBridge` | NoorItems | Build/resolve custom items, kits, sets, blueprint sets, catalog |
| `PackServiceBridge` | NoorPack | Generated models, item definitions, translations |
| `ChatGlyphServiceBridge` | NoorPack | Chat emoji/tag glyphs baked into the pack font |
| `ChatTagServiceBridge` | NoorChatExtras | Cosmetic chat tags — unlocked, selected, catalog |
| `GeodeServiceBridge` | NoorGeodes | Geodes and their exact reward items |
| `QuestServiceBridge` | NoorQuests | Start quests, report custom progress, resolve `quest:` flags |
| `FishServiceBridge` | NoorFish | Recognise custom fish, compute catch rewards, assemble rods |
| `FishingContestServiceBridge` | NoorEvents | Feed catches into contests, inspect contest state |
| `GuideNpcServiceBridge` | NoorNPCs | Register external guide dialogue actions |
| `EntityServiceBridge` | NoorEntities | Query/validate custom entities |
| `AdvancementServiceBridge` | NoorAdvancements | Query custom advancements |
| `RankServiceBridge` | NoorRanks | Open the rank menu |
| `FriendServiceBridge` | NoorFriends | Open the friend menu |
| `FamilyServiceBridge` | NoorFamilies | Child-account chat policy, audit logging, teleport policy |
| `MarriageServiceBridge` | *NoorMarriage — no repo yet* 📋 | Cached marriage/spouse/shared-home/Bond snapshots |
| `PickupServiceBridge` | NoorPickup | Offer direct, player-attributed drops |
| `UtilServiceBridge` | NoorUtils | Shared helpers |
| `EconomyTransactionServiceBridge` | NoorEconomy | Audited money mutation |
| `EconomyAuditUiBridge` | NoorEconomy | Audit history/health surfaces for other UIs |

> ⚠️ `MarriageServiceBridge` (23 methods) and its Bond/shared-home DTOs are **already in
> NoorCore and documented in its README**, but there is no `NoorMarriage` repository in the
> org. Either the plugin is unstarted, lives somewhere not visible, or the bridge should be
> pulled. See [Open Questions](#open-questions--design-drift).

---

# Ranks & Perms

🟡 — **NoorRanks** exists and drives `/ranks` and `/rankup`, but the rank ladder below is
config/design, and prestige is not implemented.

> All ranks get access to 1 Tier 1, Tier 2, or Tier 3 Crate upon leveling up depending on where in the Ranks path someone is.

| Tier |    Rank    |     Cost     |                         Commands                          | Other perks                                                                                                                                                                                                                                                                                                                                         |
|:----:|:----------:|:------------:|:---------------------------------------------------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  1   |  Newcomer  |      $0      | `/sethome` `/home` `/msg` `/reply` `/tpa` `/rtp` `/trash` | RTP CD - 3 Min                                                                                                                                                                                                                                                                                                                                      |
|  1   |  Settler   |   $10,000    |                    `/craft` `/furnace`                    | +4 Free [Chunk Claims](#towns-camps--nations) (4)                                                                                                                                                                                                                                                                                                   |
|  1   |  Laborer   |   $15,000    |                 `/condense` `/blastfurn`                  | +1 [Promotion-eligible Job](#jobs--promotions) (1)                                                                                                                                                                                                                                                                                                  |
|  1   | Apprentice |   $25,000    |              `/anvil` `/stonecutter` `/back`              | +2 Homes (2)                                                                                                                                                                                                                                                                                                                                        |
|  1   |  Citizen   |   $50,000    |                          `/hat`                           | +1 Home (3) <br> +1 [Town](#towns-camps--nations) (1) <br> +1 [Job](#jobs--promotions) (2)                                                                                                                                                                                                                                                          |
|  1   |  Merchant  |   $75,000    |                 `/pw` `/chestshop` `/ah`                  | +2 Auction House Listings (2) — `nooreconomy.auctionhouse.listings.2`                                                                                                                                                                                                                                                                               |
|  1   |  Artisan   |   $150,000   |           `/cartogrophy` `/loom` `/grindstone`            | Ability To Craft Concrete                                                                                                                                                                                                                                                                                                                           |
|  1   | Prospector |   $200,000   |                          `/hdb`                           | +1 [Daily Quest](#quests) per difficulty (2) — `noorquests.extradaily.1` <br> RTP CD - 1 Min                                                                                                                                                                                                                                                        |
|  1   |  Explorer  |   $350,000   |                `/ec` `/ptime` `/pweather`                 | +1 [Job](#jobs--promotions) (3) <br> +1 Weekly Small [Crate](#crates) (1) <br> 1 Large [Crate](#crates) <br> Unique [Cosmetic](#cosmetics) <br> ✅ grants `bonfire-key` via NoorCrates rank reward                                                                                                                                                   |
|  1   |   Total    |   $875,000   |                                                           |                                                                                                                                                                                                                                                                                                                                                     |
|      |            |              |                                                           |                                                                                                                                                                                                                                                                                                                                                     |
|  2   |  Captain   |   $500,000   |                  `/heal` `/near` `/show`                  | +1 Auction House Listing (3) <br> Heal CD - 2 Min                                                                                                                                                                                                                                                                                                   |
|  2   | Magistrate |  $1,000,000  |                 `/speed` `/nick` `/pv 1`                  | +1 [Job](#jobs--promotions) (4) <br> Unique [Cosmetic](#cosmetics)                                                                                                                                                                                                                                                                                  |
|  2   |   Baron    |  $1,500,000  |                         `/pw 1:2`                         | +2 Homes (5) <br> +1 Daily Reward (2) <br> +1 [Nation](#nations) (1)                                                                                                                                                                                                                                                                                |
|  2   |   Count    |  $2,500,000  |             `/repair` `/rename` `/jumpboost`              | Repair CD - 2 Hour                                                                                                                                                                                                                                                                                                                                  |
|  2   |  Viscount  |  $4,000,000  |                          `/pv 2`                          | +1 [Daily Quest](#quests) per difficulty (3) <br> +1 [Job](#jobs--promotions) (5)                                                                                                                                                                                                                                                                   |
|  2   |    Earl    |  $6,500,000  |                                                           | +2 Homes (7) <br> Colored `/nick` <br> +1 Group Vault (1)                                                                                                                                                                                                                                                                                           |
|  2   |  Marquis   | $10,000,000  |                  `/lore` `/feed` `/pv 3`                  | Feed CD - 2 Min <br> Keep Exp on death                                                                                                                                                                                                                                                                                                              |
|  2   |    Duke    | $11,500,000  |                `/glow` `/strength` `/back`                | +1 [Town](#towns-camps--nations) (2) <br> Back CD - 5 Min <br> +1 [Job](#jobs--promotions) (6)                                                                                                                                                                                                                                                      |
|  2   |  Governor  | $12,500,000  |                      `/tfly` `/head`                      | +3 Homes (10) <br> Head CD - 1 Day <br> All Level 10 [Job](#jobs--promotions) Perks <br> +1 Weekly Medium [Crate](#crates) (1) <br> 3 Large [Crates](#crates) <br> Unique [Cosmetic](#cosmetics)                                                                                                                                                    |
|  2   |   Total    | $50,000,000  |                                                           |                                                                                                                                                                                                                                                                                                                                                     |
|      |            |              |                                                           |                                                                                                                                                                                                                                                                                                                                                     |
|  3   |   Regent   | $20,000,000  |                          `/pv 4`                          | +3 Emotes (3) <br> Access to more [Town Upgrades](#town-upgrades)                                                                                                                                                                                                                                                                                   |
|  3   |  Viceroy   | $25,000,000  |                          `/top`                           | No Back CD <br> RTP CD - 3 Sec <br> +1 [Job](#jobs--promotions) (7)                                                                                                                                                                                                                                                                                 |
|  3   |   Archon   | $30,000,000  |                      `/fly` `/haste`                      | +2 Homes (12) <br> Fly Time - 1 Hour                                                                                                                                                                                                                                                                                                                |
|  3   | Chancellor | $35,000,000  |                      `/pw 3` `/pv 5`                      | 3 Biome RTP/Day                                                                                                                                                                                                                                                                                                                                     |
|  3   |  Grandee   | $40,000,000  |                       `/repair all`                       | +3 Homes (15) <br> No Feed CD <br> No Heal CD <br> +1 [Job](#jobs--promotions) (8)                                                                                                                                                                                                                                                                  |
|  3   |  Highlord  | $45,000,000  |              `/stack` `/resistance` `/pv 6`               | Basic Cosmetics <br> Gradient `/nick`                                                                                                                                                                                                                                                                                                               |
|  3   |  Overlord  | $55,000,000  |                         `/trail`                          | +5 Homes (20) <br> No Fall Damage                                                                                                                                                                                                                                                                                                                   |
|  3   | Sovereign  | $70,000,000  |            `/fireres` `/nfly` `/pv 7` `/pw 4`             | Keep Inventory <br> +1 [Job](#jobs--promotions) (9)                                                                                                                                                                                                                                                                                                 |
|  3   | Imperator  | $80,000,000  |                  `/pw 5` `/regeneration`                  | +5 Homes (25) <br> No Fly Timer <br> No Repair Cooldown <br> [Access To All Trails](#cosmetics) <br> All Level 25 [Job](#jobs--promotions) Perks <br> +1 [Town](#towns-camps--nations) (3) <br> Unique [Cosmetic](#cosmetics) <br> +1 Weekly Large [Crate](#crates) <br> 10x Large [Crates](#crates) <br> Pay for others rankups <br> [Prestige](#prestige) |
|  3   |   Total    | $400,000,000 |                                                           |                                                                                                                                                                                                                                                                                                                                                     |

### Permission nodes these map onto ✅

The rank ladder is expressed through real permission nodes, not bespoke rank logic:

| Perk | Node |
|------|------|
| Auction House listing cap | `nooreconomy.auctionhouse.listings.<amount>` (or `.*` for unlimited) |
| Extra daily quests | `noorquests.extradaily.<amount>` — **does not stack**, highest enabled value wins |
| Personal camp size | `noortowns.camp.size.<3-10>` |
| Town flight | `noortowns.tfly` |
| Wilderness PvP opt-out | `noortowns.passive` |
| Weekly crate deliveries | `noorcrates.weekly.<delivery>` |
| Per-crate access | `noorcrates.crate.<crateId>` |

> `nq.prospector` is kept as a **legacy alias** granting +1 daily quest per difficulty, which
> is where the Prospector rank's perk originally came from.

### Prestige

📋 — Designed, not implemented. NoorRanks currently exposes only `/ranks`, `/rankup` and a
reload command; there is no prestige command, no prestige state, and no forfeit logic.

Intent: once reaching Imperator, you can prestige, giving special perks for the forfeit of
your rank, setting you back to Newcomer.

---

# Jobs & Promotions

✅ — **NoorJobs**. This section previously described a "Jobs & Mastery" model with 9 base
jobs that each evolved into a second job at level 60, plus a separate `NoorMastery` plugin.
**That model no longer exists.** It was replaced by 14 independent jobs plus a promotion
track system inside NoorJobs. The old evolved names survive as promotion *track* names.

### The 14 jobs

Every job is always available to every player. There are no advancement slots, no `base-job`
semantics, and no `jobs.advancements.*` permissions — that system was removed.

| ID | Display | Main activity | Reward actions | Configured actions |
|----|---------|---------------|----------------|------------------:|
| `miner` | Mining | Ores, stone, valuable blocks, terracotta, Nether stone | `break` | 46 |
| `lumberjack` | Lumberjack | Logs, woods, stems, stripped variants, leaves, roots, vines | `break` | 40 |
| `farmer` | Farming | Mature crops and farm blocks (+ `till`, `plant`) | `break` | 13 |
| `hunter` | Hunting | Hostile and huntable mob kills | `kill` | 21 |
| `rancher` | Ranching | Breeding, shearing, milking, passive mob kills | `breed` `shear` `milk` `kill` | 42 |
| `builder` | Building | Placing blocks (timed) | `place` | 361 |
| `smelter` | Smelting | Furnace extraction | `smelt` | 14 |
| `brewer` | Brewing | Brewing stand ingredients | `brew` | 19 |
| `fisher` | Fishing | Caught fish and treasure | `fish` | 14 |
| `digger` | Digger | Dirt, sand, gravel, clay, mud, sandstone | `break` | 14 |
| `crafter` | Crafter | Utility crafting | `craft` | 17 |
| `blacksmith` | Blacksmith | Weapon, armor and tool crafting | `craft` | 22 |
| `wizard` | Wizard | Magical crafting and enchanting | `craft` `enchant` | 16 |
| `chef` | Chef | Food crafting and cooking | `craft` `smelt` | 19 |

> **What changed vs. the old design:** Blacksmith, Chef, Wizard, Crafter and Forester were
> previously *level-60 evolutions* of Smelter, Fisherman, Arcane, Builder and Lumberjack.
> Blacksmith, Chef, Wizard and Crafter are now standalone jobs. "Arcane" is now `brewer`.
> "Fisherman" is now `fisher`. Forester, Helldiver, Hellhunter and Zookeeper became
> promotion tracks instead of jobs. `digger` is entirely new and was never in this document.

### Promotion system ✅

Promotions are player-selected paths layered on top of normal job levels.

- One promotion point per **10 job levels**, capped at **10 points at level 100**.
- Points are derived from current level, not stored as authoritative data.
- Each promotion-enabled job has **3 tracks**, each with **ranks I–IV**.
- Rank gates: **I → level 25, II → 50, III → 75, IV → 100**.
- Each active purchased rank adds **+10% money and +10% XP** to that track's matching actions.
- Rank IV gives the fourth +10% *and* unlocks the track's dynamic permission hook.
- Ranks I–III are multiplier-only.
- 3 tracks × 4 ranks = **12 possible ranks but only 10 points**, so a job can never be fully
  completed. This is deliberate.
- **Builder promotions are disabled** — timed Builder payouts no longer vary by material.

Stored ranks are retained if an admin lowers a job level, but inactive ranks grant nothing
until the gate is met again. Definitions live in `plugins/NoorJobs/promotions.yml`.

| Job | Tracks |
|-----|--------|
| Miner | Quarrier · Prospector · Helldiver |
| Lumberjack | Woodcutter · Forester · Helljack |
| Farmer | Agricultist · Botanist · Horticultist |
| Hunter | Paladin · Exterminator · Hell Hunter |
| Rancher | Zoologist · Shearer · Slaughterer |
| Smelter | Refiner · Kilnmaster · Fuelmaster |
| Brewer | Alchemist · Toxicologist · Distiller |
| Fisher | Angler · Salvager · Treasure Hunter |
| Digger | Excavator · Sifter · Sandman |
| Crafter | Artisan · Mechanist · Provisioner |
| Blacksmith | Weaponsmith · Armorer · Toolsmith |
| Wizard | Enchanter · Runecrafter · Arcanist |
| Chef | Baker · Butcher · Gourmet |

> ⚠️ **Naming collision:** `Prospector` is both a Tier-1 *rank* and a Miner *promotion track*.
> Likewise `Forester` and `Artisan`. Worth renaming one side before launch.

> 🟡 Rank IV perks are currently **permission/display hooks only**. NoorJobs does not yet
> implement the custom item, recipe, combat, drop or metadata behavior the perk names imply.

### Level perks per job

These are the milestones actually declared in the bundled job YAMLs. Only nodes listed under
`perk-permissions` are attached automatically by NoorJobs; the rest are descriptions of
behavior owned by other plugins.

#### Miner
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | `/nightvision` | `cc.nightVision` |
| 25 | Silk Touch spawners | `silkspawners.break.*`, `silkspawners.place.*` |
| 35 | `/haste` | `cc.haste` |
| 50 | Higher fragment & catalyst chance | `noorjobs.miner.geodes.*` |
| 60 | Craft Golden Carrying Bags | |
| 75 | `/fireres` | `noormisc.effect.fireres` |
| 85 | `/repair` (30 min cooldown) | |
| 100 | Fly below y-level 60 | |

#### Lumberjack
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | 10% extra saplings from leaves and logs | `noorjobs.lumberjack.saplingrefund` |
| 25 | Treefeller (sneak, capped 2×2 vertical column, same log family) | `noorjobs.lumberjack.treefeller` |
| 35 | `/haste` (Haste II) | `noormisc.effect.haste.2` |
| 50 | Climb logs like ladders | `noormisc.logclimb` |
| 60 | Access to new condensed blocks | |
| 75 | `/top` | |
| 85 | Insta-Smelt logs on break | |
| 100 | No fall damage while holding an axe | |

#### Farmer
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | Auto-replant crops (consumes seeds from inventory) | `noorjobs.farmer.autoreplant.seeds` |
| 25 | 10% chance to double crop yields | `noorjobs.farmer.doublecrops` |
| 35 | Craft stronger fertilizer | |
| 50 | Auto-replant without consuming seeds | `noorjobs.farmer.autoreplant.seedless` |
| 60 | Crop Bags (infinite crop storage) | |
| 75 | Extra farm crafting recipes | |
| 85 | `/speed` | |
| 100 | No longer go hungry | |

Farmland tilling pays via `till.farmland`; manual planting pays per planted material under
`plant`. Plant rewards are guarded per location until a mature crop is harvested, so
place/break seed farming does not pay repeatedly. Auto-replant does **not** trigger a plant payout.

#### Hunter
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | 5% chance to double mob drops | `noorjobs.hunter.doubledrops` |
| 25 | Extra mob drops and crafting recipes | |
| 35 | `/cleareffect` (2 min cooldown) | `noorjobs.hunter.cleareffect` |
| 50 | Hostile mobs ignore you | `noorjobs.hunter.deagro` |
| 60 | `/strength` | |
| 75 | `/resistance` | |
| 85 | `/capture` for hostile mobs | |
| 100 | Craft mob spawners | |

#### Rancher
| Level | Perk |
|:-----:|------|
| 10 | Extra recipes from passive mob drops |
| 25 | Craft leads and name tags |
| 35 | Craft animal feed |
| 50 | `/capture` for passive mobs |
| 60 | Disguise as passive mobs |
| 75 | Craft passive mob spawners |
| 85 | New passive mob drops and recipes |
| 100 | Custom pets and instant breeding |

#### Builder
| Level | Perk |
|:-----:|------|
| 10 | Create elevators |
| 25 | `/levitate` |
| 35 | `/slowfall` |
| 50 | `/top` |
| 60 | Create teleport pads |
| 75 | `/tfly` |
| 85 | Craft all furniture |
| 100 | `/nfly` |

> This table previously listed its rank column as "Arcane / Wizard" — a copy-paste from the
> Brewer table. Fixed.

#### Smelter
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | Craft coal blocks from charcoal, and Mini-Coal | `nooritems.recipe.mini_coal` |
| 25 | 10% chance of double smelts | `noorjobs.double_smelt.10` |
| 50 | `/top` | `essentials.top` |
| 60 | Combine Iron + Coal into Steel | |
| 75 | Smelt Obsidian and Slimeballs | |
| 85 | Create and use 1 chunk loader | |
| 100 | Smelt and create Uranium | |

> Design intent had Level 50 as "Mega-Coal (smelts 64 items)" and Level 100 as
> "Titanium **and** Uranium". The shipped YAML has Mini-Coal at 10 and Uranium only. Also
> note levels 35 and 50 differ from the original ladder — no level-35 perk is configured.

#### Brewer *(formerly "Arcane")*
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | Brew Potions of Saturation | `noorjobs.brewer.saturation` |
| 25 | Brew Potions of Luck | `noorjobs.brewer.luck` |
| 35 | Brew level I & II Resistance potions | |
| 50 | Brew level I & II Haste potions | `noorjobs.brewer.luck` ⚠️ |
| 60 | Brew with Redstone Blocks for 15:00 duration | |
| 75 | Brew level III potions | |
| 85 | Brew 5-minute Flight potions (cannot be extended) | |
| 100 | Brew level IV potions | |

> ⚠️ `noorjobs.brewer.luck` is granted at **both** 25 and 50 in the current YAML. Likely a bug.

#### Fisher
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | 5% chance to catch double fish | `noorjobs.fisher.doublefish.basic` |
| 25 | `/waterbreathing` | `cc.waterBreathing` |
| 35 | `/dolphinsgrace` | `noormisc.effect.dolphinsgrace` |
| 50 | Fishing rods take no durability | `noorjobs.fisher.nodurability` |
| 60 | Disguise as aquatic mobs | |
| 75 | `/conduit` | `noormisc.effect.conduit` (+ aliases) |
| 85 | 25% chance to catch double fish | `noorjobs.fisher.doublefish.advanced` |
| 100 | Catch top-level dusts and cores | |

#### Digger
| Level | Perk |
|:-----:|------|
| 10 | Better rewards from common terrain |
| 25 | Excavation utility recipes |
| 35 | Improved rewards from loose stone and sand |
| 50 | Better rewards from clay and mud |
| 60 | Craft advanced digging supplies |
| 75 | Better rewards from desert materials |
| 85 | Rare excavation recipes |
| 100 | Master excavation bonuses |

#### Crafter
| Level | Perk | Granted node |
|:-----:|------|--------------|
| 10 | Better utility crafting rewards + Magic Chests | `noorjobs.crafter.magicchest` |
| 25 | Extra utility recipes | |
| 35 | Improved storage-crafting rewards | |
| 50 | Better rewards from complex recipes | |
| 60 | Craft advanced supplies | |
| 75 | Better rewards from precision crafting | |
| 85 | Rare utility recipes | |
| 100 | Master crafting bonuses | |

Crafters also get `noorjobs.crafter.campcrafting` — crafting through a NoorTowns Camp Workbench.

#### Blacksmith
| Level | Perk |
|:-----:|------|
| 10 | Better basic tool rewards |
| 25 | Metalworking recipes |
| 35 | Improved iron gear rewards |
| 50 | Better diamond gear rewards |
| 60 | Craft advanced forge supplies |
| 75 | Better armor crafting rewards |
| 85 | Rare forge recipes |
| 100 | Master blacksmith bonuses |

#### Wizard & Chef
🟡 — Both jobs are live and pay out, but **every** perk milestone (10/25/35/50/60/75/85/100)
is currently declared as `Coming Soon!`. These are the two biggest content gaps in NoorJobs.

Wizard does ship three standalone commands: `/bottlexp`, `/disenchant`, `/bookstack`.

### Boosters & Magic Chests ✅

- `/booster` — virtual job booster menu. Boosters can be given, frozen, unfrozen and
  cancelled by admins (`noorjobs.admin.booster`).
- `/magicchest` — Crafter perk. Converts single chests into Magic Chests. Output chests
  support **Auto Send to Camp Storage**, which deposits through `TownServiceBridge` and
  leaves any undeposited remainder in Magic Chest storage.

---

# Quests

✅ — **NoorQuests**. This section was previously two empty headings. Here is what actually ships.

### Daily Quests

- Each player receives **5 easy, 5 medium and 5 hard** quests per day by default.
- Reset is at **midnight in `daily.refresh-time-zone`**, default `America/New_York`.
- Pools are configured in `config.yml`; assignments persist in `player-quests.yml`.
- `/nq daily` opens the menu — 6 quests per difficulty per page, with pagination beyond that.
- `noorquests.extradaily.<amount>` adds that many quests to **every** difficulty. Nodes do
  **not stack**; the highest enabled positive value wins. `nq.prospector` is a legacy +1 alias.
- Assignments are capped by the number of unique quests in each difficulty pool.
- Newly granted bonuses expand the current day without wiping progress. Reductions apply at
  the next reset, so nothing in-flight is destroyed.

### Streaks

Completing **at least 1 easy, 1 medium and 1 hard** quest before the next reset increases the
daily streak by 1. Milestone rewards live under `streak.milestone-rewards` and run as console
commands supporting `{player}`, `{uuid}` and `{streak}`.

### Story Quests

`/quests` (alias `/storyquests`) opens a read-only questline menu — in-progress quests first
with their current objective and progress, then completed ones. Story quests are defined in
`plugins/NoorQuests/quests/` and started with `/nq quest start <player> <quest_id>` or through
`QuestServiceBridge`. Programmatic registrations may carry completion reward commands and can
be unregistered on reload without touching file-backed quests or saved progress.

### Reward table ✅

| Completion | Money | XP | Item chance |
|------------|------:|---:|-------------|
| Easy quest | $250 | 25 | 20% Common Fragment |
| Medium quest | $750 | 75 | 25% Uncommon Fragment |
| Hard quest | $2,000 | 200 | 30% Rare Fragment · 5% Rare Geode |
| One of each difficulty | $2,500 | 250 | Common Geode + streak |
| All assigned dailies | $10,000 | 1,000 | Uncommon Geode |

Money requires Vault and an economy provider. Geode rewards dispatch through NoorGeodes
console commands.

### Quest flags for other plugins ✅

`quest:<id>`, `quest:<id>:active`, `quest:<id>:started` and `quest:<id>:<objective>` resolve as
condition flags — NoorNPCs guide dialogue uses them in stage `requires` / `requires-not`.

---

# NPCs

🟡 — **NoorNPCs**. There are now **two distinct NPC systems**, and this document only ever
described one of them.

## Guide NPCs ✅ (new — was not in this doc)

Static, configured, dialogue-tree NPCs. Defined in `guide-npcs.yml` or
`plugins/NoorNPCs/NPCs/*.yml`.

- **Appearance:** any spawnable living entity via `model.entity` (default `MANNEQUIN`);
  `HORSE` colours, `WOLF`/`DOG` variants, `size-scale` from `0.0625` to `16.0`.
  Mannequin skins live in a shared `skins.yml` keyed by `skin-id`, categorised
  `male` / `female` / `unobtainable`. `/reloadnpc` applies edits without a restart.
- **Dialogue:** node/button trees with `close`, `command`, `message` and menu actions.
  Full formatting support — `&0`–`&f`, `&k`–`&r`, and hex as `&#RRGGBB`, `&x&R&R&G&G&B&B` or `<#RRGGBB>`.
- **Stage gating** via `requires` / `requires-not`:
  - `quest:<id>` and friends — NoorQuests progress
  - `event:fishing_contest_active`, `event:fishing_contest_lead`, `event:fishing_rewards` — NoorEvents
  - `permission:<node>` — any LuckPerms node
  - `tag:<id>`, `tag:any`, `tag:selected`, `tag:selected:<id>`, `tag:count>=n` — NoorChatExtras tags
- **Placeholders:** `{fishing_contest_fish}`, `{fishing_contest_heaviest}`, `{fishing_contest_leader}`,
  `{fishing_contest_region}`, `{tag}`, `{tag_name}`, `{tag_id}`, `{tag_category}`, `{tag_count}`, `{tag:<id>}`.
- **External actions:** other plugins register action types through `GuideNpcServiceBridge`.
  NoorEvents registers `event-quest`, letting any guide start a configured event quest.
- Guide dialogue stays clickable inside WorldGuard regions that deny entity interaction —
  NoorNPCs handles only the dialogue action and leaves the original event cancelled, so leads,
  recruitment and trading do **not** bypass region protection.
- Progression flags are managed with `/guideprogress <flag|unflag|reset|show> <player> <npc> [flag]`.

## Roaming / worker NPCs 🟡

This is the system originally described here. Behavior below is design intent; the parts
confirmed in code are marked.

### Behavior 📋
NPCs spawn like mobs but at a lower rate, appearing more often near town outskirts. On encounter, they may:
- **Flee** from the player
- **Stay put**
- **Attack** (only if visibly armed)

Players can kill any NPC, but killing an **innocent, unarmed NPC** lowers their reputation with all future wild NPCs. This penalty resets naturally over time, so early or accidental kills aren't permanent.

✅ `/spawnnpc` spawns a roaming NPC; `/npcspawn toggle` toggles free-roam spawning around you.

### Peaceful Interactions 📋
If an NPC engages peacefully, the player has three options:
- **Generic dialogue**
- **Invite to town** — recruit the NPC to join the player's existing town
- **Personal assistant** — available only if the player has no town

### Recruitment & Stationing 🟡

NPCs who join a town or become a personal assistant **despawn immediately**, simulating travel
to the player's home or town.

> **Changed:** they are resummoned with a **Tent**, not an "NPC Workbench".
> `/noornpcs givetent <player>` issues one. The workbench concept is gone.

Once stationed, players can access the NPC's inventory, give the NPC money, and assign tasks.
`/npcs` opens your assigned NPC list; `/npchelp` opens the help menu.

✅ **Auto storage:** `noornpcs.auto.storage` lets players route assigned-NPC job loot straight
into **Personal Campfire (camp) storage**.

✅ **Lead NPCs:** `noornpcs.lead.manage` lets a player assign and use a Lead NPC for automatic
NPC jobs. Leads may be attached to a stationary NPC but cannot drag it, and only
`noornpcs.admin.uselead` can attach or manually remove them.

### Jobs & Leveling 🟡
- NPCs can be **paid to work** at a base rate *(still TBD)*
- They perform the same jobs as players but are **limited to one job at a time**
- **Switching professions resets the NPC back to level 1**
- Higher levels increase **roll chance** for better loot per minute worked and **stamina
  duration** before the NPC tires and returns home
- Pay scales with town size and NPC level: more NPCs in town = higher base pay; each NPC level
  adds a **variable rate equal to their level %** on top of base pay

✅ `/npclevel <npc-id-or-name> <level>` sets an assigned NPC's job level (admin).

### Care & General Use 📋
- NPCs require **food and shelter** to stay loyal — neglect decays happiness until they **abandon your town**
- Every minute an NPC works, they consume **1 hunger bar's worth of food**
- Certain foods give buffs: higher **work ethic**, higher **luck** (better roll chances while working)

### Town NPC slots ✅ (new — was not in this doc)

NPCs are a **town-capped resource**. Towns track NPC slots (`/noortown npcslots`), buy more
through the `npcslots` upgrade, and each town tier sets an `npc-slot-cap`. Placing an NPC in a
town is gated by the `Add NPC to Town` role permission (default ON for founder/mayor, OFF for
other roles) and `noorpc.town.addnpc`.

## Tasks

### Building 📋
- If a player has a town with no assigned builder, they may designate an NPC as a **builder**
- NPCs have access to **10 preloaded house builds** by default
- Players may upload their own builds as **schematics** to a personal catalogue *(may be level-dependent)*
- As NPCs complete builds they **level up**, reducing future build times
- Server-provided builds have **predetermined build times between 2–5 hours**
- Custom builds: **base 2 hours + 10 seconds per block placed** (at level 1)

✅ NoorTowns ships the supporting build tooling already: `/buildplacementtool`, `builds.yml`
metadata with `/reloadtownbuilds`, schematic material breakdown via `/testschemmats`, and
`/ignorebuildrequirements` for testing. `/noortown build requirements <Builder Workshop>`
reports what a build needs.

---

# Towns, Camps & Nations

✅ — **NoorTowns**. This is the single most under-described system in the old document.
Personal Camps did not appear at all, and they are now the entry point to the whole feature.

## Personal Camps ✅ (new — was not in this doc)

Every player gets a solo claim before they ever touch a town.

- A **Personal Campfire** item creates the claim. `/noortowns givecampfire` issues one.
- Lighting it claims a **centered 3×3 chunk area**. Breaking the anchor removes the claim.
- Claimed chunks block non-owners from placing/breaking.
- Expiration is configurable via `camp.claim-expiration-seconds`.
- Right-click the campfire, or `/camp`, to open the camp menu: teleport to marker, reset
  marker to your current in-claim location, rename camp (updates the campfire hologram),
  toggle claim border particles, set spawn.
- Default size scales with permission: `noortowns.camp.size.<3-10>` → 3×3 up to 10×10 chunks.
  This is what the store's "3x3 / 4x4 / 5x5 / 7x7 / 10x10 Area of Personal Land" perks map to.
- Members: `/camp add|ban|unban|accept|deny|leave`.

### Camp storage ✅

- `/cs` or `/camp storage` — searchable (`/cs search <query>`) personal storage.
- **Transaction logs** for every deposit and withdrawal, opened with the `Storage Logs`
  button. Custom items log as themselves — item name, custom item id, base material and item
  model — not a bare material. Retention defaults to **1 week**
  (`camp.storage-logs.retention-days`); `camp.storage-logs.visible-to-players` decides whether
  members see them or they stay admin-only.
- **Salvage:** `noortowns.camp.storage.salvage` allows `/cs salvage` — one rarest-first batch,
  capped by `camp.storage-salvage.max-items-per-run` (default `1000`). **Noorish and Divine
  items are never selected.**
- Camp storage is the **settlement layer for the economy** — market purchases are delivered
  here and market sells are drawn from here (see [Economy](#economy-market--auction-house)).

### Camp transfers ✅

`/noortowns admin transfercamp <sender> <recipient>` requests a permanent merge; the online
recipient must `/camp transfer accept` (or `deny`) within **120 seconds**.

On accept: storage is consolidated, the **higher** storage/size/duration upgrades are retained,
environment entitlements are unioned without copying active selections. A recipient with no
live camp receives account storage and upgrades but no camp is created. The sender's campfire,
claim, tents, settings, access lists, profiles and storage history are removed; their assigned
NPCs stay theirs but go camp-inactive. Merged storage may sit **above capacity** — withdrawals
still work, deposits stay blocked until usage drops below the cap. Accepted transfers use a
durable recovery journal and lock both accounts from conflicting camp actions until cleanup completes.

## Wilderness PvP ✅ (new — was not in this doc)

Consent-based, through `/passive`:

- Passive mode is **on by default** and persists across reconnects and restarts.
- PvP in unclaimed land works **only when both players have disabled passive mode**.
- A player cannot re-enable passive mode until **15 seconds** after the last PvP damage they
  dealt or received.
- Town and camp claims keep their existing combat permissions; arena plots keep arena PvP.
- `pvp.force-disabled` still blocks claimed-area PvP except arenas, but mutually non-passive
  players can fight in the wilderness regardless.

## Towns ✅

Towns organize groups of players around shared goals — building, job grinding, or NPC
farming/production — and encourage collaboration.

- `/noortown create <town_name>` (requires `noortowns.town.create`). No spaces in the input —
  underscores render as spaces, hyphens are allowed.
- Claims: `/noortown claim` / `unclaim`, with per-role permissions for place/break/ignite,
  block interactions, workbench usage, entity interactions and entity attacks.
- **Roles & permissions** are per-town and editable in the Town Menu's `Roles & Permissions`
  submenu. Hierarchy is configured in `config.yml` under `town.roles`, thresholds under
  `town.role-actions`.
- An **`Outsider` role** governs non-members, so each town sets its own outsider rules.
- **Bank:** `/noortown bank deposit|withdraw <amount>`, with town **upkeep** (`/town upkeep runnow` for admins).
- **Vault:** `/noortown vault` (gated by the `Use Town Vault` role permission).
- **Outposts:** `/noortown outpost`, `/tpoutpost <name>`. **Regions:** `/noortown region`, `/tpregion <name>`.
- **Spawn:** `/town spawn [town_name]`; spawn access is managed separately from invite-only
  joining via `/town spawnaccess <public|private>`.
- Directory `/noortown list`, info `/noortown info <town_name>`, menu `/noortown menu`.
- `/tfly` — flight inside your town or personal camp.

### Town tiers ✅ (new — was not in this doc)

Configured under `town.tiers`. Each tier can require:

`required-players` · `required-claims` · `required-npcs` · `required-bank-money` — and sets an `npc-slot-cap`.

### Town Upgrades ✅

Configured under `town.upgrades`. The real upgrade axes are:

| Upgrade | Configured as |
|---------|---------------|
| **Claims** | price + extra claims per level |
| **NPC Slots** | price + extra slots per level |
| **Vault** | price + extra double chests per level |

Bought with `/noortown upgrades buy <claims|npcslots|vault>` or the `Town Upgrades` button.

> **Design intent** (kept for reference; the numbers below are *not* in config yet):
>
> | Upgrade | Increment | Starting Cost | Max |
> |---------|-----------|---------------|-----|
> | Claims | +100 per upgrade, increasing in cost each time | — | 5,000 (49 upgrades) |
> | Outposts | 1 at a time, each costing twice the last | $25,000 | 10 |
> | Town Vaults | 1 at a time, each costing twice the last | $5,000 | 10 |
>
> ⚠️ Note the mismatch: **Outposts are a feature, not an upgrade axis**, and **NPC Slots are
> an upgrade axis the design never listed**. Reconcile before pricing.

## Nations

📋 — Designed only. No `NoorNations` repository exists.

Intent: Nations **ally towns** and allow interaction within each other's territory.
Governance features are planned but **not scheduled for Open Beta**.

---

# Economy, Market & Auction House

✅ — **NoorEconomy** (the repo is `NoorEconomy`; this document previously called it "NoorEco"
and listed its status as "Unsure"). It is one of the most complete plugins in the suite.

Vault-backed: NoorEconomy coordinates every enriched money mutation while the captured Vault
provider (e.g. EssentialsX Economy) remains the player-balance store. It can load before a
provider and finishes enabling when one registers.

### Authoritative audit ✅

Auditing is **enabled and fail-closed by default** — an unhealthy integrity key, writer,
staging spool, queue or index prevents mutation outright. `audit.enabled: false` permits
unaudited passthrough and logs a prominent warning.

- Append-only YAML histories per player and per account type
- `audit-index.db` — rebuildable byte-offset index
- `audit-integrity.key` — separate 256-bit HMAC key, never silently rotated if history exists
- Restarts reuse authenticated per-stream checkpoints and validate each stream's durable tail,
  so enable no longer waits for a full historical scan; full verification continues in the
  background while mutations stay fail-closed until it succeeds

Admin: `/nooreconomy audit [player] · transaction <uuid> · health · verify <player> · export <player> · rebuild-index`

> The Vault compatibility proxy records callers that still mutate through Vault as
> legacy/unattributed. Known NoorEconomy, NoorTowns, NoorRanks and NoorJobs mutation sites use
> the NoorCore transaction bridge directly.

### Commodities Market ✅ (new — was not in this doc)

`/market`, `/market orders`

- Buy orders are created from a categorized vanilla item catalog and **reserve money immediately**
- Sell orders and quick sells are **camp-storage-only**, moving items into escrow through an
  idempotent NoorTowns withdrawal
- Matching is **price-time priority** with partial fills
- Purchases are delivered to **personal camp storage**; anything that doesn't fit stays queued
- Uncertain operations are held in durable **quarantine** while unrelated players keep trading;
  reconciliation is a two-step prepare/confirm flow committed atomically to SQLite
- State lives in `market.db` (SQLite, WAL enabled) so the website can read the same database

### Auction House ✅

`/auctionhouse`, `/ah` — the `/ah` the Merchant rank and store ranks sell.

- Browse and buy any player's listing through one audited buyer-to-seller transaction
- Right-click your own listing to cancel and reclaim
- List via the emerald button → select stack → price and quantity dialog
- Expired/cancelled listings return immediately, or queue until there's inventory space
- Listing caps: `nooreconomy.auctionhouse.listings.<amount>`, `.*` for unlimited,
  `default-listing-limit` for everyone else (`-1` unlimited, `0` browse/buy only)
- `listing-duration-hours` and `cleanup-interval-seconds` are configurable

---

# Items, Sets & Blueprints

✅ — **NoorItems**. YAML-defined custom items with no code required.

- **Items:** id, name/flavor/lore with colours, type (`Item`/`Tool`/`Armor`), base material,
  armor slot / tool type, custom durability, `texture_id`, furnace `fuel` + `burntime`,
  attributes (with operations and equipment slot groups), and **behaviors** — trigger/action
  pairs like `Interact→Explode`, `Hurt→Heal`, `Attack→Effect`.
- **Recipes:** inline on the item or standalone in `recipes/`. Shaped, Shapeless, Smelting,
  Blasting, Smoking, Campfire, Stonecutting, Smithing. Optional `recipe_book` discovery and
  `requirements.Permissions` gating.
- **In-game editors:** `/itemeditor` and `/recipeeditor` — build a recipe visually, save, and
  it registers immediately and writes `recipes/{id}.yml`.
- **Kits:** `kits/` YAML with cooldowns and auto-generated `NoorItems.kit.{ID}` permissions. `/kit`.
- **Enchantments:** `/noorenchant <enchantment> [level]`, with `nooritems.enchantments.basic`
  controlling custom enchant acquisition from enchanting tables.
- **Item updater:** `/updateinventory [player]` and `/updatestorage [player]` re-run the
  updater across an inventory or a player's camp storage.
- **Texture credits:** `/credit` shows the credit for the held item; `/creditbook` opens the
  credit book for every registered item.

### Custom, Tiered & Powerful Tools

Mobs have a **very rare chance** to drop custom tools and armor, including ultra-powerful
variants. ✅ This is the NoorDrops runtime, now living inside NoorGeodes — see below.

### Blueprints 🟡

Blueprints are a gacha system giving you a random item from a given set.

✅ **What exists:** blueprint sets are implemented as **NoorItems item sets**. Each set carries
a **scrap** item and a **blueprint** item, plus a weighted item list (chance + amount per
entry). NoorCore exposes them through `ItemServiceBridge.getBlueprintSets()` →
`BlueprintSetSnapshot(setId, scrapItemId, blueprintItemId, scrapItem, blueprintItem)`, and
`getItemsFromSet(setId)` returns the weighted roll table. `/collection [set]` opens the
collection menu.

📋 **What does not exist yet:** the currency and pity economy around it —

> You can buy a random blueprint from the crafter in exchange for 16 assorted scraps. You can
> unlock the ability to buy specific blueprints from the crafter in exchange for 2 stacks of
> the corresponding scrap; this costs 32x of that scrap per blueprint.
>
> Rolls have a pity system where each roll gives a set amount of Essence. Using a set amount of
> Essence you can conjure any given item in the set unless a given item has conjuring disabled.
>
> The function for a given roll's Essence, where E is the resulting essence, m is the max
> amount given (when the chance is 100%), C is the normalized percentage, and k is the curve.
> m defaults to 50, k defaults to 1.5:

$E=floor\left(\left(m-1\right)\cdot\left(\frac{C}{1}\right)^{k}\right)+1$

> There is no Essence item, no conjuring, no pity counter and no scrap→blueprint exchange in
> the codebase. The scrap and blueprint *items* exist; the loop between them does not.

---

# Geodes, Drops & Salvage

✅ — **NoorGeodes**. Note: **NoorDrops no longer exists as a separate plugin** — its runtime
was absorbed into NoorGeodes, which keeps `/noordrops`, the `plugins/NoorDrops` data folder and
the `noordrops:*` persistent data keys for compatibility.

### The loop ✅

1. A player receives an eligible **tiered relic** from the drops system (mob drops) or a job perk.
2. They open the salvage menu — from the configured salvage block, or `/salvage` with permission.
3. Eligible relics in the GUI convert into **fragments** and **catalysts**.
4. **8 fragments + 1 catalyst** of the same tier craft **1 geode**.
5. Right-clicking a geode consumes it for a random reward from that tier.

> **Terminology fix:** this document previously said "**shards**/catalysts". The item is a
> **fragment**. Salvage odds are `FRAGMENT_CHANCE = 90.0`, so catalysts are the remaining 10%.

### Tiers ✅

The old text said the rarity name was "TBD". It is not:

```
common · uncommon · rare · epic · legendary · noorish · divine
```

- **Divine is blocked from salvage** — Divine relics are returned rather than broken down.
- **Divine is terminal** — there is no upgrade recipe out of it.
- Fragments and catalysts upgrade to the next tier with a **full 3×3 grid of the lower tier**,
  so 9 Noorish fragments make 1 Divine fragment (same for catalysts).
- Noorish and Divine items are also excluded from NoorTowns camp-storage bulk salvage.

### Salvage GUI ✅

A double chest titled `Relic Salvage`. Top five rows are input; the bottom row is protected UI
with the `Salvage` button in slot 49. On click, every eligible relic is consumed, each rolls
fragment-or-catalyst, invalid items are returned, rewards go to inventory or drop naturally if
full, and the menu closes. Closing without pressing Salvage returns everything.

An item is salvageable when it carries supported persistent data tags — `noordrops:tier` +
`noordrops:item_id`, or `nooritems:drop_tier` + `nooritems:drop_item_id` — and its tier is
valid and not blocked.

### Job-driven fragment drops ✅

Six jobs can roll fragments and catalysts while working, gated per tier:

`noorjobs.<miner|farmer|lumberjack|rancher|fisher|digger>.geodes.<rare|epic|legendary|noorish>`

### Cross-plugin drop sourcing ✅

A NoorDrops tier item can pull its base `ItemStack` from elsewhere instead of being rebuilt:

```yaml
items:
  - id: cool_hat_drop
    item: "nooritems:cool_hat"        # NoorItems registry (direct lookup)
  - id: starter_rod_drop
    item: "noorfish:Basic_Rod:Basic_Line:Basic_Bobber"   # assembled rod via FishServiceBridge
  - id: plain
    item: "minecraft:diamond"
```

Any field you set still overrides the base. Always keep a `material` fallback — an unresolvable
source falls back to it.

### Perfect Items 📋

Not implemented. No `perfect` flag, star marker or 2% roll exists in NoorGeodes.

Intent: some incredibly rare equipment displays a **★ star** marking it **"Perfect"**. Every
Geode opened has a **2% chance** of yielding one.
- Perfect items have **all base enchantments upgraded by one level**
- Perfect items do **not** have Unbreaking — they are permanently **Unbreakable**

---

# Fishing

✅ — **NoorFish** (version `1.0.0`). Listed in this document as "Post-Beta / Not Started";
it is neither.

- Custom fish with rarity-in-location and size driving reward math. All payout tuning lives in
  NoorFish and is exposed to other plugins through `FishServiceBridge`.
- **Modular rods:** `/rod <part|build|list>` — rods are assembled from a **rod, line and
  bobber**, and an assembled rod is addressable as `noorfish:<Rod>:<Line>:<Bobber>`.
- `/collection` — fish collection menu.
- `/fishbundle` — toggle sending catches straight into your first bundle.
- `/fishdebug <beta|tune>` — live bobber physics and bite tuning.
- Feeds catches into NoorEvents fishing contests via `FishingContestServiceBridge`.
- `loadbefore: Geyser` so its custom items reach Bedrock.

---

# Chat & Social

## NoorChatExtras ✅ (new — was not in this doc)

A full multi-channel chat system.

**Channels:** `global` `local` `trade` `town` `party` `staff` `content` `mod` `admin` `team`
`benefactors` — each with its own command, alias and permission, plus private `/tell` + `/reply`.

**Features:**
- `/chat` channel menu, `/chatsettings` personal display settings
- **Custom chat tags** — `/tags [menu|list|set <tag>|clear]`, exposed to other plugins through
  `ChatTagServiceBridge`. Tags carry `id`, `prefix`, `displayName` and `category`; prefixes are
  legacy-formatted with `&` codes including `&#RRGGBB` hex. Whether a player has a tag is a
  **permission check**, so player-scoped lookups resolve only while that player is online.
  Crate-unlockable tags can be granted at random with `noorchatextras.tags.randomgrant`.
- **Interactive placeholders** — `[item]`, `[inv]`, `[money]`, previewed with `/nceview <token>`
- **Emoji glyphs** baked into the resource pack by NoorPack (`ChatGlyphServiceBridge`)
- `/show` held item in chat
- **Moderation:** `/chatspy [channel|all|off]` per-channel spy, `/slowmode <player>`
  (one global message per 10s for 1 hour), `/helpop`, `/ignore` + `/unignore` + `/ignorelist`
  with `noorchatextras.admin.ignorebypass`
- **Prefix arrangements:** `staff`, `premium`, `regular`
- Trade-chat cooldowns with a bypass node

## NoorFriends ✅ (new — was not in this doc)

- `/friends [list|add|accept|deny|remove|requests|settings]`
- `/party <create|invite|accept|deny|leave|kick|disband|list|invites>` — party **chat** itself
  is the NoorChatExtras `/pc` channel
- `/frtp <friend>` — teleport to a friend who has friend teleport enabled

## NoorFamilies ✅ (new — was not in this doc)

Parental controls and child-account chat auditing. `/noorfamilies` lets staff set up and manage
child profiles (`noorfamilies.admin`) and lets guardians open controls for linked child accounts
(`noorfamilies.use`), including dated log review. Publishes `FamilyServiceBridge`, which also
carries the teleport-policy hook used for spouse requests and shared homes.

## Marriage 📋

`MarriageServiceBridge` is in NoorCore with 23 methods covering cached active-marriage, spouse,
shared-home and **Bond** lookups — all immutable, Bukkit-free and safe from any Folia thread —
but **no NoorMarriage repository exists**. See [Open Questions](#open-questions--design-drift).

---

# Events

✅ — **NoorEvents** (listed here as "Post-Beta / Not Started"; it ships a whole seasonal event).

- Backend timed event management with admin start/end/time/status controls and **private test
  sessions** (`noorevents.admin.test`)
- **Summer Event:** `/event` menu, `/eventquests`, `/eventshop` token shop, and **Summer Tokens**
  as an event currency — `/eventtokens [balance|pay <player> <amount>]`, so tokens are transferable
- **Fishing contests:** started/ended/redeemed by staff, fed by NoorFish catches, with a
  **bingo board** (`/fishbingo`) and live state readable by guide NPCs
- Registers the `event-quest` guide action so any NPC can start an event quest
- The token shop's products name NoorItems items with `nooritems:` and keep a vanilla material
  as fallback

---

# Resource Pack & Bedrock

✅ — **NoorPack** (new — was not in this doc). This is the plumbing that makes custom content
work on both editions.

- **Zero-config models:** NoorPack watches its asset folder and, for every texture under
  `assets/<namespace>/textures/item/`, generates the model *and* the 1.21.4+ item definition
  automatically.
- **Chat glyphs:** scans `textures/chat/emoji/` and `textures/chat/tag/`, assigns each image a
  private-use codepoint and writes the bitmap font definitions — a picture in chat is a
  *character*, drawn wherever that codepoint sits.
- **Blockbench models:** `/spawnmodel <model> [scale] | bind | unbind | list | info | parts | clear`
  and `/modelanim play <animation> [layer/speed/weight/fade/blend/loop/bones] | stop | layers | list`
  — multi-layer animation on spawned or mob-bound models.
- `/reloadpack` reloads the server resource pack.

### Why Bedrock needs this ✅

On Java a custom item is just a base material wearing a different model, so the texture is
enough. Bedrock can only draw an item it has been **told about as its own item**. Geyser
bridges that by swapping the Java item for a purpose-built Bedrock one — but it needs to know
which base material each custom model sits on, and only NoorItems knows that.

So NoorItems hands its whole catalogue to NoorPack a tick after items load (and after every
`/reloaditems`), and NoorPack builds the Bedrock pack and writes the Geyser mappings. Armor
carries its slot and `armorModel` across so it renders on a Bedrock player's body.

- **Namespace rule:** NoorItems' `pack_provider` must match NoorPack's `pack.item-namespace`
  (both default `noorpack`). That key is what ties a Java item to its Bedrock counterpart.
  NoorGeodes and NoorDrops read it through `PackProvider` rather than defaulting on their own,
  so no plugin drifts onto a namespace NoorPack never wrote a mapping for.
- **Registration matters:** a texture nothing registers is **invisible to Bedrock**. NoorGeodes
  registers all three item families (fragments, catalysts, geodes) on enable and on reload, and
  names any missing model in the startup log.
- Geyser reads its folder once at startup, so a rebuild reaches Bedrock players on Geyser's
  next restart. NoorPack, NoorItems, NoorGeodes and NoorFish all declare
  `loadbefore: [Geyser, Geyser-Spigot]`.

---

# Moderation & Safety

## NoorAdmin ✅ (far beyond "Moderation Tools")

A staff **reputation and punishment** system, not just a command pack.

- Punishments: `/warn` `/kick` `/mute` `/tempmute` `/unmute` `/ban` `/tempban` `/unban`
  `/ipban` `/tempipban` `/ipunban` `/ipscan`
- **Staff reputation scoring** with configured infractions, manual `add`/`deduct`/`set`,
  audit `history`, and `undo` to reverse active entries
- **Freeze/unfreeze** with notifications and a `nooradmin.freeze.exempt` node
- **Instance switching:** `/toggleadmin` and `/togglecontent` switch between a staff member's
  player, admin and content instances
- Full GUI (`/nooradmin menu`) with per-area permissions

## NoorCore AntiCheat ✅

NoorCore itself carries the detection layer: combat, interaction, inventory, movement and
ping-anomaly trackers feeding a `FlagSystem` with typed flags and `StaffAlertLevel` escalation.

## NoorGuide ✅ (new — was not in this doc)

A **reviewed, local** knowledge base — deliberately not a live LLM.

- `/question <question>` / `/ask` searches published answers
- Staff workflow: import → draft → submit → review (approve/reject/archive/restore) → publish,
  with alias editing, staff answering of unresolved queries, and export/reindex/statistics
- Permissions split across `noorguide.use`, `.staff.edit`, `.staff.review`, `.staff.answer`, `.admin`

---

# Donator Store

## Ranks
|   Rank    | Price | Perks                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|:---------:|:-----:|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Starling  |  $10  | +5 Homes <br> 3x3 Area of Personal Land <br> +1 Job Focus <br> +1 Player Vault <br> All virtual stations <br> `/hat` <br> `/glow` (color of rank) <br> Gradient `/nick` <br> Access to Donator Chat                                                                                                                                                                                                                                                                                                                                                           |
|   Comet   |  $20  | +10 Homes <br> 4x4 Area of Personal Land <br> +2 Job Focuses <br> +2 Player Vaults <br> +1 Player Warp <br> AH access <br> +5 AH listings <br> Access to ChestShops <br>  `/hdb` <br>  `/ptime` `/pweather` <br> All virtual stations <br> `/hat` <br> `/glow` (color of rank) <br> Gradient `/nick` <br> Access to Donator Chat                                                                                                                                                                                                                             |
|  Nebula   |  $35  | +15 Homes <br> 5x5 Area of Personal Land <br> +3 Job Focuses <br> +3 Player Vaults <br> +2 Player Warp <br> AH access <br> +8 AH listings <br>  `/feed` <br>  `/near` <br>  `/nv` <br>  `/back` <br> Access to ChestShops <br>  `/hdb` <br>  `/ptime` `/pweather` <br> All virtual stations <br> `/hat` <br> `/glow` (color of rank) <br> Gradient `/nick` <br> Access to Donator Chat                                                                                                                                                                       |
|  Aurora   |  $50  | +20 Homes <br> 7x7 Area of Personal Land <br> +4 Job Focuses <br> +4 Player Vaults <br> +3 Player Warp <br> AH access <br> +12 AH listings <br> Keep Exp on death <br> +1 `/rtp biome`/day <br>  `/top` <br>  `/feed` <br>  `/near` <br>  `/nv` <br>  `/back` <br> Access to ChestShops <br>  `/hdb` <br>  `/ptime` `/pweather` <br> All virtual stations <br> `/hat` <br> `/glow` (color of rank) <br> Gradient `/nick` <br> Access to Donator Chat                                                                                                         |
| Celestium |  $80  | +25 Homes <br> 10x10 Area of Personal Land <br> +5 Job Focuses <br> +5 Player Vaults <br> +4 Player Warp <br> AH access <br> +15 AH listings <br> Keep Inventory <br> `/speed` <br> `/jumpboost` <br> `/repair` (30Min cooldown) <br> `/tfly` <br> +1 `/rtp biome`/day <br> `/top` <br> `/feed` <br> `/near` <br>  `/nv` <br>  `/back` <br> Access to ChestShops <br>  `/hdb` <br>  `/ptime` `/pweather` <br> All virtual stations <br> `/hat` <br> `/glow` (color of rank) <br> Gradient `/nick` <br> Access to Donator Chat <br> Access to Celestium Chat |

**Node mapping ✅**
- *N×N Area of Personal Land* → `noortowns.camp.size.<3-10>` — these land up exactly on the
  implemented camp sizes (3, 4, 5, 7, 10).
- *+N AH listings* → `nooreconomy.auctionhouse.listings.<amount>`
- *Donator / Celestium Chat* → NoorChatExtras channels. `benefactors` is the shipped
  supporter channel (`nce.channel.benefactor`); `content`, `mod`, `team` also exist.
- *Gradient `/nick`* → NoorChatExtras handles hex/gradient formatting nodes.
- ⚠️ *"+N Mastery Paths"* was renamed **Job Focus** here because [Mastery Paths no longer
  exist](#jobs--promotions). **This perk needs redefining** — all 14 jobs are already available
  to everyone, so there is nothing to unlock. It could become promotion points, promotion track
  slots, or be dropped. Design decision required.

## Subscriptions
| Tier | Price/Month | Perks                                                                                                                                                                                                                                                        |
|:----:|:-----------:|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  I   |     $5      | Tiered Cosmetics <br> Discord Role & Chat <br> Gradient `/nick` <br> `/hat` <br> +1 Large Weekly [Crate](#crates) <br> 1x Weekly 5% Global Universal Jobs XP Booster (4h a piece, cannot be paused, cannot be sold) <br> Access to Sub Feature Votes         |
|  II  |     $15     | Tiered Cosmetics <br> Discord Role & Chat <br> Gradient `/nick` <br> `/hat` <br> +3 Large Weekly [Crates](#crates) <br> 1x Weekly 10% Global Universal Jobs XP Booster (4h a piece, cannot be paused, cannot be sold) <br> Access to Sub Feature Suggestions |
| III  |     $25     | Tiered Cosmetics <br> Discord Role & Chat <br> Gradient `/nick` <br> `/hat` <br> +5 Large Weekly [Crates](#crates) <br> 1x Weekly 15% Global Universal Jobs XP Booster (4h a piece, cannot be paused, cannot be sold) <br> Access to Sub Feature Suggestions |

✅ The delivery mechanics for all three exist: NoorCrates **weekly virtual-key deliveries**
handle the crate grants, and NoorJobs **virtual boosters** (freezable/cancellable, admin-granted)
handle the XP boosters. "Cannot be paused" maps to simply not granting the freeze capability.

## Crates
|       Tier       | Amount | Price |
|:----------------:|:------:|:-----:|
|      Small       |   1    |  $1   |
|      Small       |   5    |  $4   |
|      Small       |   10   | $7.50 |
|      Small       | 1Serv  |  $10  |
|      Small       | 5Serv  |  $40  |
|                  |        |       |
|      Medium      |   1    |  $2   |
|      Medium      |   5    |  $8   |
|      Medium      |   10   |  $15  |
|      Medium      | 1Serv  |  $20  |
|      Medium      | 5Serv  |  $80  |
|                  |        |       |
|      Large       |   1    |  $3   |
|      Large       |   5    |  $12  |
|      Large       |   10   |  $20  |
|      Large       | 1Serv  |  $30  |
|      Large       | 5Serv  | $120  |
|                  |        |       |
| Monthly Cosmetic |   1    |  $5   |
| Monthly Cosmetic |   5    |  $20  |
| Monthly Cosmetic |   10   |  $35  |

✅ **NoorCrates** backs all of this and is feature-complete:

- Crates and keys are individual YAML files (`crates/<id>.yml`, `keys/<id>.yml`); legacy
  `crates.yml` is auto-imported and backed up
- **Physical keys** are real items tagged with a `PersistentDataContainer` — a renamed item
  without NoorCrates data will not validate. **Virtual keys** are stored per player
- Keys can restrict which crates they open (`allowed-crates`) and enable physical/virtual independently
- **Weighted reward pools:** pick a pool by pool weight, then a reward inside it by reward weight
- Full GUI editor (`/nc editor`), preview, holograms, per-block particles, crate models
- **"Buy for all" / server-wide grants** are covered by `manual-key-distributions.yml` — standalone
  configurable commands that grant virtual keys to every tracked player with a given LuckPerms
  permission, with a confirmation step, bounded async permission checks, and offline delivery
- **Weekly deliveries** (`weeklycrates.yml`) with `America/New_York` scheduling, a delivery
  window, and persisted entitlements for players offline up to `offline-player-lookback-days`
- **Rank-up rewards:** on a NoorRanks rank-up, grant keys by rank (default: `explorer` → 1 `bonfire-key`)

> ⚠️ The crate **tier names** here (Small/Medium/Large/Monthly Cosmetic) are not yet the crate
> IDs in config — shipped examples use `starter`, `premium`, `bonfire-key`. Naming needs a pass.

## Boosters
- 10% Global/Universal Job EXP Booster (1h): $10
- 10% Global/Universal Job Money Booster (1h): $10

- 25% Global/Universal Job EXP Booster (1h): $25
- 25% Global/Universal Job Money Booster (1h): $25

- 50% Global/Universal Job EXP & Money Booster (1h): $100

✅ Backed by NoorJobs virtual boosters (`/booster`).

## Tools (Names and enchants TBD)
- Individual - $2.00
- Full Bundle (Full Armor, Sword, Pick, Axe, Shovel, Hoe, 2 Wildcards) - $20.00

✅ Deliverable through NoorItems + the NoorDrops `item:` sourcing described
[above](#cross-plugin-drop-sourcing).

## Cosmetics
📋 — Still TBD, and there is **no `NoorCosmetics` repository**. The pieces that exist today:
NoorChatExtras chat tags and emoji, NoorPack Blockbench models and animations, and `/glow` /
`/nick` formatting. Trails and cosmetic slots have no owner plugin yet.

---

# Structure

Statuses below are read from the code as of the last verification pass, not from intent.

**Status key:** `Prod Ready` = complete and deployable · `Mature` = broad feature set, actively
developed · `Started` = real functionality, incomplete · `Initialized` = scaffold/stub ·
`Not Started` = no repository.

| Priority  |     Group     | Money | Status | Plugin | Depends | Features |
|:---------:|:-------------:|:-----:|:------:|:------:|---------|----------|
| Open Beta | Backend | Free | **Prod Ready** | **NoorCore** | — | Service registry & 21 bridges <br> Event bus <br> Feature flags <br> AntiCheat + logging <br> Shared menu framework <br> Shaded FoliaLib <br> Economy transaction/audit contracts |
| Open Beta | Backend | Free | **Started** | **NoorMisc** | NoorCore | `/espeed` `/mfly` `/logclimb` `/rename` <br> Portal control <br> Horse mounts <br> Store announcements <br> YAML basic commands <br> `/death` `/discord` `/store` `/drestart` |
| Open Beta | Backend | Free | **Initialized** | **NoorUtils** | NoorCore | ⚠️ Currently a stub — only `/pingnoorutils`. Intended: utilities, formatting, validation, logging, metrics, sim tools |
| Unknown | Backend | Free | **Started** | **NoorMenus** | NoorCore | `/menu` main menu <br> Live menu editor `/menuedit` <br> Legacy `cc.*` permission aliases |
| Post-Beta | Backend | Free | **Not Started** | ~~NoorData~~ | — | ⚠️ No repo. SQL/async data work currently lives per-plugin (SQLite in NoorEconomy, MySQL in NoorNPCs, YAML elsewhere) |
| Unknown | Backend | Free | **Started** | **NoorWeb** | NoorCore, Vault | Paper plugin for website integration. Market data is already exposed via `market.db` (SQLite/WAL) |
| Unknown | Backend | N/A | **Started** | **NoorBot** | — | Python Discord bot — tickets, transcripts, forums, audit, embeds, UI components |
| | | | | | | |
| Open Beta | Economy | Web API | **Mature** | **NoorEconomy** | NoorCore, **Vault** | Vault-backed currency <br> Fail-closed authoritative audit (HMAC + append-only YAML + SQLite index) <br> Commodities market with price-time matching, escrow & quarantine <br> Player auction house |
| Open Beta | Economy | Free | **Prod Ready** | **NoorJobs** | NoorCore | 14 jobs <br> XP, levels & payouts <br> Promotion tracks (I–IV) <br> Virtual boosters <br> Magic Chests <br> Leaderboards & admin panels <br> Payout history + undo |
| Post-Beta | Economy | Paid | **Not Started** | ~~NoorMastery~~ | — | ⚠️ **Superseded.** Replaced by the promotion system inside NoorJobs |
| | | | | | | |
| Open Beta | Monetization | Paid | **Started** | **NoorRanks** | NoorCore | `/ranks` `/rankup` <br> Rank menu <br> Rank-up events consumed by NoorCrates <br> ⚠️ No prestige, no store-rank handling yet |
| Post-Beta | Monetization | Paid | **Prod Ready** | **NoorCrates** | NoorCore | Per-file crates & keys <br> Physical + virtual keys <br> Weighted reward pools <br> GUI editor, previews, holograms <br> Weekly deliveries <br> Manual mass distributions <br> Rank-up rewards |
| Post-Beta | Monetization | Paid | **Initialized** | **NoorSupporter** | NoorCore | Only `/vote` so far. Intended: vote logic, subscription perks, booster distribution, Discord sync |
| | | | | | | |
| Open Beta | Towny | Paid | **Mature** | **NoorTowns** | NoorCore | Personal camps + campfire claims <br> Camp storage, logs & salvage <br> Camp transfers/merges <br> Consent-based wilderness PvP <br> Town claims, roles & Outsider role <br> Tiers, upgrades, bank, upkeep <br> Vaults, outposts, regions <br> Schematic build tooling |
| Post-Beta | Towny | Paid | **Not Started** | ~~NoorNations~~ | — | ⚠️ No repo. Nation creation, alliances, chat, shared perms |
| Unknown | Towny | Free | **Not Started** | ~~NoorWarps~~ | — | ⚠️ No repo. Player/town/nation warps, rank-based limits |
| Open Beta | Towny | Paid | **Mature** | **NoorNPCs** | NoorCore | Guide NPCs with gated dialogue trees <br> Entity models, variants, scaling, shared skins <br> Roaming NPC spawning <br> Assigned NPCs, levels, tents <br> Lead NPCs <br> Auto-send loot to camp storage |
| | | | | | | |
| Post-Beta | Entities | Free | **Started** | **NoorEntities** | NoorCore | Custom entity spawning (`/noorentityspawn`) <br> Custom mannequin brains (`/teach`) <br> `EntityServiceBridge` |
| | | | | | | |
| Open Beta | Items | Free | **Prod Ready** | **NoorItems** | NoorCore | YAML items, tools, armor <br> Attributes & trigger/action behaviors <br> Standalone + inline recipes (8 types) <br> In-game item & recipe editors <br> Kits, item sets, blueprint sets <br> Custom enchantments <br> Item updater, texture credits |
| Open Beta | Items | Paid | **Prod Ready** | **NoorGeodes** | NoorCore, **NoorItems** | 7-tier fragment/catalyst/geode economy <br> Salvage GUI + salvage block <br> Tier upgrade recipes <br> **Absorbed NoorDrops runtime** <br> Cross-plugin `item:` drop sourcing |
| Open Beta | Items | Free | **Started** | **NoorPack** | NoorCore | Auto model + item-definition generation <br> Chat glyph fonts <br> Blockbench model spawning & animation <br> Geyser/Bedrock mappings |
| Open Beta | Items | Free | **Started** | **NoorPickup** | NoorCore | Folia-safe AutoPickup for direct player drops <br> Filter menus |
| Post-Beta | Items | Paid | **Not Started** | ~~NoorCosmetics~~ | — | ⚠️ No repo. Cosmetic items and slot handling |
| | | | | | | |
| Post-Beta | Feature | Paid | **Mature** | **NoorFish** | NoorCore, **NoorItems** | Custom fish + rarity/size reward math <br> Modular rods (rod/line/bobber) <br> Collection menu <br> Bobber physics tuning <br> Feeds NoorEvents contests |
| Post-Beta | Feature | Paid | **Not Started** | ~~NoorDungeons~~ | — | ⚠️ No repo. Generation, themes, rewards, bosses |
| Post-Beta | Feature | Paid | **Not Started** | ~~NoorCooking~~ | — | ⚠️ No repo. Partly superseded by the `chef` job + NoorItems recipes |
| | | | | | | |
| Open Beta | Miscellaneous | Paid | **Prod Ready** | **NoorQuests** | NoorCore, **Vault** | Daily quests by difficulty <br> Streaks + milestone rewards <br> Story questlines <br> Rank-scaled quest counts <br> NPC quest hooks via `QuestServiceBridge` |
| Post-Beta | Miscellaneous | N/A | **Mature** | **NoorEvents** | NoorCore | Timed event framework + test sessions <br> Summer Event: menu, quests, token shop, transferable tokens <br> Fishing contests + bingo |
| Open Beta | Miscellaneous | Free | **Mature** | **NoorAdmin** | NoorCore | Full punishment suite incl. IP bans & scans <br> Staff reputation scoring <br> Freeze <br> Audit history + undo <br> Admin/content instance switching |
| Open Beta | Miscellaneous | Free | **Started** | **NoorChatExtras** | NoorCore | 11 chat channels + PMs <br> Custom chat tags <br> Emoji glyphs <br> Interactive placeholders <br> Spy, slowmode, helpop, ignores |
| Open Beta | Miscellaneous | Free | **Started** | **NoorFriends** | NoorCore | Friends & requests <br> Parties <br> `/frtp` |
| Open Beta | Miscellaneous | Free | **Started** | **NoorFamilies** | NoorCore | Child accounts <br> Guardian controls <br> Chat auditing & dated logs <br> Teleport policy hook |
| Unknown | Miscellaneous | Free | **Started** | **NoorGuide** | NoorCore | Reviewed local knowledge base <br> `/question` search <br> Draft → review → publish workflow |
| Unknown | Miscellaneous | Free | **Started** | **NoorAdvancements** | NoorCore | Embedded datapack replacing the vanilla advancement menu |
| Unknown | Miscellaneous | Free | **Started** | **NoorGames** | NoorCore | Wordle in a double chest (`/wordle`) — first of a games set |
| Unknown | Miscellaneous | Free | **Started** | **NoorRadio** | — | Resource-pack music on demand (`/radio`) <br> ⚠️ Targets Paper 1.20+ and references **Oraxen**, not NoorPack |
| Unknown | Miscellaneous | Free | **Started** | **NoorMapArt** | — | Map art tooling (no README yet) |
| — | Miscellaneous | Free | **Absorbed** | ~~NoorDrops~~ | — | Runtime now lives inside NoorGeodes; `/noordrops`, `plugins/NoorDrops/` and `noordrops:*` keys preserved |
| — | Social | Free | **Not Started** | ~~NoorMarriage~~ | — | ⚠️ No repo, **but `MarriageServiceBridge` already ships in NoorCore** |

---

# Monetization (Through content)

> This section is made purely for the sake of building up not only a player community, but also a contributor community. I'll lay out my ideas below
> 
> -Milo

## Plugin Pricing
This was outlined above, dumbass [Go read it](#structure)

---

## Plugin Dependency Bundles

Plugins that extend or inherit from another paid plugin cannot be purchased standalone without the parent. When a user attempts to buy a dependent plugin without owning the parent, the storefront forces a bundle purchase at a **5–10% discount** off the combined price.

This prevents users from purchasing extensions they can't use, and rewards full ecosystem adoption.

**Hard dependencies that force a bundle today** (from `plugin.yml` `depend:`):

| Dependent | Requires |
|-----------|----------|
| *Everything* | NoorCore (free — never gates a bundle) |
| NoorGeodes | NoorItems |
| NoorFish | NoorItems |
| NoorEconomy | Vault (third-party) |
| NoorQuests | Vault (third-party) |

> ⚠️ Only **NoorGeodes → NoorItems** and **NoorFish → NoorItems** are true paid-parent bundles.
> Since NoorItems is currently marked **Free**, neither actually triggers the bundle rule.
> Either NoorItems needs a price or the rule needs a different anchor.

---

## Community Config Pack Marketplace

Most plugins load content through YAML config files (items, entities, crates, etc.). The marketplace hosts a community section where creators can upload and sell config packs for these plugins.

**This is a strong fit for the current architecture** — NoorItems items/recipes/kits/sets,
NoorCrates crates/keys/pools, NoorGeodes tiers, NoorQuests pools, NoorNPCs guide definitions
and NoorJobs job files are all already per-file YAML that drops into a folder. The in-game
`/itemeditor` and `/recipeeditor` also lower the barrier to authoring a pack.

### Platform Cut (Standalone)
- Creators set their own price
- We take a **20% cut** per sale
- Example: a \$5 pack → creator earns \$4.00, we earn $1.00
---

## Curated Bundles

When a community pack meets a quality bar, it may be selected for an official **Curated Bundle** - a themed collection of packs that serve a cohesive server style (e.g. a Fantasy Bundle containing a weapons pack, a creatures pack, a structures pack, etc.).

### Pricing
- Bundle price = sum of each included pack's standalone price, with **$0.50 added per pack that uses a free plugin**
- A **5% discount** is applied to the total
- We take a **5% cut** of each bundle sale
### Creator Revenue (Curated)
- Remaining 95% is split proportionally based on each pack's standalone price
- Creators earn **more per sale in a bundle than standalone**
- Free-plugin pack creators who were earning nothing now receive a cut
### Example
| Pack                       | Standalone Price       | Standalone Earnings (80%) | Bundle Earnings (proportional share of 95%) |
|----------------------------|------------------------|---------------------------|---------------------------------------------|
| User1 – Fantasy Weapons    | $4.00                  | $3.20                     | $3.608                                      |
| User2 – Fantasy Creatures  | $3.00                  | $2.40                     | $2.706                                      |
| User3 – Fantasy Fishing    | $2.50                  | $2.00                     | $2.255                                      |
| User4 – Fantasy Structures | $0.00                  | $0.00                     | $0.451                                      |
| **Bundle Total**           | \$10.00x0.95=**$9.50** | —                         | **\$9.02 to creators <br> $0.48 to us**     |

### Curation Standards
Curated status is selective and must remain meaningful. Criteria include content quality, config documentation, balance, and update history. Packs are not curated indiscriminately — the label is what drives conversions.

If a curated pack becomes unmaintained after a plugin update, it is either pulled from the bundle, replaced, or maintained internally, since the platform's name is attached to it.
 
---

## Competition Bundles

On a regular cadence (roughly quarterly), we run **themed community competitions**. These are primarily a community-building initiative, not a revenue driver.

### How It Works
1. A specific theme is announced (e.g. *Undead Siege*, *Merchant Guild*)
2. Creators submit packs privately — no public submissions to avoid popularity contests
3. We select **5–10 packs** across three internal categories
4. Selected packs are bundled at a **static price**, split **evenly** among all selected creators
5. **We take no cut** — the effort to review and test submissions is low, and the goal is community building
### Selection Categories

| Category                 | Slots     | Criteria                                                                                                            |
|--------------------------|-----------|---------------------------------------------------------------------------------------------------------------------|
| **Theme Interpretation** | 2–4 packs | Creatively interpreted the theme, even if execution isn't perfectly polished                                        |
| **Newcomer Spotlight**   | 1–2 packs | From newer creators; if content is promising but rough, we collaborate with the creator to polish it before release |
| **Quality & Polish**     | 2–4 packs | Exceptionally well-executed content that raises the bundle's overall ceiling                                        |

These categories are an internal selection framework only — from the creator's perspective, they are simply selected. Category overlap is handled with internal flexibility.

### Design Principles
- Themes are specific enough to produce cohesive results, broad enough for creative interpretation
- The newcomer slots and collaborative polish process actively develop the creator pipeline, not just reward those already experienced
- Even revenue splits and no platform cut keep the format perceived as genuine community celebration
- Quarterly cadence keeps momentum without burning out the submission pool

---

# Open Questions & Design Drift

Things where the design and the code genuinely disagree, and someone has to decide.

### Needs a design decision

1. **"Mastery Paths" perk is now meaningless.** Ranks and store ranks both sell "+N Mastery
   Paths". All 14 jobs are available to everyone and mastery no longer exists. Should this
   become promotion points, promotion track slots, or be removed and repriced?
2. **Name collisions.** `Prospector`, `Forester` and `Artisan` are each both a *rank* and a
   *promotion track*. Rename one side.
3. **Crate tier naming.** Store sells Small/Medium/Large/Monthly Cosmetic; config ships
   `starter`, `premium`, `bonfire-key`. Pick canonical IDs.
4. **Town upgrade axes.** Design prices Claims/Outposts/Vaults; code implements
   Claims/**NPC Slots**/Vaults, with Outposts as a free-standing feature. Reconcile before pricing.
5. **Bundle rule has no teeth.** The only real paid-parent dependencies are
   NoorGeodes→NoorItems and NoorFish→NoorItems, and NoorItems is marked Free.
6. **Smelter ladder drifted.** Design says Mega-Coal at 50 and Titanium+Uranium at 100; YAML
   says Mini-Coal at 10 and Uranium only, with no level-35 perk.

### Needs a code decision

7. **`MarriageServiceBridge` is orphaned.** 23 methods and DTOs ship in NoorCore, documented in
   its README, with no NoorMarriage plugin to implement them. Build it, or remove the bridge.
8. **NoorUtils is a stub.** It is listed as a dependency-worthy backend module but only has a
   ping command, while `UtilServiceBridge` exists in Core.
9. **NoorRadio is off-baseline.** Paper 1.20+, Maven, and it documents **Oraxen** for sounds
   while the rest of the suite standardises on NoorPack. Align or retire it.
10. **`noorjobs.brewer.luck` granted twice** — at Brewer level 25 and again at 50.
11. **Wizard and Chef have no perks.** Every milestone for both jobs reads `Coming Soon!`.
12. **Rank IV promotion perks are display-only.** The custom item, recipe, combat, drop and
    metadata behavior implied by the track names is unimplemented.
13. **No prestige implementation** despite Imperator selling it.
14. **Perfect Items unimplemented** despite being a documented Geode outcome.
15. **Blueprint Essence/pity loop unimplemented.** Scrap and blueprint items exist; the
    exchange, Essence currency and conjuring do not.
16. **No NoorData.** Persistence is per-plugin: SQLite (NoorEconomy), MySQL (NoorNPCs), YAML
    (most others). Decide whether to centralise or formally drop the plan.
