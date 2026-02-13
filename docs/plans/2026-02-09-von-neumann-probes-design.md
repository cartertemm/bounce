# Von Neumann Probes & Per-Planet Economy Redesign

## Overview

Transform the planet system from a cosmetic layer into a full logistics simulation. Planets have local resources, Von Neumann probes autonomously discover new worlds, shipping probes move rubber between planets, and construction probes auto-build infrastructure.

## 1. Per-Planet Resource Model

### Current State
- All resources (crude, monomers, polymer, compound, finishedRubber, rubber) are global on the `game` object
- All buildings across all planets contribute to one shared pool
- Planet specialization (rubber/ball/hybrid roles) is cosmetic

### New State
- Each planet object holds its own supply chain stockpiles:
  ```
  planet.crude, planet.monomers, planet.polymer,
  planet.compound, planet.finishedRubber, planet.rubber
  ```
- Each planet's rubber buildings produce into and consume from that planet's local pools
- Ball factories consume rubber from the planet they're on
- **Balls remain global** - when any planet produces balls, they go into `game.balls` / `game.totalBalls`

### Game Loop Changes
- Instead of one global supply chain pass, iterate over each planet and run its supply chain independently
- Ball production per planet: consume local rubber, add to global balls
- Remove `game.crude`, `game.monomers`, `game.polymer`, `game.compound`, `game.finishedRubber` from global state
- Keep `game.rubber` only as a display convenience (sum of all planet rubber) or remove entirely

### Migration
- On load from old save: move `game.crude`, `game.monomers`, etc. onto Earth's planet object
- New planets start with all intermediates at 0

## 2. Planet Specialization & Balance

### Role Bonuses
- **Rubber World:** 1.5x production multiplier on all rubber chain buildings
- **Ball World:** 1.5x production multiplier on all ball factories
- **Hybrid:** 1x (no bonus)

### Hybrid Building Cap
- Hybrid planets can build at most **25 of each building type**
- Specialized planets have **no building cap**
- Construction probes respect the cap (stop building when reached)

### Balance Rationale
- Early game: Earth (hybrid) is self-contained, 25-cap is plenty, no shipping needed
- Mid game: Specialization + shipping becomes clearly better than more hybrids
- Late game: Specialized worlds with 100+ buildings at 1.5x completely eclipse hybrids
- The 1.15x cost scaling amplifies specialization advantage exponentially

### Earth
- Starts as hybrid, can be changed to specialized like any other planet
- No special treatment beyond being the starting planet

## 3. Von Neumann Probe Discovery System

### Unlocking
- The probe launch UI appears once the player can reasonably afford it (e.g., 500K+ balls)
- Launching the first probe costs balls (e.g., 500,000)
- **You only ever need to launch one probe** - that's the point of Von Neumann

### Travel Phase
- After launch, the probe enters a travel phase (displayed as a progress bar or countdown)
- First destination: ~60 seconds travel time
- During travel, nothing happens - the probe is in transit

### Arrival & Self-Replication
- When the probe arrives at a system, it begins self-replicating
- First replication cycle: ~60 seconds
- Each subsequent cycle takes 1.5x longer than the previous
- Each replication cycle **discovers** one new planet

### Discovery vs Colonization
- **Discovered planets** appear in the planets list with a "Discovered" tag, grayed out
- The player can see discovered planets but cannot build on them or assign roles
- To **colonize** a discovered planet, the player must meet a totalBalls milestone

### Colonization Milestones
| Planet # | Total Balls Required |
|----------|---------------------|
| 1st      | 1,000,000 (1M)      |
| 2nd      | 5,000,000 (5M)      |
| 3rd      | 25,000,000 (25M)    |
| 4th      | 125,000,000 (125M)  |
| 5th      | 625,000,000 (625M)  |
| nth      | 1M * 5^(n-1)        |

- Milestones are **gates, not costs** - reaching the milestone doesn't spend balls
- When a milestone is met, a "Colonize" button appears on any discovered planet
- Colonizing lets the player choose a role (rubber/ball/hybrid)
- The planet then becomes active with 0 buildings and 0 resources

### Replication Chain
- Probes on colonized planets eventually build child probes that travel further
- This is automatic and invisible to the player (just discoveries appearing)
- Discovery rate accelerates slightly as more planets have replicating probes
- Practically: discoveries come in bursts at first, then steady out

