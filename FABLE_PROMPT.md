# Prompt for Fable: deep improvements to Flights of Fantasy

Paste everything below the line into a new session on the `flightsim2026` repo.

---

You are working on **Flights of Fantasy** (repo `flightsim2026`), a Unity 6 flight
and space simulator whose rule is **derive, don't tune**. Every number is either
cited (a handbook, an ICAO annex, a NASA mission report) or labelled a choice in
the source next to it.

## Before you touch anything

1. Read `CLAUDE.md` in full. The "Rules I keep having to learn" section records
   mistakes that have each cost real time, and you are expected to follow it.
2. Pull the latest master. Other sessions push to it often.
3. Run the headless suite: `dotnet test Tools/SimCoreTests/SimCoreTests.csproj`.
   Note what passes before you start.
4. A trawl on 29 Sep 2026 produced the list below. **Every item is a claim, not a
   fact.** Open the cited file and confirm the fault before fixing it. Line
   numbers will have drifted. If a claim is wrong, say so and move on.

## How to work

- **SimCore has no `using UnityEngine`, and it stays that way.** Decisions that
  change what a craft is asked to do belong in SimCore as pure, tested functions.
  `Game` only reads state and writes demands. `Game` has no tests, so logic moved
  into SimCore is logic that becomes testable.
- **Tests assert against something outside the code:** a POH table, a mission
  report, IERS constants, plain geometry. Never a round trip through the model's
  own inputs.
- **After any change to `Game` or `Editor`**, run `./Tools/check-unity-build.sh`
  if you are on a Mac with Unity. If you can't, say plainly in the commit message
  and the summary that the Game code has not been compiled by Unity.
- **When you change behaviour**, update `Docs/Manual.md` in the same commit, and
  `Docs/WorldRules.md` if it has a row for it. Then run
  `python3 Tools/BuildManual/build-manual.py`.
- **Commit and push:**
  - one commit per theme, with a message that says what was wrong and what it
    cost;
  - fetch and merge `origin/master` before pushing, and never reset or
    force-push.
- **At the end**, give me:
  - a short report: what you fixed, what you verified, what is unverified in the
    Editor (say exactly which screen and what to press to see it), and what you
    found to be a false claim;
  - a ranked list of what to do next.

## Part A: confirmed or near-certain bugs (do these first, all small)

1. **Clouds use render coordinates.** In `Game/World/CloudLayer.cs`, `Paint`,
   `builtAround` and the sheet Y ignore `WorldOrigin`, and the class never
   subscribes to `WorldOrigin.Shifted`. So the cloud pattern jumps about 5 km at
   every origin shift, and the base drifts up by the altitude of the shift.
   `WorldSetup.cs` (around line 1496) asks `Weather.InCloudAt` with the render
   position, so the "in cloud" fog disagrees with the drawn cloud and with the
   storm cell (`ModernFlightModel.cs` around line 2511), which uses true
   coordinates.
2. **A reset on the pad doesn't stop the countdown.** `countingDown` and
   `countdownStartedAt` (`WorldSetup.cs` around line 5295) are not reset in
   `BackToDefaults` (around line 4213), so the count can `Ignite()` the new
   craft. Also decide and document whether `recordingMoment`, `sectionsLeft` and
   `sectionWas` should be reset.
3. **A getter steps the hop state machine.** `ProgrammeThrottle` (around line
   9287) is a property getter that calls `hop.Step`, writes state and can end
   the hop. `TraceTheDescent` (around line 9766) reads it every frame just to log
   it. That makes a third caller where `HopProgramme.cs` says there must be two.
   Make it an explicit once-per-physics-step method, have the trace read
   `hop.Last`, and grep for other getters with side effects.
4. **The moment trail is cached once for all missions.** `momentPlaces` /
   `momentStages` (`WorldSetup.cs` around line 6290) is filled the first time and
   never cleared, but `MissionMoments` differs per mission. Flying Apollo 11 and
   then Apollo 13 in one session gives the wrong trail, and possibly an index
   out of range, which Error Pause turns into what looks like a hang. Key it on
   the mission, as `plannedFor` does.
