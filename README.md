# KĀLCHAKRA: SHADOWS OF DHARMA
## FINAL PRODUCTION CODEX (INTERNAL)
**Document Class:** Creative + Technical Master Spec  
**Audience:** Game Direction, Engineering, AI, Narrative, Audio, Animation, Tech Art, Production  
**Build Intent:** Shipping-quality vertical-slice-to-full-production blueprint  

---

## SECTION 1 — PROJECT FOLDER STRUCTURE (UNREAL ENGINE 5.4 RECOMMENDED)

> We are standardizing on **Unreal Engine 5.4 C++ + Blueprint hybrid** for high-fidelity rendering, scalable systems architecture, and deterministic performance control across PC + mobile targets.

### 1.1 Root Structure

```text
/Kalchakra
  /Build
  /Config
  /Content
  /Plugins
  /Source
  /Scripts
  /Tools
  /Docs
  /Tests
  /DerivedDataCache (excluded from source control)
```

### 1.2 Production Content + Code Tree

```text
/Source
  /Kalchakra
    /Core
      KCGameInstance.*
      KCGameMode.*
      KCWorldSubsystem.*
      KCAssetManager.*
      KCPlatformProfile.*
    /Framework
      KCStateMachine.*
      KCEventBus.*
      KCDataRegistry.*
      KCSaveSubsystem.*
      KCInputRouter.*
    /Gameplay
      /Player
        KCPlayerCharacter.*
        KCPlayerController.*
        KCPlayerMovementComponent.*
        KCPlayerCombatComponent.*
        KCPlayerStaminaComponent.*
        KCPlayerInjuryComponent.*
        KCPlayerFearComponent.*
        KCInventoryComponent.*
        KCInteractionComponent.*
      /Combat
        KCDamageModel.*
        KCHitResolver.*
        KCWeaponBase.*
        KCMeleeWeapon.*
        KCRangedWeapon.*
      /Items
        KCItemDefinition.*
        KCItemInstance.*
        KCConsumable.*
        KCAmmo.*
      /World
        KCInteractableBase.*
        KCCheckpointActor.*
        KCLightFlickerController.*
    /AI
      /Core
        KCAIControllerBase.*
        KCPerceptionComponent.*
        KCAIMemoryComponent.*
        KCAggressionModel.*
      /Behavior
        BT_* (Behavior Trees)
        BB_* (Blackboards)
        EQS_* (Environment Queries)
      /Enemies
        KCBhutaController.*
        KCPretController.*
        KCAsuraEliteController.*
      /Director
        KCHorrorDirectorSubsystem.*
        KCEncounterBudgeter.*
    /Animation
      KCAnimInstance_Player.*
      KCAnimInstance_Enemy.*
      KCRagdollBlendComponent.*
      KCMotionWarpProfile.*
    /Audio
      KCAudioStateSubsystem.*
      KCFearAudioModulator.*
      KCWhisperEmitter.*
      KCSoundPropagation.*
    /UI
      /HUD
      /Menus
      /Inventory
      /Prompts
      KCUIManagerSubsystem.*
      KCUIScalingPolicy.*
    /Narrative
      KCNarrativeSubsystem.*
      KCDialogueGraph.*
      KCLoreEntry.*
      KCWhisperTrack.*
    /PhysicsMath
      KCCollisionChannels.h
      KCDamageCurves.*
      KCFearCurves.*
      KCTimeDilationModel.*
    /Optimization
      KCPerfBudgeter.*
      KCLODPolicy.*
      KCObjectPool.*
      KCStreamingRules.*
    /Platform
      /PC
      /Android
      /iOS
        KCPlatformInputMap_*.*
        KCPlatformPerfProfile_*.*

/Content
  /Art
    /Characters
    /Enemies
    /Environment
    /VFX
    /Materials
  /Animation
    /Player
    /Enemies
    /Cinematics
  /Audio
    /SFX
    /VO
    /Music
    /Ambience
    /Procedural
  /Cinematics
  /UI
  /Narrative
    /DataAssets
    /Dialogue
    /Lore
  /Design
    /DataTables
    /Curves
    /Tuning
  /Levels
    /Hub
    /Chapters
    /TestMaps
  /AI
    /BehaviorTrees
    /Blackboards
    /EQS
```