### Game State
```
game.probes = {
    launched: false,
    travelProgress: 0,      // 0 to 1
    travelDuration: 60,      // seconds
    arrived: false,
    replicationTimer: 0,
    replicationCycle: 1,
    discoveredPlanets: []     // planets found but not yet colonized
}
```

## 4. Shipping Probes

### Purpose
Move rubber from rubber-producing planets to ball-producing planets.

### Network Model (not explicit routes)
- Shipping probes form a global fleet
- All planets are automatically part of the distribution network
- Rubber flows from planets with **surplus** (production > consumption) to planets with **deficit** (consumption > production)
- Distribution is proportional to demand

### Mechanics
- **Building shipping probes:** costs balls, scales like buildings (baseCost * 1.15^count)
- **Capacity:** each shipping probe moves X rubber/sec (e.g., 5 rubber/sec per probe)
- **Total fleet capacity** = count * rate per probe

### Flow Algorithm (per tick)
1. Calculate each planet's rubber surplus/deficit rate
   - Surplus = rubber production rate - rubber consumption rate (if positive)
   - Deficit = rubber consumption rate - rubber production rate (if positive)
2. Total available = min(total surplus across all planets, fleet capacity)
3. Total needed = total deficit across all planets
4. Actual shipped = min(total available, total needed)
5. Distribute shipped rubber to deficit planets proportional to their demand
6. Deduct shipped rubber from surplus planets proportional to their supply

### Display
- New "Logistics" panel showing:
  - Shipping fleet size and total capacity
  - Rubber flow summary: total exported, total imported
  - Per-planet: surplus/deficit indicator
- Planet supply chain display shows "Shipped in: +X/s" or "Shipped out: -X/s"

### Game State
```
game.shippingProbes = {
    count: 0,
    baseCost: 1000,
    costMult: 1.15,
    ratePerProbe: 5  // rubber/sec per probe
}
```

## 5. Construction Probes

### Purpose
Automatically build infrastructure on colonized planets based on their role.

### Mechanics
- **Building construction probes:** costs balls, scales like buildings
- Each probe works autonomously, one building at a time
- **Build time:** ~30 seconds per building
- **Building costs:** paid from global ball pool at normal price (baseCost * 1.15^count)
- If the player can't afford the next building, the probe waits

### Auto-Optimize Logic

**Rubber World:**
- Builds the supply chain balanced - always builds the stage with the fewest buildings
- If all stages equal, builds from the top (extractor first)
- Goal: keep all 6 stages level for maximum throughput

**Ball World:**
- Builds ball factories, prioritizing the cheapest available type
- Starts with basic factories, moves up tiers as counts grow

**Hybrid:**
- Alternates between rubber chain (balanced) and ball factories
- Respects the 25-building cap per type

### Assignment
- Construction probes auto-assign to the planet with the fewest total buildings
- When a planet hits the hybrid cap or is "balanced enough," the probe moves to the next neediest planet
- No manual assignment needed

### Game State
```
game.constructionProbes = {
    count: 0,
    baseCost: 5000,
    costMult: 1.15,
    buildTime: 30,           // seconds per building
    assignments: []           // tracked internally: which probe is building where
}
```

## 6. UI Changes

### Stats Panel
- **Balls:** global (unchanged)
- **BPS:** global actual rate (unchanged)
- **Rubber:** show "Total across all planets" or selected planet's rubber
- **RPS:** global actual throughput (unchanged)
- **Planets:** "X colonized / Y discovered"

### Supply Chain Display
- Shows the **selected planet's** local stockpiles and supply/demand rates
- If the planet receives shipped rubber, show "+X/s shipped in" on the rubber step
- If the planet exports rubber, show "-X/s shipped out"

### Planet Selector
- Works as currently implemented
- Only shows colonized (active) planets
- Discovered-but-not-colonized planets shown in the planets panel, not the selector

### Planets Panel
- **Colonized planets:** show role, building counts, role-change buttons (as now)
- **Discovered planets:** show name, "Discovered" tag, "Colonize" button (if milestone met)
- **Probe status:** show travel progress, replication status, discovery count

### New Logistics Panel
- Shipping fleet: count, total capacity (X rubber/sec)
- Buy shipping probe button
- Buy construction probe button
- Construction activity: "Building [building name] on [planet name]... X seconds remaining"
- Network flow summary

## 7. Achievement Updates

New achievements to add:
- "First Contact" - Launch your first Von Neumann probe
- "Self-Replicating" - First probe replication / first discovery
- "Logistics Network" - Build your first shipping probe
- "Automated Empire" - Build your first construction probe
- "Five Worlds" - Colonize 5 planets
- "Galactic Industrialist" - Have 10+ specialized planets producing simultaneously