5. **A fourth "a section's end is not the next event".** The journey map
   (`WorldSetup.cs` around line 5820) places arrival at `get + ex.NextEventS`
   when `TliCutOffGetS` is NaN, which is every cold start after TLI.
   `NextEventS` is the docking, or on Apollo 13 the explosion. CLAUDE.md says
   that the fourth time this happens, the cure is structural: add a `SectionEndS`
   written only by the machine that owns the section, and split it from the warp
   stop in `MoonExplorer` (around lines 190–201 and 339–387).
6. **Terrain ring 0 is sunk 0.8 m.** `TerrainStreamer.cs` sinks every ring by
   `lodSink*step`, including the innermost one, which has no ring inside it. The
   wheels float above visible ground off the runway. Use zero sink at level 0.
7. **A GPU stall once a second.** `Sky.cs` (around line 1137) does synchronous
   `ReadPixels` / `GetPixel` up to 8 times a second. Use `AsyncGPUReadback`.
8. **The night-sky log floods the flight log.** `NightSky.cs` (around line 502)
   builds a 12-field string every frame, and fast-changing fields defeat the
   dedupe, so real events are pushed out of the ring buffer. Log only on a state
   change. Also:
   - `NightSky.cs` (around line 577) allocates a new array every frame;
   - `SkyPath.cs` (around line 634) creates a new `GUIStyle` for every label;
   - `PlaceMarkers`, `SurfaceLabels` and `LunarPlaces` create a new
     `MaterialPropertyBlock` for every marker every frame.
9. **Vne is checked against true airspeed.** `ModernFlightModel.cs` (around line
   2702) compares true airspeed with `VneMs * 1.15`, but the P-51 and Spitfire
   Vne figures are indicated airspeed. At altitude that halves the dive envelope.
   Compare indicated airspeed, and cite the 1.15 margin or label it a choice.
10. **Two Earths and two Moons.**
    - `Cislunar.EarthMu` = 9.80665·R², which is 0.14% below IERS 3.986004418e14,
      while `ColossusEntry` implies the true GM.
    - `LunarCraft` uses 4.9028e12, which differs from `Cislunar.MoonMu`.
    - Gravity fallbacks: `?? 1.625` appears 20 times and `?? 9.80665` about 5
      times.
    - Hard-coded 3.711 and 1.625 appear in `LanderPanel` and `MenuBar`.

    Put each constant in one place, add one `WorldSetup.Gravity`, and add a test
    against IERS Conventions 2010 and GRAIL.
11. **C# version mismatch.** `Tools/SimCoreTests/SimCoreTests.csproj` uses
    `LangVersion 10.0` and Unity uses 9, so C# 10 code in SimCore passes CI and
    then breaks Unity. Pin the headless project to 9.

## Part B: deep themes (pick in this order unless I say otherwise; each is a day or more)

### B1. Architecture: carve up the Game seam so it can be tested

- `WorldSetup.cs` is 12,000 lines, `MoonExplorer` 5,700 and `MenuBar` 5,100.
- **Programme supervisor.** Move "which programme is flying" (`Programme Auto`
  is assigned in about 25 places; see `CheckTheProgramme` and
  `ProgrammeThrottle`) into a `SimCore.Flight.ProgrammeSupervisor`. It takes a
  situation snapshot and returns the next programme, the throttle and a list of
  events. WorldSetup only turns the events into speech and log lines.
- **Shared input pipeline.** `MoonExplorer` (around lines 5140–5420) and
  `MarsHopper` (around lines 370–581) hold copies of the input-to-demand code
  that have already drifted: the hopper scales the autopilot's demands by stick
  authority. Replace both with one `SimCore.Controls.PilotDemand.Compose`, used
  by both and by the rig.