### 1.3 Folder Responsibilities, Naming, Dependencies

- **Core**  
  - Purpose: engine lifecycle, startup orchestration, subsystem registration.  
  - Script types: game instance, world subsystem, platform profiles.  
  - Naming: `KC{SystemName}`.  
  - Dependencies: none upward; every layer may depend on Core.

- **Framework**  
  - Purpose: reusable architecture utilities (FSM base, event bus, save interfaces, data registry).  
  - Script types: abstract classes/interfaces only; no game fiction details.  
  - Naming: `IKC*` for interfaces, `FKC*` for pure structs, `UKC*` / `AKC*` by Unreal convention.  
  - Dependencies: Core only.

- **Gameplay**  
  - Purpose: player + combat + interaction + items runtime logic.  
  - Script types: actor components, actor classes, gameplay tags, tuning adapters.  
  - Naming: `KCPlayer*`, `KCWeapon*`, `KCItem*`.  
  - Dependencies: Framework + PhysicsMath + Audio + Animation.

- **AI**  
  - Purpose: enemy cognition and encounter orchestration.  
  - Script types: AI controllers, BT tasks/services/decorators, perception adapters.  
  - Naming: `KCAI*`, `BT_KC_*`, `BB_KC_*`.  
  - Dependencies: Framework + Gameplay read-only interfaces + Narrative flags.

- **PhysicsMath**  
  - Purpose: deterministic formulas and simulation constants.  
  - Script types: curve assets, equation utilities, collision masks.  
  - Naming: `KC*Model`, `KC*Curve`.  
  - Dependencies: Core only.

- **Animation**  
  - Purpose: locomotion graphs, layered blends, hit reactions, IK, ragdoll handoff.  
  - Dependencies: Gameplay state exposure + PhysicsMath constants.

- **Audio**  
  - Purpose: fear-reactive mix states, diegetic cues, occlusion/propagation.  
  - Dependencies: Gameplay/AI events via event bus.

- **UI**  
  - Purpose: HUD, inventory grid, diegetic prompts, accessibility overlays.  
  - Dependencies: Framework event bus + Gameplay query interfaces only.

- **Narrative**  
  - Purpose: lore graph, whisper trigger timeline, chapter states, symbolic motif binding.  
  - Dependencies: Core + Framework; publishes to Audio/UI.

- **Optimization**  
  - Purpose: frame-time budget enforcement, pool management, streaming hints.  
  - Dependencies: all runtime layers read this policy; this layer depends on none game-specific.

- **Platform**  
  - Purpose: platform-specific input maps, quality tiers, save encryption adapters.  
  - Dependencies: Core + Framework only (to avoid branching deep gameplay logic).

---

## SECTION 2 — PLAYER SYSTEMS ARCHITECTURE

### 2.1 System Topology

Player is an **Entity-Component aggregate** on `AKCPlayerCharacter`:
- `KCPlayerMovementComponent`
- `KCPlayerCombatComponent`
- `KCPlayerStaminaComponent`
- `KCPlayerInjuryComponent`
- `KCPlayerFearComponent`
- `KCInventoryComponent`
- `KCInteractionComponent`
- `KCCameraRigComponent`

All components communicate through an internal priority-ordered signal bus (`FKCPlayerSignal`).

### 2.2 Movement System

Modes:
- **Walk**: base locomotion, low noise, stable aim sway.
- **Sprint**: velocity boost, high stamina drain, high footstep sound emission.
- **Crouch**: reduced profile, reduced speed, reduced noise, improved stealth.

Locomotion state machine:
- `Idle -> Walk -> Sprint`
- `Walk <-> Crouch`
- Any state -> `Stagger` on heavy hit
- Any state -> `Limp` when leg injury threshold reached

Priority resolution:
1. Hard crowd-control (grab, knockback)
2. Injury forced mode (`Limp`, `Fall`)
3. Player intention (input)
4. Cosmetic overlays (breathing sway)