## 8. Save/Load Changes

### New fields to save
- `game.probes` (discovery state)
- `game.shippingProbes` (fleet state)
- `game.constructionProbes` (builder state)
- Per-planet: `crude, monomers, polymer, compound, finishedRubber, rubber`
- `game.colonizationCount` (how many planets have been colonized, for milestone tracking)

### Migration from old saves
- Move `game.crude`, `game.monomers`, etc. to `game.planets[0]` (Earth)
- Initialize new systems to defaults (no probes launched, 0 shipping/construction probes)
- Existing planets keep their buildings, assigned Earth's former resource values

## 9. Implementation Order

Recommended sequence to minimize breakage:

1. **Per-planet resources** - Move intermediates to planet objects, update game loop, update display. Everything still works but is now per-planet (with only Earth, behavior is identical).

2. **Specialization balance** - Add 1.5x bonus for specialized worlds, 25-cap for hybrids. Update building logic and display.

3. **Milestone system** - Replace flat probe cost with milestone-gated colonization. Remove old probe launch, add milestone tracking.

4. **Von Neumann discovery** - Add probe launch, travel timer, self-replication, discovery pipeline. Discovered planets appear in UI.

5. **Shipping probes** - Add shipping fleet, flow algorithm, logistics display. This makes multi-planet ball production actually work.

6. **Construction probes** - Add auto-building, assignment logic, build timer display. This is the "automation endgame."

7. **Achievements & polish** - New achievements, UI refinements, balance tuning.

Each step is independently testable and the game remains playable after each one.

## 10. Cosmic Endgame & Prestige

### Cosmic Tiers

As balls accumulate in space, they gravitationally coalesce into increasingly massive cosmic bodies. Each tier is a threshold on **total balls produced** (cumulative across the current run):

| Tier | Cosmic Body | Total Balls Threshold |
|------|------------|----------------------|
| 1 | Moons | 1 Billion (1e9) |
| 2 | Planets | 100 Billion (1e11) |
| 3 | Stars | 10 Trillion (1e13) |
| 4 | Giant Stars | 1 Quadrillion (1e15) |
| 5 | Black Holes | 100 Quadrillion (1e17) |
| 6 | Singularity | 10 Quintillion (1e19) |

- Each tier unlocks visually in the UI
- Each tier provides a passive **2x global production multiplier** (stacking: tier 3 = 8x total)
- When Singularity is reached, a **"Collapse Universe"** button appears

### Singularity Reset (Prestige)

Clicking "Collapse Universe" triggers a prestige reset:

**Resets:**
- All balls, rubber, intermediates (per-planet stockpiles)
- All buildings on all planets (rubber buildings, ball factories)
- All planets except Earth (back to 1 hybrid planet)
- Von Neumann probe state (must re-launch)
- Shipping and construction probe counts
- Cosmic tier progress

**Persists:**
- Upgrades (already purchased)
- Achievements
- Singularity Points earned
- Prestige upgrade tree purchases

### Singularity Points

- **Base:** 1 SP per singularity
- **Speed bonus:** extra SP if you reach singularity faster than your previous best time
- Spent in a **Prestige Upgrade Tree** with branches:
  - **Production:** +X% base production per level
  - **Efficiency:** reduce building cost scaling (1.15 → 1.14 → 1.13...)
  - **Logistics:** shipping/construction probes start with bonus capacity
  - **Discovery:** Von Neumann probes travel/replicate faster
  - **Cosmic:** reduce cosmic tier thresholds by X% per level

### Colossal Singularity (Win Condition)

After multiple singularity resets, the player accumulates a **Singularity Counter**. When enough singularities have formed, they attract each other into the **Colossal Singularity**.

- Displayed as a progress bar filling with each singularity reset
- **Number of singularities required:** configurable constant (`SINGULARITIES_TO_WIN`), player decides
- When the Colossal Singularity completes:
  - Time stands still - all production freezes
  - Victory screen with stats: total time played, total balls ever produced, singularities completed, fastest run
  - Option to continue in **sandbox mode** (keep playing with all prestige bonuses, no more win triggers)