- **One craft interface.** Add an `IVerticalCraft` interface for `HopperCraft`
  and `RocketCraft`, which share about 45 members. That removes about 54
  `Machine.Hopper` branches in WorldSetup.
- **Verb buttons.** Move the verb-button priority rules out of
  `PhaseHud.cs` (around lines 1327–2047) into
  `SimCore.Controls.Verbs.For(situation)`, with tests.
- **Then split the files mechanically:**
  - WorldSetup, using its own section headers: WorldBuilder, HopController,
    AttitudeOwnership, LaunchPad, MissionDirector, LandingWatch, HudAssembly;
  - MoonExplorer, into timeline, Eagle and docking, and the craft seam;
  - MenuBar, into one item provider per menu.

### B2. Flight model: make stalls and spins real

1. **Full rotational equations.** Add the missing `−ω×(Iω)` term and the
   complete propeller ω×H (`ModernFlightModel.cs` around lines 2912 and
   2991–3008). Then relax the rate guards, which clip real spin rates.
   Sources: Stevens & Lewis ch. 1, Etkin & Reid.
2. **Post-stall aerodynamics.**
   - Lift and drag through high angle of attack. Today drag *falls* past the
     stall because lift collapses (around lines 2660–2684), and the 25 in the
     lift fall-off is uncited.
   - Left and right wing strips, each with its own local angle of attack
     including the roll-rate term, so wing drop and autorotation come out of
     the physics.
   - Stall hysteresis.

   Sources: Sheldahl & Klimas SAND80-2114, Hoerner, NASA Langley's
   general-aviation stall/spin programme.
3. **Side force from sideslip** (currently 0 at around line 2734), and the
   cross-derivatives: roll due to yaw rate, yaw due to roll rate, and adverse
   aileron yaw. Drive the slip ball from sideways acceleration, not sideslip
   angle. Sources: Roskam, DATCOM.
4. **A flight-test card suite** that flies `ModernFlightModel.Step` and compares
   with the handbooks: C172S ground roll 960 ft, climb 730 ft/min, glide 68
   KIAS, stall speeds, cruise true airspeed.
   - This replaces the round-trip tests: the climb test reads back
     `BestClimbRateMs`, the stall and top-speed tests re-derive their own
     inputs, and the take-off check allows under 900 m against 293 m.
   - Also add a timestep-convergence test (0.02 s against 0.005 s).
5. **Later:** a propeller map (NACA TR-640), a tyre friction circle with surface
   grip, a pitching-moment build-up with CG so loading changes the handling, and
   MIL-F-8785C turbulence above 2,000 ft.

### B3. Apollo: fly the trip in three dimensions

1. **Carry the Moon in 3-D.** `Cislunar` puts it at Y = 0. Use
   `Lunar.EclipticAt` latitude, set the parking orbit's plane from the real
   launch azimuth and time, give TLI and LOI-1 out-of-plane components, and
   solve Apollo 13's PC+2 in magnitude, pitch and yaw. The target is MSC-02680
   table 4-III: 860.5 ft/s, 263.8 s. The known misses are PC+2 431 m/s against
   262, Apollo 12's Moon jumping 42° at TLI, and an impulsive 19 m/s plane
   change Apollo never flew.
2. **Rendezvous.** Replace the Hohmann transfer with a Lambert solution for TPI
   through 130°, on the 26.6° line of sight, and add CDH and the midcourses.
   Today it takes 67 minutes against 43 and ends 646 m away against 152. Source:
   Apollo 11 Mission Report table 3-I.
3. **Descent.**
   - Use Klumpp's P63/P64 guidance (MIT/CSDL R-695). The braking range of
     496 km was tuned, which breaks the project's rule, and high gate arrives at
     53 m/s against 46.
   - Give each mission its own LM mass: today Eagle, Intrepid and Aquarius all
     fly J-class masses.