### 2.3 Camera Logic

- Over-shoulder default offset: `(X=0, Y=58, Z=64)` right shoulder.
- Shoulder swap input toggles Y sign with 0.2s smooth damp.
- Dynamic FOV curve:
  - 72° default
  - 78° sprint
  - 68° ADS / precision throw
  - 64° panic tunnel vision when Fear > 0.85
- Camera collision probe uses sphere sweep to avoid clipping.
- Trauma shake layered from Injury and near-miss events; capped to preserve readability.

### 2.4 Combat System

- **Melee-first survival model**; ranged is scarce and intentionally punitive.
- Melee attacks: light, heavy, shove-break.
- Ranged: improvised pistol/bow equivalents with severe ammo scarcity.
- Hit validation: animation notify opens hit window; server-authoritative in deterministic timeline.
- Stance penalties:
  - Low stamina: slower windup + weaker stagger chance.
  - Bleeding: increased recoil jitter.

### 2.5 Stamina + Breath

- Stamina pool (0–100), nonlinear regen with breath cadence.
- Breath control mechanic:
  - Hold breath to reduce aiming sway / audio signature.
  - Holding breath raises panic debt; if overused, induces gasp spike (sound emission burst).

### 2.6 Injury System

Tracks zones: `Head`, `Torso`, `LArm`, `RArm`, `LLeg`, `RLeg`.

Effects:
- Leg severe -> limp gait + no sprint.
- Arm severe -> slower reload/melee recovery.
- Head trauma -> intermittent blur + tinnitus ring.
- Torso bleed -> periodic health drain until bandaged.

### 2.7 Sanity / Fear Meter

Fear is continuous scalar `Fear ∈ [0,1]` influenced by:
- darkness exposure,
- proximity to unseen entities,
- gore events,
- whisper triggers,
- low resources.

Threshold effects:
- `0.35`: subtle breathing amplification.
- `0.60`: UI edge distortion + minor input jitter.
- `0.85`: perception unreliability events (false footsteps, shadow traces).

### 2.8 Inventory (Grid Survival)

- Fixed grid (e.g., 6x4 baseline), item footprints (`1x1`, `1x2`, `2x2`, irregular masks).
- Stack rules by item category and condition.
- Quick slots are mirrored subset, not separate containers.
- Weight + bulk both matter: bulk blocks placement; weight impacts stamina regen.

### 2.9 Interaction System

- Priority cone trace every 100ms (not every frame).
- Candidates scored by `distance * angle * context weight * narrative urgency`.
- No floating objective arrows; only diegetic prompts and actor-facing interaction framing.

### 2.10 Save + Checkpoint Logic

- Save channels:
  1) Checkpoint autosave (safe shrines / major progression)
  2) Emergency suspend save (platform-level interruption)
  3) Manual save only at sanctified rest points
- Serialized domains:
  - World state deltas
  - Inventory / injury / fear
  - AI director intensity seed
  - Narrative flags + consumed lore entries

### 2.11 Update Loops

- `Tick_PrePhysics`: input aggregation, intention decoding.
- `Tick_Physics`: movement, collision, stamina drain.
- `Tick_PostPhysics`: camera settle, animation sync, audio events.
- `FixedInterval(100ms)`: interaction scan, AI noise publish.
- `FixedInterval(250ms)`: fear evaluation + hallucination scheduler.

### 2.12 Cross-Platform Input Abstraction

Action layer (`KCInputRouter`) maps abstract actions:
`Move`, `Look`, `Sprint`, `CrouchToggle`, `AttackLight`, `AttackHeavy`, `Aim`, `Use`, `Inventory`, `ShoulderSwap`, `BreathHold`.

Platform adapters:
- PC: mouse/keyboard + gamepad dual stack.
- Mobile: virtual sticks + contextual radial action buttons + gyro assist for fine aim.
- iOS/Android use adaptive touch zones based on screen class (small/medium/tablet).

---

## SECTION 3 — ENEMY AI & MACHINE LOGIC