### Game State
```
game.cosmic = {
    currentTier: 0,           // 0-6
    tierThresholds: [1e9, 1e11, 1e13, 1e15, 1e17, 1e19]
}

game.prestige = {
    singularityCount: 0,
    singularityPoints: 0,
    bestTime: Infinity,       // fastest singularity run in seconds
    totalBallsAllTime: 0,     // cumulative across all runs
    upgrades: {
        production: 0,
        efficiency: 0,
        logistics: 0,
        discovery: 0,
        cosmic: 0
    }
}

const SINGULARITIES_TO_WIN = 10;  // player configurable
```

## 11. Progressive Unlock System

### Core Principle
The player starts seeing only a bouncing ball and a click button. Everything else reveals itself through milestones. Each unlock feels like a discovery, not a menu dump. The player never sees a wall of locked items - just the next exciting thing appearing when they're ready.

### Unlock Chain

| Balls Produced | What Unlocks |
|---|---|
| 0 | Just the ball + click button |
| 50 | First upgrade (click power) |
| 200 | Rubber Extractor (1st chain step) |
| 500 | Upgrades panel expands (2-3 more) |
| 1,000 | Cracker (2nd chain step) |
| 2,500 | Reactor (3rd step) + new upgrades |
| 5,000 | Mixer (4th step) |
| 10,000 | Press (5th step) + more upgrades |
| 25,000 | Forming Plant (6th step) - rubber complete |
| 50,000 | Basic Ball Factory |
| 100K+ | Higher-tier factories, production upgrades |
| 500K | Von Neumann probe launch |
| 1M | First planet colonization milestone |
| 10M+ | Shipping/construction probes |
| 1B | Cosmic tier 1 (Moons) |

Each unlock shows a brief notification/toast message.

### Visibility Rules
- Upgrades appear only when the player is near affording them or hits the relevant milestone
- UI panels (supply chain, planets, logistics, cosmic) appear only when their first element unlocks
- Discovered-but-not-yet-affordable items can show as "???" teasers at most

## 12. Upgrades (101 Total)

### Early Game (0 - 50K balls) — 32 upgrades

**Click Power (5):**
| # | Name | Effect | Unlock |
|---|------|--------|--------|
| 1 | Stronger Fingers | +1 per click | 50 balls |
| 2 | Rubber Gloves | +2 per click | 200 balls |
| 3 | Spring-Loaded Palm | +5 per click | 1,000 balls |
| 4 | Pneumatic Press | +10 per click | 5,000 balls |
| 5 | Quantum Tap | +25 per click | 25,000 balls |

**Rubber Chain Efficiency (18 — 6 stages x 3 tiers):**
Each supply chain stage gets 3 tiers that stack:
- Tier 1: +50% output speed (unlocks at 3 buildings of that stage)
- Tier 2: +100% output speed (unlocks at 8 buildings)
- Tier 3: +200% output speed (unlocks at 15 buildings)
- Naming pattern: "Improved [Stage]," "Advanced [Stage]," "Quantum [Stage]"

**Cost Reduction (6 — one per supply chain stage):**
Reduces building cost scaling from 1.15x to 1.12x for that stage. Unlocks at 10 buildings of that stage.

**Auto-Collect (3):**
| # | Name | Effect | Unlock |
|---|------|--------|--------|
| 1 | Auto-Bounce | +1 ball/sec passive | 500 balls |
| 2 | Auto-Bounce II | +5/sec | 5,000 balls |
| 3 | Auto-Bounce III | +25/sec | 25,000 balls |

### Mid Game (50K - 10M balls) — 28 upgrades

**Ball Factory Efficiency (12 — 4 factory types x 3 tiers):**
Each ball factory type gets 3 efficiency tiers:
- +50%, +100%, +200% output
- Unlock as you build more of that factory type (5, 15, 30 built)

**Bulk Processing (6 — one per supply chain stage):**
Each building processes double the batch size. Unlocks at 10+ of that building.

**Synergy Upgrades (6):**
| # | Name | Requirement | Bonus |
|---|------|-------------|-------|
| 1 | Streamlined Pipeline | All 6 stages at 5+ | +25% all rubber |
| 2 | Industrial Complex | All at 10+ | +50% |
| 3 | Manufacturing Empire | All at 20+ | +100% |
| 4 | Rubber Baron | All at 35+ | +200% |
| 5 | Polymer Singularity | All at 50+ | +400% |
| 6 | Molecular Mastery | All at 75+ | +800% |