4. **Smaller items:**
   - the Meeus ch. 47 lunar series, ΔT (the code treats UTC as TT, about 40 s
     out in 1969) and a solar term;
   - a low-degree GRAIL gravity field, with LOI-2 targeted elliptical;
   - TEI aimed at the recovery area (today it splashes 851 nmi away);
   - the stack drawn 95 m off its flown point;
   - the S-IVB burns 106 s against 147.
5. **Refresh the stale docs.** `Docs/ApolloReview.md` and `MissionChecklist.md`
   still list TLI, MCC and the return as missing, but they are built.

### B4. Terrain and world

1. Add one `TerrainStreamer.DrawnHeightAt(x, z)` and use it for rocks, roads,
   trees and buildings. Each guesses the drawn surface differently today, and
   roads assume mesh steps of `{64, 256}`, which match no real ring.
2. **Geomorphing** between rings, normals that match at ring boundaries,
   `vertsPerSide` 129, and shadows from rings 0–3.
3. **Earth curvature in the vertex shader** (terrain and water). Earth is flat
   today, and Manchester is about 1.15 km "too high".
4. **Load time.**
   - The land-cover texture is built on the main thread (about 75 MB of garbage
     for Heathrow–Manchester).
   - The globe maps go through a JPEG decode and a `Compress`; pre-bake BC1.
   - `GlobalDem.Load` runs synchronously.
   - Pool the mesh buffers.
5. **Weather.**
   - Clouds that drift with the wind.
   - Ridge lift from the terrain gradient, testable in SimCore.
   - Winds aloft above 600 m.
6. **Airfields from data.** Add `aeroway=runway` to the Overpass query in
   `RegionImporter` to bring in real runway thresholds.

### B5. Confidence: Unity in CI

1. A GameCI job on Linux that compiles all four assemblies. Today
   `check-unity-build.sh` is Mac-only and defines only
   `UNITY_EDITOR;UNITY_STANDALONE_OSX`.
2. A PlayMode assembly that references Game, loads `Modern.unity`, cycles every
   world, every craft, every pause page and `BackToDefaults`, and asserts
   `LogAssert.NoUnexpectedReceived()`.
3. A source-reading test that fails on any `Shader.Find` name that won't be in
   a player build, and bans `"Standard"`, which renders pink under URP. This is
   release blocker A1 in `ReleaseStudy.md`.
4. Allowlist `ScreenChainTests` exemptions per line, not per whole file (six
   files, about 10,200 lines, are exempt).
5. Commit one fresh flight log so `ReplayTests` stops being ignored.
6. Add a test that the key tables in Manual §1.2 equal `Bindings`, and a CI step
   that fails when `manual.html` is stale.

### B6. Player experience

1. **Gamepad and touch.** A pad can open the pause menu but can't navigate it.
   Give the UI Toolkit surfaces focus and navigation, let pad controls be
   rebound, and add long-press tooltips on touch.
2. **Onboarding.**
   - A first-launch tour of three or four cards.
   - A "Getting started" item in Help.
   - A shipped player guide. Manual.md is 23,500 lines, and only about 9% of it
     is for players.
   - Split it into Player Guide, Reference, Building and a Notes journal.
3. **Accessibility.**
   - Move Good/Watch/Over into `SimCore.Controls.Palette`.
   - Test the colours against WCAG contrast and a colour-vision simulation
     (Machado 2009).
   - Pair every state with a word or shape as well as a colour.
4. **UI Toolkit migration order:** pause pages → messages → picker → lesson
   (with the phone bottom sheet) → crash/splash → MenuBar last.
5. **The ILS work from the last session.**
   - Move the ILS indicator to the bottom row, give it a visible hide control,
     and add a NAV radio: a localiser frequency and Morse identifier per ILS,
     tuned by the player, with auto-tune kept as an option.
   - Use real frequencies only where the UK AIP confirms them; otherwise label
     them invented.

Start with Part A, commit it, then do B1 (it makes everything after it
testable), then B2. Stop and ask me before any change that alters how an
aircraft or mission *feels* to fly in a way a player would notice, and tell me
what they will notice.