### 3.1 AI Stack Overview

Each enemy:
- `KCAIControllerBase`
- `KCPerceptionComponent` (Sight/Sound/Vibration)
- `KCAIMemoryComponent`
- Behavior Tree + Blackboard
- Local FSM for animation-tight transitions

Global:
- `KCHorrorDirectorSubsystem` manages pacing and unpredictability budget.

### 3.2 Perception Model

- **Sight**: cone + luminance-adjusted detection (player in darkness harder to resolve).
- **Sound**: event-driven; each player action emits `NoiseEvent{loudness, surfaceType, dampening}`.
- **Vibration**: ground impulse channel for heavy steps/sprints in proximity.

Detection confidence accumulates over time; single-frame spikes cannot fully detect player unless at extreme proximity.

### 3.3 FSM (Micro-State)

States:
- `Dormant`
- `Patrol`
- `Investigate`
- `Search`
- `Chase`
- `Attack`
- `Retreat` (for fear-reactive species)
- `Enrage`
- `Stunned`
- `Dead`

Text diagram:

```text
Dormant -> Patrol
Patrol -> Investigate (stimulus heard/seen weak)
Investigate -> Search (no confirmation)
Investigate -> Chase (confirmed player)
Search -> Patrol (timeout)
Search -> Chase (reacquire)
Chase -> Attack (range + LOS)
Attack -> Chase (target moved)
Chase -> Enrage (aggro meter high + ally death trigger)
Any -> Stunned (heavy impact)
Stunned -> Search/Chase (by confidence)
Any -> Dead (health <= 0)
```

### 3.4 Behavior Tree (Macro Decision)

Root selectors:
1. Survival override (dead/stunned/retreat)
2. Combat branch
3. Hunt/search branch
4. Patrol/ambient branch

Services update:
- target confidence,
- last known position,
- aggression scalar,
- local threat map.

### 3.5 Alert Propagation

- Enemies publish alerts to nearby allies with attenuation by walls/material.
- Alert payload: `position`, `confidence`, `threatType`, `timestamp`.
- Recipients blend own confidence with incoming signal; prevents instant hive-mind omniscience.

### 3.6 Search Patterns

EQS-generated candidate points around Last Known Position (LKP):
- spiral outward sample,
- cover points,
- high-ground vantage,
- sound-reflective blind spots.

Pattern phase timings randomized by constrained seed to preserve unpredictability without chaos.

### 3.7 Memory Model

`KCAIMemoryComponent` keeps:
- `LKP`
- `LastHeardEvent`
- `LastDamageSource`
- `ThreatDecayTimer`
- `PlayerBehaviorBias` (player often crouches/sprints/uses light)

Memory decays using species-specific half-life.

### 3.8 Dynamic Aggression Scaling

Aggression scalar `A ∈ [0,1.5]` based on:
- player dominance (recent kills),
- enemy losses,
- director tension budget,
- chapter narrative stakes.

High `A`: shorter windups, bolder flanks, reduced retreat chance.

### 3.9 Horror Unpredictability Logic

Controlled entropy layer:
- At high fear, director can inject “non-lethal anomalies” (distant roar, false movement).
- Hard cap on anomalies per 10-minute window to avoid fatigue.
- Never invalidate player mastery: fake cues may mislead, never rewrite confirmed facts.

### 3.10 AI Pseudocode

```pseudo
function UpdateAI(dt):
    perceive = SenseWorld(dt)
    memory.Update(perceive, dt)

    confidence = ComputeTargetConfidence(memory, perceive)
    aggression = AggressionModel.Evaluate(context, directorIntensity)

    fsm.TransitionByRules(confidence, aggression, damageState)

    switch fsm.state:
        case PATROL:
            ExecutePatrolRoute()
        case INVESTIGATE:
            MoveTo(memory.LastStimulusPosition)
            if confidence > CONFIRM_THRESHOLD:
                fsm.SetState(CHASE)
        case SEARCH:
            points = EQS.GenerateSearchPoints(memory.LKP, threatMap)
            Sweep(points)
        case CHASE:
            PredictivePursuit(playerVelocityEstimate)
            if InAttackWindow(): fsm.SetState(ATTACK)
        case ATTACK:
            ExecuteAttackCombo(aggression)
            BroadcastAlertIfPlayerConfirmed()
        case ENRAGE:
            ApplyAggroBuffs()
            ForceFlankPattern()
```

