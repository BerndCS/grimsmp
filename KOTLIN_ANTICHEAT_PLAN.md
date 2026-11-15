# Build Instructions: Ultra-Lightweight Kotlin Movement Anticheat ("BoomAC" / "Boom Anticheat")

> This document is the blueprint/prompt for a **brand-new** plugin project.
> It is independent of the existing GrimAC fork (`common`/`bukkit`/`fabric`) and shares
> **no code** with it — only insights from its architecture feed into the "avoid this"
> vs. "keep this" decisions below.

## 0. Findings from analyzing the GrimAC fork (rationale for the decisions below)

| Observation in the fork | Consequence for the new plugin |
|---|---|
| `common`/`bukkit`/`fabric` multi-module setup with `buildSrc` convention plugins (`grim.base-conventions`, `grim.shadow-conventions`), Java 21 toolchain, bytecode target 17, `libs.versions.toml` version catalog | Reuse the same base tooling (version catalog + convention plugin), but as **a single module** — no multi-platform complexity (no Fabric/vanilla target needed) |
| PacketEvents (`com.github.retrooper:packetevents`) as the core dependency; `CheckManagerListener` hooks into **every** client packet (`PLAYER_BLOCK_PLACEMENT`, `PLAYER_DIGGING`, `PLAYER_FLYING`, `USE_ITEM`, ...) | **No PacketEvents/ProtocolLib.** The new plugin uses only native Bukkit/Paper events (`PlayerMoveEvent`, `PlayerVelocityEvent`, `EntityMoveEvent` where available) — zero packet-interception overhead, no ViaVersion/Folia-Netty threading compatibility issues |
| `CheckManager` registers 150+ checks in an `ImmutableClassToInstanceMap`, including the entire block placement/breaking legality suite (`scaffolding`, `breaking`, `combat.Reach`, etc.) | **Only 4 checks total.** Placement/interaction/reach code is never written in the first place — not "disabled", but **structurally absent from day one** |
| `GrimPlayer` holds a full simulation state (compensated world, prediction engine, latency compensation) — powerful but heavyweight | The new plugin needs **no** physics simulation of the world. A lean `PlayerMovementState` (last position, velocity, timestamp, violation level) per player is enough |
| `bukkit/build.gradle.kts` already declares `foliaSupported = true`, but the architecture relies on global player objects + packet listeners (not truly region-thread-safe) | The new plugin must be **genuinely** Folia/CanvasMC-safe: all per-player state is only ever mutated on the thread that owns that player's region (`RegionScheduler`/`EntityScheduler`/`GlobalRegionScheduler`) |
| Punishment system via YAML groups (`punishments/*.yml`) with thresholds/commands | For v1, a simple in-memory violation counter with decay and a minimal threshold → command execution is enough. No complex group/rule system needed |

## 1. Architecture & Compatibility

- **Platform:** PaperMC API (`paper-api`), target versions 1.20–1.21+, compatible with **Folia** and **CanvasMC** (both are Paper forks with region-based multithreading).
- **No packet listeners.** No dependency on PacketEvents, ProtocolLib, or similar. Exclusively Bukkit API events:
  - `PlayerMoveEvent` (core driver for Fly/Speed/HighJump)
  - `VehicleMoveEvent` (for BoatFly)
  - `PlayerToggleFlightEvent`, `PlayerToggleSprintEvent` as state sources (no raw packet parsing)