**Planet Prep (4) — build anticipation before planets reveal:**
| # | Name | Unlock | Effect |
|---|------|--------|--------|
| 1 | Long Range Sensors | 100K balls | "Detect something... out there" |
| 2 | Signal Processing | 200K balls | "The signals are getting clearer" |
| 3 | Orbital Telescope | 350K balls | Reveals planet count in stats |
| 4 | Probe Blueprints | 500K balls | Unlocks Von Neumann probe launch |

### Late Game (10M - 1B balls) — 20 upgrades

**Shipping Fleet (5):**
| # | Name | Effect |
|---|------|--------|
| 1 | Cargo Optimization | +50% rubber per shipping probe |
| 2 | Hyperspace Lanes | +100% shipping capacity |
| 3 | Quantum Entanglement Shipping | Rubber arrives instantly |
| 4 | Fleet Coordination | Shipping probes cost 25% less |
| 5 | Galactic Trade Network | Surplus rubber auto-sells for balls at 10:1 |

**Construction Automation (5):**
| # | Name | Effect |
|---|------|--------|
| 1 | Rapid Assembly | Build time -25% |
| 2 | Prefabrication | Build time -50% (stacks) |
| 3 | Nanoscale Construction | Build time -75% (stacks) |
| 4 | Blueprint Sharing | Probes build 2 buildings simultaneously |
| 5 | Self-Improving Builders | Probes get faster the more they build |

**Planet Mastery (6):**
| # | Name | Effect |
|---|------|--------|
| 1 | Specialized Workforce | Specialized bonus 1.5x → 1.75x |
| 2 | Expert Workforce | Bonus → 2x |
| 3 | Planetary Governor | Hybrid cap 25 → 35 |
| 4 | Regional Planning | Hybrid cap → 50 |
| 5 | Terraforming I | New planets start with 5 pre-built buildings |
| 6 | Terraforming II | New planets start with 15 pre-built |

**Discovery Acceleration (4):**
| # | Name | Effect |
|---|------|--------|
| 1 | Faster-Than-Light Probes | Travel time -50% |
| 2 | Parallel Replication | Two planets discovered per cycle |
| 3 | Replication Efficiency | Cycle scaling 1.5x → 1.3x |
| 4 | Deep Space Network | Discoveries happen offline |

### Cosmic & Prestige (1B+ / post-singularity) — 21 upgrades

**Cosmic Tier Bonuses (6 — one per tier reached):**
| # | Name | Tier | Effect |
|---|------|------|--------|
| 1 | Lunar Gravity | Moons | Click value x10 |
| 2 | Planetary Mass | Planets | All building output x2 |
| 3 | Stellar Fusion | Stars | Chain stages feed 25% faster |
| 4 | Giant Star Pressure | Giant Stars | Ball factories use 25% less rubber |
| 5 | Event Horizon | Black Holes | All costs reduced 15% |
| 6 | Singularity Proximity | Singularity | Everything x5 |

**Prestige Tree (15 — bought with Singularity Points):**

*Production Branch (5):*
- Residual Momentum I-V: +20%/+50%/+100%/+250%/+500% base production

*Efficiency Branch (3):*
- Remembered Blueprints I-III: cost scaling 1.15→1.13→1.11→1.09

*Logistics Branch (3):*
- Probe Memory I-III: start each run with 1/3/10 free shipping probes

*Discovery Branch (2):*
- Déjà Vu I-II: Von Neumann travel time halved, then halved again

*Cosmic Branch (2):*
- Compressed Spacetime I-II: cosmic tier thresholds -25%, then -50%

## 13. Updated Implementation Order

Revised to include all systems:

1. **Progressive unlock system** - Visibility engine: panels/elements hidden until milestone met. Start with just click + ball.
2. **Upgrade system (101 upgrades)** - Data-driven upgrade definitions, unlock conditions, purchase logic, effect application.
3. **Per-planet resources** - Move intermediates to planet objects, update game loop and display.
4. **Specialization balance** - 1.5x bonus for specialized, 25-cap for hybrids.
5. **Milestone system** - Milestone-gated colonization.
6. **Von Neumann discovery** - Probe launch, travel, self-replication, discovery pipeline.
7. **Shipping probes** - Fleet, flow algorithm, logistics display.
8. **Construction probes** - Auto-building, assignment logic.
9. **Cosmic tiers** - Tier thresholds, production multipliers, visual indicators.
10. **Prestige system** - Singularity reset, SP earning, prestige upgrade tree.
11. **Colossal Singularity** - Win condition, victory screen, sandbox mode.
12. **Achievements & polish** - New achievements, UI refinements, balance tuning.