### 3.11 AI Performance Rules

- Perception LOD tiers by distance/on-screen relevance.
- Off-screen enemies use reduced tick rate (250–500ms).
- BT service frequencies staggered to avoid frame spikes.
- Use object pools for transient investigation markers.
- Navmesh queries budgeted per frame; overflow deferred.

---

## SECTION 4 — PHYSICS & MATHEMATICS

### 4.1 Collision + Hit Detection

- Capsule for character locomotion.
- Multi-sphere traces for melee arcs during notify windows.
- Projectile raycast with penetration check against material resistance.
- Hurtboxes per body zone for injury specificity.

### 4.2 Damage Formula


a) Base impact:
\[
D_{raw} = W_{base} \cdot M_{attackType} \cdot M_{stamina} \cdot M_{crit}
\]

b) Mitigation:
\[
D_{final} = \max(1, D_{raw} \cdot (1 - R_{armor}) \cdot (1 - R_{zone}))
\]

- `M_stamina = lerp(0.75, 1.0, staminaRatio)` prevents full output while exhausted.
- `R_zone` enables helmets/ritual armor/body vulnerabilities.

**Why:** preserves survival tension by rewarding preparation and punishing reckless spam.

### 4.3 Injury Accumulation

For each zone `z`:
\[
I_z(t+1) = clamp( I_z(t) + D_{zone} \cdot K_{injuryType} - H_{treated}, 0, 100)
\]

Thresholds trigger locomotion/aim penalties.

**Why:** damage has narrative consequence beyond health bar attrition.

### 4.4 Stamina Drain + Regen

Drain:
\[
S_{t+1} = S_t - (C_{action} \cdot W_{load} \cdot F_{injury})\Delta t
\]

Regen (nonlinear):
\[
S_{t+1} = S_t + (R_{base} \cdot (1 - Fear^{1.6}) \cdot BreathControlFactor)\Delta t
\]

**Why:** fear physiologically suppresses recovery, tying mechanics to horror tone.

### 4.5 Fear Escalation Curve

\[
Fear_{t+1} = clamp(Fear_t + \sum stimuli_i\cdot w_i - CalmFactors, 0, 1)
\]

Stimulus amplification near cap:
\[
w_i' = w_i\cdot (1 + Fear_t^2)
\]

**Why:** panic spirals are non-linear and humanly believable.

### 4.6 Time Dilation (Kālchakra Effect)

Local time scale around anomalies:
\[
\tau_{local} = \tau_{global} \cdot (1 - A_{chakra}\cdot e^{-d^2/\sigma^2})
\]

where `d` = distance to fracture center.

- Affects particles, cloth, ambient audio pitch, some enemy windups.
- Player input latency compensation ensures controls remain crisp.

**Why:** mythological time fracture must be felt physically, not only seen.

### 4.7 Ragdoll + Animation Blend

Blend weight:
\[
\alpha = smoothstep(v_{impact}, v_{threshold}, v_{max})
\]

Pose blend from animation to physics over 120–300ms based on impact severity.

**Why:** keeps kills visceral while preserving readability and performance.

### 4.8 Sound Falloff + Occlusion

Distance attenuation:
\[
V(d)=\frac{1}{1+(d/d_0)^p}
\]

Occlusion factor by material stack:
\[
V_{occ}=V(d)\cdot\prod_j (1-\mu_j)
\]

**Why:** stealth relies on trustworthy audio simulation.

---

## SECTION 5 — WORLD BUILDING & LORE SYSTEM

### 5.1 Origin of Kālchakra

Kālchakra is an ancient cosmological engine built by rishis to synchronize **Time (Kāla)**, **Action (Karma)**, and **Order (Dharma)**. It was never a weapon. It was a balancing mechanism distributing consequence across cycles of existence. When invoked by empires for immortality, it fractured into recursive temporal wounds. The world now bleeds moments from dead ages into present night.