- **Scheduler strategy (Folia/CanvasMC):**
  - Runtime detection: `Bukkit.getServer().getClass().getPackage().getName()` or `try { Class.forName("io.papermc.paper.threadedregions.RegionizedServer") }` → feature flag `isFolia`.
  - On Folia/Canvas: **always** use `player.scheduler.run(...)` (per-entity `EntityScheduler`) for anything tied to a specific player, `Bukkit.getRegionScheduler().run(plugin, location, task)` for location-bound work (e.g. block/ground checks at a position), and `Bukkit.getGlobalRegionScheduler()` **only** for truly global, player-independent bookkeeping (e.g. periodic cleanup of the state registry) — never for per-player mutation.
  - On classic Paper/Spigot: fall back to `BukkitScheduler` (synchronous, since `PlayerMoveEvent` already fires on the correct thread).
  - **No global tick loop that iterates over all players.** Core checks run purely event-driven (`PlayerMoveEvent`/`VehicleMoveEvent`), keeping baseline load at zero when nobody is moving.
  - **Optional per-player periodic validation:** if a check ever needs sampling beyond move events (e.g. detecting a client that stopped sending movement packets while airborne), use `player.scheduler.runAtFixedRate(plugin, task, retired, delay, period)` (Folia's per-entity scheduler) — never a single global `runTaskTimer` looping over all online players, since that reintroduces a cross-region bottleneck.
  - **State storage:** `ConcurrentHashMap<UUID, PlayerMovementState>` — accessed only from the relevant event-handler thread (the region thread on Folia), so no plugin-wide locks are needed since exactly one thread ever writes to a given player's state at a time.
- **No custom bounding-box/collision simulation.** Reuse native Paper values wherever possible (`Player#getVelocity()`, `Player#isFlying()`, `Player#getAllowFlight()`, `Block#isPassable()` for a single ground check) instead of re-implementing physics.

## 2. Performance Design (hard constraints)

- **Zero-allocation hot paths.** `onMove`/`onVehicleMove` and every `MovementCheck.evaluate()` must not allocate on the common (non-violating) path: prefer primitive fields on a mutable `PlayerMovementState` over creating a new `MovementContext` object per event; avoid boxed `Double`/`Float`, avoid creating lists/maps per tick, avoid string formatting unless a violation actually fires.
- **Simple statistical models, not state machines.** Each check is a stateless pure function of `(previousState, currentEvent) -> Violation?` based on a threshold/tolerance comparison (e.g. expected vs. actual delta compared against a constant). No multi-step phase/state-machine tracking, no historical buffers beyond the single previous sample needed for the comparison.
- **No GUI, no complex configuration.** A handful of primitive thresholds in `config.yml` (max fly tolerance, max speed, max jump height, boat-fly grace ticks) is the entire configuration surface. Violation tracking is a minimal in-memory `IntArray`/counter per player with time-based decay — no persistent storage, no rule groups, no punishment DSL.

## 3. Included Checks (Scope)

Only purely mathematical, low-state checks — each check class exposes a **single** `evaluate(): Violation?` entry point, invoked from `MovementHandler`:

1. **FlyCheck** — Vertical movement without proper fall behavior: allowed upward acceleration per tick exceeded while `allowFlight == false` and no elytra/levitation/explosion context is active. Purely mathematical: `deltaY` compared against the expected vanilla gravity curve (`(lastVelocityY - 0.08) * 0.98`) with a tolerance band.
2. **SpeedCheck** — Horizontal velocity: `sqrt(deltaX² + deltaZ²)` per tick compared against an allowed maximum (accounting for sprint/sneak/potion effects like Speed/Slowness as simple additive modifiers only — no block-friction simulation).
3. **HighJumpCheck** — Jump height: peak of `y - groundY` after a jump (detected via the `onGround` transition) compared against the vanilla max jump height (~1.2521 blocks, + jump-boost level as an additive term).
4. **BoatFlyCheck** — Same idea as FlyCheck, but for `VehicleMoveEvent` on `Boat`/`ChestBoat` entities: no water/ground contact over N consecutive ticks while `deltaY` stays positive.

All four checks:
- No access to chunk/block NBT data beyond a single `getBlock(...).getType()` call for one ground check (no ray tracing, no multi-block iteration).
- No setback/rollback of the player's position (no custom netcode handling) — on violation, only increase the violation level and optionally alert/kick via command; **never** reset the client's claimed position (that would require re-introducing movement-packet/state tracking, which is intentionally out of scope).

## 4. Excluded Checks (IMPORTANT — never implement, not even as disabled code)

- No block placement validation whatsoever (no `AirLiquidPlace`, `RotationPlace`, `PositionPlace`, `FabricatedPlace`, `MultiPlace` equivalent).
- No block breaking validation whatsoever (no `FastBreak`, `RotationBreak`, `NoSwingBreak` equivalent).
- No reach/hitbox/interaction validation whatsoever (no `Reach`, `Hitboxes`, `MultiInteract`, `SelfInteract` equivalent).
- **No rotation check of any kind** — neither standalone aim/rotation validation nor rotation checks coupled to block or entity interaction.
- Explicitly **allowed and never flagged**:
  - **AirPlace** (placing blocks against air/fluid)
  - **Litematica EasyPlace** and **Litematica Printer** (fast, repeated, look-direction-independent placements)
  - **Tick-based/stretched placements in mid-air** ("air-tick-place")
- Since the plugin registers **no** `PlayerInteractEvent`/`BlockPlaceEvent`/`BlockBreakEvent` listener at all, there is structurally no point where any of this could ever be blocked — no feature flag, no "disableable" check, simply not present.

## 5. Target Project Structure

```
boomac/
  build.gradle.kts                # Kotlin DSL, Kotlin plugin + Shadow, paper-api compileOnly
  gradle.properties
  settings.gradle.kts
  libs.versions.toml               # minimal: paper-api, kotlin-stdlib, adventure (optional)
  src/main/kotlin/ac/boomac/
    BoomACPlugin.kt                 # JavaPlugin entry point, Folia detection, scheduler wrapper, config load
    scheduler/
      RegionSchedulerAdapter.kt     # abstracts Folia RegionScheduler/EntityScheduler/GlobalRegionScheduler vs. Bukkit scheduler behind one interface
    state/
      PlayerMovementState.kt        # data class: lastX/Y/Z, lastVelocityY, groundTick, violationLevel, decayTimestamp
      PlayerStateRegistry.kt        # ConcurrentHashMap<UUID, PlayerMovementState> + PlayerQuitEvent cleanup
    handler/
      MovementHandler.kt            # central PlayerMoveEvent/VehicleMoveEvent listener, invokes checks, manages violations
    checks/
      MovementCheck.kt              # interface: fun evaluate(state: PlayerMovementState, event: MovementContext): Violation?
      FlyCheck.kt
      SpeedCheck.kt
      HighJumpCheck.kt
      BoatFlyCheck.kt
    violation/
      ViolationManager.kt           # decay logic, thresholds, command execution via scheduler
  src/main/resources/
    plugin.yml                      # foliaSupported: true, no depend: [ProtocolLib/packetevents]
    config.yml                      # per-check thresholds, decay time, alert/punish commands
```

## 6. Code Skeleton (implementation requirements)

### `MovementHandler.kt` (core, event-driven, no tick loop)
```kotlin
class MovementHandler(
    private val states: PlayerStateRegistry,
    private val checks: List<MovementCheck>,
    private val violations: ViolationManager,
) : Listener {

    @EventHandler(ignoreCancelled = true, priority = EventPriority.MONITOR)
    fun onMove(event: PlayerMoveEvent) {
        val to = event.to ?: return
        val state = states.getOrCreate(event.player.uniqueId)
        val ctx = MovementContext.fromBukkitMove(event.player, event.from, to, state)

        for (check in checks) {
            val violation = check.evaluate(state, ctx) ?: continue
            violations.record(event.player, check.name, violation)
        }

        state.update(ctx) // purely arithmetic update, only one getBlock() call for the ground check
    }

    @EventHandler
    fun onVehicleMove(event: VehicleMoveEvent) {
        val boat = event.vehicle as? Boat ?: return
        // BoatFlyCheck runs through the same pipeline
    }

    @EventHandler
    fun onQuit(event: PlayerQuitEvent) = states.remove(event.player.uniqueId)
}
```

### `FlyCheck.kt` (example of a "purely mathematical" check)
```kotlin
class FlyCheck(private val config: FlyCheckConfig) : MovementCheck {
    override val name = "Fly"

    override fun evaluate(state: PlayerMovementState, ctx: MovementContext): Violation? {
        if (ctx.player.allowFlight || ctx.player.isGliding || ctx.isNearGround) return null

        val expectedDeltaY = (state.lastVelocityY - 0.08) * 0.98 // vanilla gravity formula
        val actualDeltaY = ctx.to.y - ctx.from.y

        return if (actualDeltaY - expectedDeltaY > config.tolerance) {
            Violation(weight = 1.0, details = "deltaY=$actualDeltaY expected=$expectedDeltaY")
        } else null
    }
}
```

### Gradle (`build.gradle.kts`, sketch)
```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    id("com.gradleup.shadow") version "9.3.0"
}

dependencies {
    compileOnly(libs.paper.api) // API compatibility only, no PacketEvents/ProtocolLib dependency
}

kotlin {
    jvmToolchain(21)
}
```

## 7. Acceptance Criteria

- [ ] The plugin registers **only** `PlayerMoveEvent`, `VehicleMoveEvent`, `PlayerQuitEvent` (+ optionally `PlayerJoinEvent` for state init).
- [ ] `build.gradle.kts` has no dependency on PacketEvents, ProtocolLib, or Netty interception.
- [ ] No `BlockPlaceEvent`/`BlockBreakEvent`/`PlayerInteractEvent` listener anywhere in the project.
- [ ] No standalone or interaction-coupled rotation/aim check exists anywhere in the project.
- [ ] All per-player state mutation demonstrably happens on the event thread (the region thread on Folia) — no plugin-wide `synchronized` blocks needed.
- [ ] `plugin.yml` declares `foliaSupported: true` and has **no** `depend`/`softdepend` on packet libraries.
- [ ] Hot paths (`onMove`, `evaluate()`) perform no heap allocation on the non-violating path.
- [ ] Manual test scenario: Litematica EasyPlace/Printer + AirPlace produce **zero** violations (since no code path exists that could ever flag them).
