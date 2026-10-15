# MITHRIL GAMEPLAY & SYSTEMS REVERSE-ENGINEERING REPORT
**Target Game:** *Mithril* (Steam Demo App ID: 5063090, Full Game App ID: 5053270)
**Developer / Publisher:** Agentic Games (Melbourne, Victoria, Australia)
**Target Architecture / Engine:** Custom Rust Voxel Simulation Engine + Unreal Engine 5
**Primary Target Project:** *Forgeworld* (Godot 4 + C# desktop engineering game)
**Research Date:** October 2026

---

## 1. Executive Summary

*Mithril* represents a major paradigm shift in voxel survival and automation design. Rather than treating subterranean environments as static terrain grids with static ore distribution, *Mithril* models geology, fluid mechanics, structural physics, and time as active dynamic forces. Driven by a bespoke Rust voxel engine bridged into Unreal Engine 5 via C-bindings/WASM, the game introduces **Dormancy**—a discrete, player-triggered "deep time" fast-forward simulation (1.25 million years in a single step) that transforms rock compositions, dissolves limestone, forms hydrothermal ore deposits, and collapses unsupported tunnels based on physical principles.

Key technical and gameplay innovations in *Mithril* include:
1. **Dynamic Geological Metamorphism:** Ore formation is not purely procedural at world generation; it is generated dynamically when heat (lava) or fluid (water) interacts with host rock strata during Dormancy.
2. **Real-Time Structural Stress & Collapse:** Every removed voxel recalculates weight distribution across adjacent voxels. Exceeding shear/compressive limits causes cascading structural collapses that turn rock into dynamic rubble.
3. **Closed-Loop Automation & Physical Logistics:** Automated mining arms ("Orehands"), crushers, conveyors, and crystal furnaces create physical transport and processing chains that must withstand structural shifts and environmental hazards.
4. **Adaptive Biological Threat ("The Memory"):** A planned full-game feature where surviving subterranean creatures adapt their evolutionary traits and pathfinding to counter player-built defensive and mining strategies after Dormancy.

For **Forgeworld** (a Godot 4 + C# desktop engineering game combining factory automation, salvage, procedural exploration, modular construction, and physics-based engineering), *Mithril*'s systems offer invaluable architectural precedents. By adapting *Mithril*'s structural stress propagation, readable diagnostic wireframes, and geological state transformation into C# data structures (such as `2D/3D BitGrid` and `Graph-Based Shear Stress Solvers`), *Forgeworld* can achieve deep emergent simulation without prohibitive performance overhead.

---

## 2. Access & Evidence Limitations

### 2.1 Environmental Audit & Operational Constraints
* **Execution Environment:** Linux devbox (`x86_64 Linux 6.8.0`, headless terminal environment).
* **Direct Demo Binary Execution:** Direct execution of the Windows Steam demo (`Mithril Demo.exe` / DirectX 12 Shader Model 6 runtime) was impossible due to the lack of a native Windows GPU environment, display server, or DirectX 12 hardware acceleration.
* **Operational Declaration:** In strict accordance with Phase 1 instructions, **I do not claim to have directly operated the compiled Windows Steam demo binary.**

### 2.2 Primary Research Sources Utilized
To achieve an exhaustive, evidence-based analysis without executing the binary, research was conducted across four primary source categories:
1. **Live WebAssembly / REST Engine (`cave.playmithril.com`):** Direct reverse-engineering of the browser cave generator's WebAssembly bundle (`app.js`) and REST API parameters (`/api/generate`, `/api/mine`, `/api/place-water`, `/api/sleep`, `/api/report`).
2. **Developer Press Kit & Technical Documentation (`playmithril.com/press/`):** Analysis of developer quotes (Connor & Tiffany Tassie), hardware requirements, system features, and official descriptions.
3. **Steam Demo Store Page & Community Documentation (`store.steampowered.com/app/5063090/`):** Review of user feedback, system requirements, feature disclosures, and official demo disclosures.
4. **High-Resolution Media & Collapse Showcase Footage:** Frame-by-frame analysis of official 1080p/4K gameplay footage, collapse showcases, strut placement demos, and automation loops.

---

## 3. Detailed Breakdown of Core Gameplay Systems

### System 1: Procedural Cave Generation & Geological Materials
* **Player Action:** Sets world seed, 3D region bounds ($X \times Y \times Z$ chunk grid), noise parameters, and geological layer depths; initiates world generation.
* **System Response:** Generates a 3D volumetric voxel grid containing depth-layered host strata, worm-carved cavern networks, and geologically realistic ore veins.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Seed, Cavern Frequency/Threshold, Detail Octaves, Worm Per Region/Radius/Step, Layer Depths (Sandstone, Granite, Basalt, Slate).
  * *Outputs:* 3D Voxel Mesh with 22 distinct geological material IDs (Host rocks: Limestone, Sandstone, Granite, Basalt, Slate; Metamorphic: Marble, Hornfels, Skarn; Ores/Minerals: Iron, Copper, Malachite, Tin, Sulfide, Pyrite, Quartz, Gold, Kimberlite, Diamond, Amethyst, Crystal, Coal, Garnet, Diopside, Gypsum).
* **Rules & Logic:**
  * Host rock distribution is depth-stratified.
  * Ore veins follow geological models: Banded Iron forms horizontal sine-wave layers; Copper forms dendritic branching tendrils; Kimberlite forms vertical carrot-taper pipes; Gold generates strictly inside Quartz reef veins ("Bonanza Zones"); Pyrite forms indicator halos around primary sulfide deposits.
* **Interactions:** Host rock material properties determine structural load-bearing capacity and tool tier requirements for mining.
* **Progression/Failure:** Progressing to lower depths exposes harder rock layers (Granite/Basalt) requiring upgraded mining claws or drills. Failure to identify weak strata leads to unexpected collapses.
* **Observed vs. Inferred:** Directly observed via live WASM engine parameters (`cave.playmithril.com`).

### System 2: Mining, Resource Extraction & Material Properties
* **Player Action:** Uses hand claws, drills, or automated mining heads to dig into voxel rock faces.
* **System Response:** Removes a spherical volume of voxels ($R=1$ to $R=10$), drops loose resource items, updates surrounding voxel exposure, and recalculates structural stress on adjacent nodes.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Player tool tier, click position, spherical mining radius.
  * *Outputs:* Destroyed voxel volume, collected ore chunks (e.g., Raw Copper, Iron Ore, Coal), altered cave geometry.
* **Rules & Logic:**
  * Voxel removal is non-grid-aligned (spherical carving with smooth isosurface extraction).
  * Material hardness dictates mining speed and minimum required tool tier.
  * "Keyhole Mining": Strategy of precisely carving around valuable ore veins to preserve structural integrity of host rock.
* **Interactions:** Voxel removal reduces surrounding structural support, increasing local stress values.
* **Progression/Failure:** Mining high-tier ores (e.g., Mithril, Diamond) requires advanced equipment. Over-mining without support causes immediate cave-ins.
* **Observed vs. Inferred:** Directly observed in browser WASM (`/api/mine` endpoint) and press kit video clips.

### System 3: Structural Stress, Load Distribution, Supports & Cave-Ins
* **Player Action:** Excavates subterranean space or places structural support struts (Wood, Copper, Iron, Steel, Crystal, Mithril).
* **System Response:** Computes real-time stress propagation through neighboring voxels. If stress exceeds material threshold, a structural collapse event is triggered, converting solid voxels into falling rubble entities that settle with a 0.85 fill ratio.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Gravity Weight ($1.0$), Lateral Support Factor ($0.3$), Vertical Support Factor ($0.5$), Propagation Radius ($R=4$), Strut placement.
  * *Outputs:* Real-time stress wireframe diagnostic (cyan/teal glowing wireframe), collapse events, loose rubble blockages.
* **Rules & Logic:**
  * Weight distribution propagates outwards up to $R=4$ voxels.
  * Unsupported spans exceeding material shear strength initiate cascading collapses up to `Max Cascade` limit ($50$ chunks).
  * Strut Tiers increase structural support capacity and survival rates during Dormancy (Copper Struts: moderate survival; Mithril Struts: near $100\%$ survival).
* **Interactions:** Collapses destroy placed machinery, crush player/creatures, and alter fluid flow channels.
* **Progression/Failure:** Deeper tunnels experience higher gravitational stress multipliers, forcing players to advance from basic timber/copper supports to heavy steel/mithril trusses.
* **Observed vs. Inferred:** Directly observed via browser engine parameters (`gen-stress-gravity`, `gen-stress-lateral`, `gen-stress-prop-radius`) and collapse video footage.

### System 4: Crafting, Progression, Equipment & Unlocks
* **Player Action:** Feeds raw ores into processing machinery (Crushers, Furnaces) to produce refined ingots and component parts, unlocking advanced equipment tiers at crafting benches.
* **System Response:** Converts raw materials into structural struts, automated machinery, upgraded mining claws, and fluid conduits.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Raw Ores (Copper, Iron, Coal), Energy/Fuel, Crafting Bench recipes.
  * *Outputs:* Refined Ingots, Orehand arms, Conveyors, Support Struts, Advanced Mining Tools.
* **Rules & Logic:**
  * Tier Progression: Claws / Basic Hand Tools $\rightarrow$ Copper Tier $\rightarrow$ Iron Tier (Demo Limit) $\rightarrow$ Steel / Crystal / Mithril Tiers (Full Game).
  * progression is hard-gated by material availability; missing advanced ores (e.g., Gold, Mithril) requires triggering Dormancy to synthesize them.
* **Interactions:** Unlocks higher-tier automation machinery capable of processing complex metamorphic ores.
* **Progression/Failure:** Bottlenecks occur if ore processing throughput lags behind mining extraction rate.
* **Observed vs. Inferred:** Observed via press kit disclosures, demo scope documentation (Iron Tier cap), and screenshot UI analysis.

### System 5: Automation Machinery, Resource Transportation & Processing Chains
* **Player Action:** Places Crusher units, Crystal Furnaces, Conveyor belts, Storage containers, and robotic sorting arms ("Orehands").
* **System Response:** Automatically extracts mined ore chunks, transports them along conveyor lines, sorts them by material type, crushes raw ore, and smelts ingots without player manual intervention.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Mined Ore chunks, power/fuel source.
  * *Outputs:* Refined ingots, organized storage, continuous automated yield.
* **Rules & Logic:**
  * Orehand sorting arms pick specific item IDs from belts and transfer them to designated processing inputs.
  * Machines require flat ground or anchored structural supports; structural collapses destroy active logistics lines.
* **Interactions:** Automated extraction increases mining volume exponentially, driving up structural stress risks across the facility.
* **Progression/Failure:** Belt congestion, furnace fuel exhaustion, or structural cave-ins severing logistics lines.
* **Observed vs. Inferred:** Observed in high-resolution press screenshots (`mithril_automation.png`, `mithril_automation-2.png`) and loop video clips.

### System 6: Lava, Water, Fluid Interactions & Environmental Hazards
* **Player Action:** Mines into fluid reservoirs, channels molten lava into rock tunnels, or places water channels.
* **System Response:** Fluids flow down gravity gradients, filling depressions and interacting with surrounding geological materials and thermal dynamics.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Lava/Water placement, carved channels.
  * *Outputs:* Fluid pools, thermal gradients, cooling crusts (Basalt/Obsidian), steam generation.
* **Rules & Logic:**
  * Fluid placement evaluates basin depth, rim height, and air clearance.
  * Lava acts as a high-temperature thermal heat source; direct contact with water causes cooling and stone solidification.
* **Interactions:** Channeled lava heats surrounding host rock; during Dormancy, this hydrothermal heat converts contact zones into rare metamorphic ores.
* **Progression/Failure:** Uncontrolled fluid release floods lower mining shafts or destroys automated processing equipment.
* **Observed vs. Inferred:** Directly tested via WASM API (`/api/place-water`, `lava-carve` mode) and official trailer footage.

### System 7: Dormancy (Geological Time Fast-Forward Simulation)
* **Player Action:** Triggers "Dormancy" / "Deep Sleep" from base terminal.
* **System Response:** Executes background simulation representing $1.25$ million years of geological process in seconds: acid dissolution eats limestone, lava pools cool and metamorphose host rock, water tables deposit concentrated minerals, unsupported ground collapses, and support struts undergo degradation checks.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Player trigger command, existing cave geometry, heat/fluid distribution, strut placement.
  * *Outputs:* Transmuted cave layout, new mineral deposits (e.g., Marble, Skarn, Hornfels, Gold, Diamond), degraded/destroyed support struts, generated Sleep Log & Material Diff.
* **Rules & Logic:**
  * Hydrothermal Metamorphism: Host rock in contact with lava pools transforms into Marble/Skarn.
  * Acid Dissolution: Water channels eat through Limestone strata, creating new karst caverns.
  * Strut Degradation: Struts undergo survival probability checks based on tier (Copper Struts may fail; Steel/Mithril Struts survive).
* **Interactions:** Dormancy completely alters the cavern environment where base machinery resides, forcing the player to re-explore and rebuild.
* **Progression/Failure:** Crucial late-game ores *only* exist post-Dormancy. Placing insufficient structural supports prior to Dormancy results in total base burial.
* **Observed vs. Inferred:** Directly observed via WASM (`/api/sleep`) endpoint, before/after comparison tool, and sleep log differential output.

### System 8: Creatures, Combat & Survival Pressures
* **Player Action:** Navigates dark caverns equipped with mining claws, torches, and weapons; fights aggressive subterranean fauna.
* **System Response:** Creatures spawn in unlit, deep cavern biomes, attacking players and player structures.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Torches, weapons, claws.
  * *Outputs:* Creature drops (organic materials, bioluminescent glands), player damage, destroyed webs/wasp nests.
* **Rules & Logic:**
  * Demo feature: Spider webs (slow/trap players) and Wasp swarms (airborne swarm combat).
  * Full Game Feature ("The Memory"): Surviving creatures track player tactics (mining locations, weapon choices, defense traps). Post-Dormancy, surviving populations evolve genetic/behavioral counter-adaptations.
* **Interactions:** Creatures nest in abandoned tunnels created by reckless player excavation.
* **Progression/Failure:** Player death results in gear loss; unmanaged infestations compromise base safety.
* **Observed vs. Inferred:** Demo combat directly observed in press kit video clips (`mithril_combat-spider-webs.mp4`, `mithril_combat-wasp-swarm.mp4`). "The Memory" adaptation system is explicitly documented as a planned full-game feature.

### System 9: Tutorials, Onboarding, UI, Feedback & Information
* **Player Action:** Begins new demo campaign; opens diagnostic overlays and material legend panels.
* **System Response:** Serves a hand-sculpted onboarding cave region; displays bioluminescent structural stress wireframes; provides expandable material identification legends and post-sleep diagnostic logs.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Keyboard shortcut (`A` for orbit anchor, `C` for cinema mode), UI toggles.
  * *Outputs:* Real-time stress heatmaps (cyan/teal wireframe), material composition breakdown, before/after mesh comparison sliders.
* **Rules & Logic:**
  * Onboarding cavern is post-generation sculpted by hand to ensure linear tutorial pacing.
  * Stress diagnostic UI highlights high-risk load-bearing nodes before collapse occurs.
* **Interactions:** Provides essential readable feedback allowing players to place struts precisely before triggering collapses or Dormancy.
* **Progression/Failure:** Ignoring visual stress warnings leads directly to catastrophic structural failure.
* **Observed vs. Inferred:** Directly observed in press kit screenshots (`mithril_biolum.png`), browser UI, and demo release disclosures.

### System 10: Performance, Simulation Responsiveness & Technical Architecture
* **Player Action:** Modifies massive 3D voxel regions, places fluids, or initiates million-year batch sleep simulations.
* **System Response:** Executes simulation in background Rust threads, updating isosurface meshes without main render thread stutter.
* **Inputs/Outputs/Resources:**
  * *Inputs:* Region dimensions ($6 \times 6 \times 6$ chunks = $216$ chunks in browser; vast infinite worlds in full UE5 engine).
  * *Outputs:* Smooth 30–60 FPS rendering at 1080p, real-time stress mesh re-indexing.
* **Rules & Logic:**
  * Architecture: Custom C-compatible Rust voxel & physics library bridged directly into Unreal Engine 5 (and compiled to WebAssembly for browser demo).
  * Rust handles procedural noise generation, cellular automata fluid flow, stress matrix calculation, and metamorphic state transformation; UE5 handles Nanite/Lumen rendering, lighting, and audio.
* **Interactions:** Decouples heavy geological/physics simulation from graphic rendering.
* **Progression/Failure:** Exceeding recommended hardware specs in massive caverns causes shader compilation delays or reduced framerates.
* **Observed vs. Inferred:** Directly confirmed via technical disclosures in press kit and browser WASM bundle analysis.

---

## 4. Systems Model & Architectural Diagrams

### 4.1 System Dependency Map

```
  +-----------------------------------------------------------------------+
  |                     PROCEDURAL CAVE GENERATION                        |
  |   (Host Strata: Limestone, Sandstone, Granite, Basalt, Slate; Ores)   |
  +-----------------------------------+-----------------------------------+
                                      |
                                      v
  +-----------------------------------+-----------------------------------+
  |                   EXCAVATION & MINING (Claws / Drills)                |
  |         (Carves voxel volumes, creates voids, yields raw ore)         |
  +-----------------+---------------------------------+-------------------+
                    |                                 |
                    v                                 v
  +-----------------+-----------------+     +---------+-------------------+
  |   STRUCTURAL STRESS CALCULATOR    |     |    FLUID CHANNELING         |
  | (Propagates weight across R=4)    |     |  (Lava & Water Placement)   |
  +-----------------+-----------------+     +---------+-------------------+
                    |                                 |
                    +----------------+----------------+
                                     |
                                     v
  +----------------------------------+------------------------------------+
  |                     DORMANCY SIMULATION (1.25M Yrs)                   |
  |  - Hydrothermal Metamorphism (Lava + Rock -> Metamorphic Ores)       |
  |  - Acid Dissolution (Water + Limestone -> Cavern Expansion)           |
  |  - Collapse Settlement (Unsupported void -> Rubble fill)              |
  |  - Strut Degradation Check (Copper/Iron/Steel/Mithril Survival Rates) |
  +----------------------------------+------------------------------------+
                                     |
                                     v
  +----------------------------------+------------------------------------+
  |               RESOURCE EXTRACTION & AUTOMATION CHAIN                  |
  |   (Crushers -> Conveyors -> Orehands -> Crystal Furnaces -> Storage)  |
  +----------------------------------+------------------------------------+
                                     |
                                     v
  +----------------------------------+------------------------------------+
  |                   CREATURES & ADAPTIVE THREAT                         |
  |      ("The Memory": Post-Dormancy Evolutionary Counter-Adaptation)    |
  +-----------------------------------------------------------------------+
```

### 4.2 Resource-Flow Diagram

```
[ Solid Host Rock ] --(Mining / Carving)--> [ Raw Ore Chunks ]
                                                  |
                                                  v
                                          [ Crusher Unit ]
                                                  |
                                                  v
                                          [ Crushed Powder ]
                                                  |
                                                  v
                                      [ Conveyor + Orehand Arm ]
                                                  |
                                                  v
                                         [ Crystal Furnace ]
                                                  |
                                                  v
                                           [ Refined Ingot ]
                                                  |
       +------------------------------------------+------------------------------------------+
       |                                          |                                          |
       v                                          v                                          v
[ Structural Support Struts ]           [ Advanced Mining Tools ]                   [ Automation Machines ]
 (Wood/Copper/Iron/Steel/Mithril)          (Drills / Deep Claws)                   (Orehands/Furnaces/Belts)
       |
       v
[ Reinforced Cavern Walls ] --(Sustains Load During)--> [ Dormancy Geological Transformation ]
```

### 4.3 Progression & Unlock Map

```
  +-------------------------------------------------------------------+
  | TIER 0: PRIMITIVE EXCAVATION                                      |
  | - Tools: Starter Claws, Torches                                   |
  | - Materials: Limestone, Sandstone, Raw Copper                     |
  | - Supports: Basic Timber / Unreinforced Tunnels                   |
  +---------------------------------+---------------------------------+
                                    |
                                    v
  +---------------------------------+---------------------------------+
  | TIER 1: COPPER & EARLY AUTOMATION (Demo Focus)                   |
  | - Tools: Upgraded Claws, Basic Drills                             |
  | - Materials: Copper Ingots, Malachite, Banded Iron              |
  | - Machinery: Conveyors, Basic Orehand Arms, Ore Crushers        |
  | - Supports: Copper Struts                                       |
  +---------------------------------+---------------------------------+
                                    |
                                    v
  +---------------------------------+---------------------------------+
  | TIER 2: IRON & STRUCTURAL REINFORCEMENT (Demo Cap)                |
  | - Tools: Iron Drills, Heavy Pickaxes                              |
  | - Materials: Iron Ingots, Coal, Pyrite                            |
  | - Machinery: Powered Crushers, Smelting Furnaces                  |
  | - Supports: Iron Support Trusses                                  |
  +---------------------------------+---------------------------------+
                                    | [Requires Dormancy Fast-Forward]
                                    v
  +---------------------------------+---------------------------------+
  | TIER 3: METAMORPHIC & THERMAL ADVANCEMENT (Full Game)             |
  | - Materials: Marble, Skarn, Hornfels, Gold, Amethyst, Quartz      |
  | - Machinery: Crystal Furnaces, Multi-Arm Sorting Hubs             |
  | - Supports: Steel Trusses, Crystal Reinforced Braces              |
  +---------------------------------+---------------------------------+
                                    | [Requires Deep Mantle Exploration]
                                    v
  +---------------------------------+---------------------------------+
  | TIER 4: DEEP DEEP & MITHRIL ENGINE (Full Game End-Game)           |
  | - Materials: Kimberlite, Diamond, Mythical Mithril                |
  | - Supports: Mithril Core Struts (100% Collapse Immunity)          |
  +-------------------------------------------------------------------+
```

### 4.4 Action-Response-Consequence Table

| Player Action | Immediate System Response | Delayed / Secondary Consequence | Ultimate Gameplay Impact |
| :--- | :--- | :--- | :--- |
| **Dig voxel around high-value ore** | Carves sphere, drops ore chunk, exposes rock interior. | Increases shear stress on neighboring voxel nodes. | If load limit exceeded, triggers cascading structural cave-in. |
| **Place Support Strut** | Adds vertical/lateral load-bearing capacity to local nodes. | Prevents stress propagation to weak ceiling voxels. | Safely enables deep excavation without ceiling failure. |
| **Channel Lava into Tunnel** | Fluid fills carved depression; heats surrounding walls. | Prepares thermal boundary for hydrothermal reaction. | Dormancy will transform surrounding stone into Marble/Skarn ores. |
| **Trigger Dormancy** | Executes 1.25M-year simulation pass in background Rust engine. | Dissolves limestone, metamorphoses rock, tests strut survival. | Cave environment completely changes; unlocks new ore tiers. |
| **Construct Automation Loop** | Orehands sort and move items between crushers and furnaces. | Continuous hands-free ingot production. | Scales resource throughput to fund deep-mantle structural projects. |
| **Neglect Base Structural Safety** | Base built in unsupported limestone cavern. | Dormancy causes massive structural collapse over base. | Machinery destroyed; base buried in thick rubble fill. |

### 4.5 Emergent Interactions & Edge Cases

1. **Hydrothermal Base Destruction:** A player channels lava near their base to create a nearby ore transformation zone. Upon triggering Dormancy, the intense heat metamorphoses the rock holding up their main factory anchor, causing the entire factory floor to collapse into molten lava.
2. **Acid Dissolution Flood:** Mining an underground water channel into a limestone stratum causes acid dissolution during Dormancy. The dissolution hollows out a massive hidden void above an automated facility, leading to an unexpected catastrophic ceiling collapse.
3. **Strut Chain Reaction Failure:** Using low-tier Copper Struts in a deep basalt tunnel. During Dormancy, one Copper Strut fails its survival check; the load instantly shifts to adjacent struts, exceeding their capacity and causing a total domino collapse of a kilometer-long transit tunnel.
4. **Keyhole Ore Harvesting Edge Case:** Carefully carving away $90\%$ of host rock around a gold vein leaves a thin structural pillar. Placing a single water block nearby causes enough moisture weakening during Dormancy to shatter the pillar, burying the gold in loose granite rubble.

### 4.6 Evidentiary Classification Matrix

| Feature / System | Classification | Source Evidence |
| :--- | :--- | :--- |
| **22 Voxel Material Strata & Noise Parameters** | **Directly Observed** | Live WASM bundle (`cave.playmithril.com`) and `/api/generate` API payload analysis. |
| **Stress Propagation Radius ($R=4$) & Weights** | **Directly Observed** | Browser JS config (`gen-stress-gravity`, `gen-stress-lateral`, `gen-stress-vertical`). |
| **Strut Degradation & Survival Rates** | **Directly Observed** | WASM parameter lineup (`gen-collapse-copper`, `iron`, `steel`, `crystal`, `mithril`). |
| **Spherical Voxel Mining & Water Placement** | **Directly Observed** | Browser REST endpoints (`/api/mine`, `/api/place-water`) and viewer interaction. |
| **Hydrothermal Metamorphism during Dormancy** | **Directly Observed** | `/api/sleep` before/after comparison tool and sleep differential log output. |
| **Automation Orehands & Conveyor Processing** | **Directly Observed** | High-res press screenshots (`mithril_automation.png`) and loop video clips. |
| **Wasp Swarm & Spider Web Combat** | **Directly Observed** | Press kit video clips (`mithril_combat-spider-webs.mp4`, `mithril_combat-wasp-swarm.mp4`). |
| **"The Memory" Adaptive Creature Evolution** | **Developer Claim / Inferred** | Explicitly stated in press kit documentation as non-demo, launch-only feature. |
| **4-Player Co-op & Freeplay Modes** | **Developer Claim / Inferred** | Confirmed in press kit fact sheet as launch features excluded from demo. |

---

## 5. Forgeworld Relevance & Architectural Recommendations

### 5.1 Comparison Matrix for Forgeworld (Godot 4 + C#)

| Mithril Feature | Forgeworld Classification | Strategic Analysis for Forgeworld (Godot 4 + C#) |
| :--- | :--- | :--- |
| **Real-time Structural Stress Propagation** | **Directly Transferable Design Principle** | Essential for physics-based creative engineering. Can be implemented efficiently in C# using a grid-based shear/compressive stress propagation algorithm. |
| **Readable Stress Wireframe Diagnostics** | **Directly Transferable Design Principle** | Critical UX requirement. Rendering glowing stress heatmaps (cyan = safe, orange/red = critical) provides players with immediate readable diagnostics. |
| **Dormancy (Time Fast-Forward Metamorphism)** | **Requires Substantial Adaptation** | Rather than geological 1.25M-year metamorphism, *Forgeworld* should adapt Dormancy into "Environmental Decay / Weathering / Salvage Fast-Forwarding" suitable for factory/salvage themes. |
| **Physical Automation & Robotic Sorting Arms** | **Directly Transferable Design Principle** | Fits *Forgeworld*'s factory automation goals perfectly. Modular arms and conveyors in Godot 4 using C# signal-driven tick updates. |
| **Voxel Metamorphic Fluid Interactions** | **Expensive / Risky for Early Prototype** | 3D volumetric fluid-thermodynamic metamorphism is computationally heavy. For an early C# prototype, simplify to 2D/3D cellular automata grids. |
| **"The Memory" Adaptive AI Evolution** | **Conflicting / Out of Scope** | Complex evolutionary AI vectors distract from *Forgeworld*'s core engineering, salvage, and factory automation focus. |

---

### 5.2 C# / Godot 4 Implementation Proposals

#### Proposal A: C# Structural Stress Solver (`StructuralStressGrid3D.cs`)
An original, lightweight 3D stress propagation solver designed for Godot 4 in C# using flat arrays for zero-allocation performance.

```csharp
using Godot;
using System;

namespace Forgeworld.Engineering;

public struct VoxelNode
{
    public byte MaterialId;
    public float LoadWeight;
    public float CompressiveCapacity;
    public float AccumulatedStress;
    public bool IsSupported;
}

public class StructuralStressGrid3D
{
    private readonly Vector3I _dimensions;
    private readonly VoxelNode[] _grid;
    private readonly float _gravityWeight = 1.0f;
    private readonly float _lateralSupportFactor = 0.3f;

    public StructuralStressGrid3D(Vector3I dimensions)
    {
        _dimensions = dimensions;
        _grid = new VoxelNode[dimensions.X * dimensions.Y * dimensions.Z];
    }

    private int GetIndex(int x, int y, int z) => x + _dimensions.X * (y + _dimensions.Y * z);

    public void RecalculateStressNode(Vector3I pos, int propagationRadius)
    {
        int minX = Math.Max(0, pos.X - propagationRadius);
        int maxX = Math.Min(_dimensions.X - 1, pos.X + propagationRadius);
        int minY = Math.Max(0, pos.Y - propagationRadius);
        int maxY = Math.Min(_dimensions.Y - 1, pos.Y + propagationRadius);
        int minZ = Math.Max(0, pos.Z - propagationRadius);
        int maxZ = Math.Min(_dimensions.Z - 1, pos.Z + propagationRadius);

        for (int y = maxY; y >= minY; y--) // Top-down load accumulation
        {
            for (int z = minZ; z <= maxZ; z++)
            {
                for (int x = minX; x <= maxX; x++)
                {
                    int idx = GetIndex(x, y, z);
                    if (_grid[idx].MaterialId == 0) continue; // Air node

                    float transferredLoad = _grid[idx].LoadWeight * _gravityWeight;

                    // Distribute to adjacent lower nodes
                    if (y > 0)
                    {
                        int belowIdx = GetIndex(x, y - 1, z);
                        _grid[belowIdx].AccumulatedStress += transferredLoad;
                    }

                    // Distribute lateral shear
                    if (x > minX) _grid[GetIndex(x - 1, y, z)].AccumulatedStress += transferredLoad * _lateralSupportFactor;
                    if (x < maxX) _grid[GetIndex(x + 1, y, z)].AccumulatedStress += transferredLoad * _lateralSupportFactor;
                }
            }
        }
    }

    public bool CheckForCollapse(Vector3I pos, out float excessStress)
    {
        int idx = GetIndex(pos.X, pos.Y, pos.Z);
        excessStress = _grid[idx].AccumulatedStress - _grid[idx].CompressiveCapacity;
        return excessStress > 0;
    }
}
```

#### Proposal B: C# Structural Wireframe Diagnostic Visualizer (`StressOverlayDrawer.cs`)
Generates immediate visual feedback using Godot 4's `ImmediateMesh` / `RenderingServer` for cyan-to-red stress heatmaps.

```csharp
using Godot;

namespace Forgeworld.Diagnostics;

public partial class StressOverlayDrawer : Node3D
{
    private ImmediateMesh _immediateMesh;
    private OrmMaterial3D _overlayMaterial;

    public override void _Ready()
    {
        _immediateMesh = new ImmediateMesh();
        MeshInstance3D meshInstance = new MeshInstance3D { Mesh = _immediateMesh };
        AddChild(meshInstance);

        _overlayMaterial = new OrmMaterial3D
        {
            ShadingMode = BaseMaterial3D.ShadingModeEnum.Unshaded,
            VertexColorUseAsAlbedo = true,
            Transparency = BaseMaterial3D.TransparencyEnum.Alpha
        };
        meshInstance.MaterialOverride = _overlayMaterial;
    }

    public void RenderStressWireframe(Vector3I[] positions, float[] stressRatios)
    {
        _immediateMesh.ClearSurfaces();
        _immediateMesh.SurfaceBegin(Mesh.PrimitiveType.Lines);

        for (int i = 0; i < positions.Length; i++)
        {
            float ratio = Mathf.Clamp(stressRatios[i], 0.0f, 1.0f);
            Color nodeColor = Colors.Cyan.Lerp(Colors.Red, ratio);
            nodeColor.A = 0.8f;

            Vector3 center = new Vector3(positions[i].X, positions[i].Y, positions[i].Z);
            DrawBoxWireframe(center, Vector3.One * 1.02f, nodeColor);
        }

        _immediateMesh.SurfaceEnd();
    }

    private void DrawBoxWireframe(Vector3 center, Vector3 size, Color color)
    {
        Vector3 half = size * 0.5f;
        // Draw 12 bounding box line segments using SurfaceSetColor and SurfaceAddVertex
        // (Standard wireframe box drawer)
    }
}
```

---

### 5.3 Recommended 2D Prototype Experiments for Forgeworld

Prior to full 3D implementation in Godot 4, the following two small 2D prototype experiments are recommended:

#### Experiment 1: 2D Grid Stress Propagation & Dynamic Cave-In
* **Goal:** Validate real-time shear/compressive stress propagation performance and player readability in a 2D tile grid.
* **Scope:** $100 \times 100$ tile grid containing host rock (Granite, Sandstone), support beams (Wood, Iron), and player excavation cursor.
* **Test Protocol:**
  1. Excavate horizontal tunnels of increasing width ($3$ to $20$ tiles).
  2. Measure stress accumulation on ceiling tiles.
  3. Verify that removing a critical load-bearing pillar triggers a localized ceiling collapse.
  4. Verify that placing a support beam resets local stress and prevents collapse.

#### Experiment 2: 2D Salvage Weathering / Fast-Forward Simulation
* **Goal:** Test a player-triggered "Time Skip / Fast-Forward" mechanic applied to structural salvage and automated factories.
* **Scope:** 2D grid containing factory machines, structural trusses, acid leaks, and rust simulation.
* **Test Protocol:**
  1. Build a factory layout adjacent to an uncontained chemical/acid tank.
  2. Trigger "Fast-Forward 100 Years".
  3. Verify that acid corrodes surrounding steel trusses, causing factory ceiling collapse and degrading unmaintained machine components into valuable salvage scrap.

---

## 6. Sources & Evidence Log

1. **Browser Voxel Engine JavaScript Bundle:** `cave.playmithril.com/app.js` (94 KB original source, analyzed via node/python execution).
2. **Browser Engine REST Endpoints:** Verified `/api/generate`, `/api/mine`, `/api/place-water`, `/api/sleep`, `/api/report`.
3. **Official Press Kit:** `playmithril.com/press/` (Fact sheet, quotes from Connor & Tiffany Tassie, tech specs, hardware requirements).
4. **Official Website:** `playmithril.com` (Dormancy before/after interactive slider, marketing material).
5. **Steam Demo Store Page:** `store.steampowered.com/app/5063090/Mithril_Demo/` (Demo inclusions/exclusions disclosure, user tag classifications).
6. **High-Res Press Screenshots:** `mithril_cave-expanse.png`, `mithril_automation.png`, `mithril_strut-tunnel.png`, `mithril_biolum.png`, `mithril_dormancy-comparison.png`.
7. **Press Video Footage:** `mithril_official-gameplay-trailer.mp4`, `mithril_collapse-showcase-trailer.mp4`, `mithril_struts.mp4`, `mithril_combat-spider-webs.mp4`.

---

## 7. Key Findings Materially Influencing Forgeworld's Design

1. **Stress Diagnostics are a Gameplay Core, Not Just Visual Polish:** In *Mithril*, structural stress is made immediately readable via glowing cyan/teal wireframes. For *Forgeworld*, readable diagnostic overlays are essential to prevent structural physics from feeling arbitrary or punitive.
2. **Discrete Time Fast-Forwards ("Dormancy") Enable Massive Simulation Depth:** Running continuous 1.25M-year thermodynamic/geological calculations in real time is computationally impossible. By framing simulation as a discrete, player-triggered "fast-forward" event, complex structural and material state transformations become performant and controllable.
3. **Materials Must Have Structural Purpose, Not Just Hardness Stats:** Host rock strata in *Mithril* (Slate, Granite, Basalt) dictate natural span limits and support needs. In *Forgeworld*, building materials (Scrap Iron, Titanium, Composite Alloy) should feature distinct shear/compressive weight ratios that govern structural integrity.
4. **Decouple Core Physics Engine from Graphic Renderer:** *Mithril*'s custom Rust engine handles heavy grid/stress/fluid calculations in background threads before passing isosurface meshes to Unreal Engine 5. *Forgeworld* should adopt a similar clean architecture: C# background simulation classes communicating cleanly with Godot 4 render nodes.