### 5.2 Rules of Time Fracture

1. A fracture anchors to unresolved mass karma (war sites, ritual betrayals).  
2. Memory and matter desynchronize: people remember events that have not yet occurred.  
3. Repeating zones are not loops; they are **stacked timelines with slight moral divergence**.  
4. Large fractures attract corrupted devas/asuras as pattern-seeking predators.

### 5.3 Why Gods Are Corrupted

Devas were sustained by coherent worship aligned with dharma-function. Humanity shifted to transactional devotion (power for offerings), creating asymmetric feedback. Divinities became hypertrophied in one attribute (wrath, hunger, protection) and atrophied in balance. They are not evil caricatures; they are diseased absolutes.

### 5.4 Human Role in Cosmic Balance

Humans are the only agents capable of choosing between competing duties under uncertainty. That choice capacity is the missing variable the Kālchakra requires to heal fractures. The protagonist is not “chosen”; they are necessary because they still doubt.

### 5.5 Asura Philosophy

Asuras in this world are anti-stagnation intelligences. They argue that “order without renewal is slow extinction.” Their violence is ideological: break false dharma so authentic dharma can re-emerge. Some are enemies, some uneasy truth-tellers.

### 5.6 Dharma as System, Not Morality

Dharma is modeled as **relational integrity**:
- duty to self survival,
- duty to kin/community,
- duty to cosmic continuity.

No binary good/evil meter. Decisions shift dynamic world equations: NPC trust, zone hostility, fracture stability, available endings.

### 5.7 Lore Delivery System

Channels:
- **Documents:** ritual logs, military reports, temple annotations.
- **Visual relics:** murals with phase overlays that change under time distortion.
- **Whispers:** directional, language-layered voice fragments tied to fear state.
- **Embodied memory scenes:** short, controllable flash intrusions in playable space.

Rule: every lore object must either alter interpretation of a prior event or forecast mechanical danger.

### 5.8 Environmental Storytelling Rules

- Blood trails must have origin + consequence (no decorative gore).
- Shrines show maintenance state to communicate social collapse timeline.
- Weather and ash density track chapter-level fracture intensity.
- Reused spaces evolve physically and spiritually; never static revisits.

### 5.9 Symbolism Language (Art + Sound)

- **Broken circles**: interrupted cycles / failed rebirth.
- **Three-tone temple bell motif**: stable dharma zones.
- **Inverted conch drone**: corrupted divine presence.
- **Asymmetric mandala geometry** in UI distortion: fear encroachment.

---

## SECTION 6 — 30-MINUTE OPENING EXPERIENCE (CINEMATIC + GAMEPLAY)

### 6.1 Sequence Overview

- 00:00–04:30: Prologue cutscene (loss, oath, fracture omen)
- 04:30–12:00: Controlled exploration through ruined riverside quarter
- 12:00–16:00: First fear event (unseen pursuit via sound)
- 16:00–23:00: First hostile encounter (single Bhūta)
- 23:00–30:00: Escape to sanctuary, first dharma choice

### 6.2 Intro Cutscene (Emotionally Heavy)

**Scene:** Monsoon night. A half-collapsed temple archive above a flooded city edge.

**Camera Direction:**
- Macro closeups: trembling fingers binding torn scripture with bloodied thread.
- Slow dolly to reveal protagonist `ARIN` kneeling beside dying mentor `VYOMA`.
- Lightning strobe reveals mural where time-wheel spokes are missing.

**Dialogue:**
- **Vyoma (faint):** “Do not seek to win against night… only keep it from learning your name.”
- **Arin:** “Tell me where to carry this.”
- **Vyoma:** “Not where. *When.*”
- Temple groans; distant conch tone plays backward.
- **Vyoma:** “If the gods ask for obedience, ask them what they forgot.”

Vyoma dies clutching Arin’s wrist. His pulse stops exactly as bell rings third tone—then rings again out of sequence (first fracture cue).

### 6.3 Transition to Gameplay

Control returns with no HUD for first 90 seconds.
- Objective is implied: exit collapsing archive with relic cylinder.
- Environmental teaching:
  - leaning beam requires crouch,
  - flooded corridor punishes sprint with loud splash,
  - flickering lamp near shelf implies interaction prompt when centered.

### 6.4 Slow Exploration Beat

Player enters alley network:
- distant chanting with impossible directionality,
- homes marked with handprints at different heights (child/adult panic history),
- first inventory acquisition: cloth wrap + oil flask from abandoned shrine.

No enemy yet. Tension built through absence, not attack.

### 6.5 First Fear Moment (Sound-Driven)

At 12 minutes:
- All ambience ducks by 30%.
- Player hears wet footsteps matching their cadence, always one step behind.
- Turning camera shows no actor.
- Shoulder camera subtly narrows to 66°.
- Whisper in left channel: “You arrived late, again.”

Mechanically: Fear jumps to 0.58; minor aim tremor introduced.

### 6.6 First Enemy Encounter

Location: market courtyard with hanging bells.

Teaching by environment:
- broken pottery line indicates stealth path.
- light pool under shrine attracts Bhūta patrol.
- throwable pebble near corpse cues distraction system.

Encounter flow:
1. Player can sneak around (preferred) or engage.
2. If melee engaged with low stamina, Bhūta punishes with grapple and near-death choke.
3. Contextual shove-break taught via animation struggle, no text.

**Enemy vocalization:** childlike inhale before lunge (signature telegraph).

### 6.7 Sanctuary and First Choice

At ~26 minutes player reaches cracked sanctum door.
Inside: two survivors, one infected by fracture delirium.

Choice (no morality labels):
- give scarce antiseptic to child (compassion, higher risk), or
- preserve antiseptic for self (survival, survivor mistrust).

**Dialogue beat:**
- **Survivor Mother:** “If law still lives, it is in what you do now.”
- Arin remains silent; player decides.

End of opening: camera rises to skyline showing three active fracture pillars. Title card fades in over low drum + conch inversion.

---

## SECTION 7 — GAME FEEL & QUALITY BAR

### 7.1 What Makes It AAA

- Input-to-motion latency under 90ms perceived on all platforms.
- Weighty locomotion with precise stop/start readability.
- High-fidelity facial/body performance in close emotional scenes.
- Audio scene graph where every horror beat is physically authored, not random.
- Lighting that preserves silhouette readability in darkness.

### 7.2 Must Never Feel Cheap

- No canned jump-scare spam.
- No teleporting enemies inside player camera frustum.
- No animation foot sliding during emotional scenes.
- No UI popups that explain fear; player must feel it diegetically.

### 7.3 Animation Priorities

1. Locomotion transitions + turn-in-place.
2. Injury overlays (limp, guarded torso, one-arm strain).
3. Contextual melee contact and reaction sync.
4. Hand IK for doors, ledges, ritual objects.

### 7.4 Sound Priorities

1. Footstep material fidelity + wetness state.
2. Enemy telegraph signature cues.
3. Breath + heartbeat layers tied to fear/stamina.
4. Dynamic silence as active design tool.

### 7.5 Lighting Philosophy

- Darkness is volumetric and directional, not uniform black.
- Warmth indicates temporary human order.
- Cyan/green spectral spill marks time fracture bleed.
- Eye adaptation tuned slowly to preserve dread and uncertainty.

### 7.6 Horror Pacing Rules

- 3:1 ratio of anticipation to payoff in early chapters.
- Every intense encounter followed by reflective decompression window.
- Escalation by complication, not pure enemy count.

### 7.7 Polish Last (if Schedule Constrained)

- Secondary cloth simulation on distant NPCs.
- Non-critical ambient wildlife behaviors.
- Optional collectible VO variants.

### 7.8 Cut First (if Time Limited)

- Alternate weapon skin variants.
- Excessive cinematic camera flourishes during traversal.
- Non-essential branching barks with low narrative delta.

---

## SECTION 8 — TECHNICAL PIPELINE

### 8.1 Engine Recommendation

**Unreal Engine 5.4** with:
- C++ core gameplay/AI systems
- Blueprint for authoring orchestration and rapid iteration
- Control Rig + Motion Warping
- World Partition with strict streaming volumes

### 8.2 Mobile Optimization Strategy

- Feature parity of mechanics, not parity of visual complexity.
- Mobile-specific renderer profile:
  - lower shadow cascades,
  - baked GI where possible,
  - reduced volumetric step counts,
  - material quality tiers.
- Use ASTC textures and aggressive mesh LODs.

### 8.3 LOD Rules

- Characters: minimum 4 LODs + impostor for crowds.
- Enemies in combat radius locked above LOD2.
- Animation LOD: full body IK disabled beyond relevance threshold.
- Audio LOD: distant one-shots merged to ambient buses.

### 8.4 Memory Management

- Hard budgets per platform profile (example):
  - PC: 10–12 GB runtime target,
  - Android high-tier: 3.0 GB,
  - iOS high-tier: 3.5 GB.
- Stream chapter chunks; keep only adjacent cells resident.
- Object pooling for projectiles, decals, AI markers, transient VFX.

### 8.5 Save System Design

- Binary chunked saves with versioned schema.
- Deterministic GUID for world actors storing only deltas.
- Checksums + optional encryption on mobile.
- Fail-safe rolling slots: `slot_a`, `slot_b`, `slot_recovery`.

### 8.6 Cross-Platform UI Scaling

- Design in logical units anchored to safe-area constraints.
- Use adaptive presets:
  - 16:9, 19.5:9, tablet 4:3 classes.
- Minimum touch target 9mm equivalent on mobile.
- Typography scales by angular size, not fixed px.

### 8.7 Performance Targets

- PC target: 60 FPS @ 1440p (high preset), 30 FPS floor worst-case.
- Mobile target: 30 FPS locked with stable frame pacing.
- Frame budget at 30 FPS: 33.3ms
  - Game thread ≤ 10ms
  - Render thread ≤ 12ms
  - GPU ≤ 16ms (overlap expected)

---

## SECTION 9 — DEVELOPMENT PHILOSOPHY (DIRECTORATE)

This game stands for **dignity under terror**.

KĀLCHAKRA is not about power fantasy. It is about endurance when belief systems collapse, when gods become unreliable, and when survival requires choosing which duty to betray. The player must leave with the weight of their decisions, not the score of their victories.

### 9.1 Emotional Afterimage

Player should feel:
- haunted, not startled,
- exhausted, yet morally awake,
- small before cosmic machinery, yet responsible.

### 9.2 Mistakes We Must Avoid

- Confusing obscurity with depth.
- Replacing authored fear with procedural noise.
- Over-explaining mythology until it loses sacred ambiguity.
- Letting combat efficiency erase vulnerability.

### 9.3 Non-Negotiables

- Every system must reinforce theme: fractured time, costly agency, fragile dharma.
- Narrative and mechanics must agree; no ludonarrative contradiction in critical beats.
- Performance stability is creative quality; stutter destroys fear rhythm.

### 9.4 Why This Game Deserves to Exist

Most horror asks, “Can you survive?”
This game asks, “What survives *because* of you?”

KĀLCHAKRA: SHADOWS OF DHARMA deserves existence because it treats mythology as living systems design, horror as moral pressure, and player choice as cosmological architecture—not cosmetic branching. If we execute with discipline, this becomes a benchmark for culturally grounded survival horror at AAA quality.

---

## CODIFIED ACCEPTANCE CHECKLIST (GREENLIGHT GATE)

- Player can complete opening 30 minutes with zero tutorial text and still learn core verbs.
- Fear meter materially changes audio, camera, AI perception outcomes.
- Injury states alter traversal/combat in legible, fair ways.
- Enemy AI demonstrates non-scripted search + memory + alert propagation.
- Lore pickups recontextualize active gameplay risk.
- PC/mobile builds preserve mechanic parity and emotional pacing.

**If any of the above fails, content scale is reduced before quality is reduced.**
