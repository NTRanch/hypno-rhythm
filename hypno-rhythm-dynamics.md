# Hypno_Rhythm — dynamics spec (S3, COIN-DYNAMIC class)

Lane: NTRanch / nsfw. Clean-room record: behavior and tuning
constants only — no source text, art, audio, or code transfers.
Sibling records: `hypno-rhythm.md` (intake best-practices note),
`hypno-rhythm-animation.md` (S2b animation study).

## 0. Header

| Field | Value |
|---|---|
| Title / build | *HypnoRhythm* — `app.info`: RoukieStudio / HypnoRhythm (primo build dir `Hypno_Rhythm`) |
| Engine | Unity 6000.0.32f1, Mono (CENSUS.md §1; 188 files, 213,584,843 B — smallest complete Unity Mono title in the tree) |
| Original language | UNKNOWN — not recorded at intake; no shipped text was read for this spec (mechanics only, per Lane D rule) |
| Study date | 2026-11-08 |
| Evidence home (primo) | `~/.local/share/pax/hypno-rhythm-20261108/il/` |
| Primary artifact | `Assembly-CSharp.dll` (70,144 B, sha256 `c5d7b9c438862b750f905ada5333d5ca84cb2311464f5abe1c12c378433ea654`) → ikdasm IL export, **two independent runs byte-identical** (13,183 lines, sha256 `78dbdba1bfb2f8cd93b7825bb479af0e6aed560aece16ff1511eeeef960b90b0`, rc 0 both) — the Magic Coin gate pattern (COIN-DYNAMIC.md §5) |
| Secondary artifacts | `Assembly-CSharp-firstpass.dll` ikdasm export (9,422 lines — DOTween module shims only, no game logic); `DOTween.dll` ikdasm export (sha256 of export `8c10531dd6fef0ee241b8a83b07aa7cbc8c9af20c0694939292ea897fe4de4b0`) for the Ease enum literals in §6 |
| Census | CENSUS.md §1 row; MonoBehaviour census in `~/workspace/fathom-unity-mono-pilot-20261103/PILOT-REPORT.md` §3 (711 MonoBehaviours, 108 distinct script classes, 3,019 MonoScripts, 0 anomalies — this title was the fathom pilot) |
| Spec grade overall | **MEASURED** for every code-resident constant and rule below; scene/ScriptableObject-resident values are named UNKNOWN in §9, never filled |

Class inventory of the game assembly (39 classes, IL export):
the rhythm core is `RhythmManager`, `GameManager`,
`AudioManager`, `ObjectManager`, `CharacterController`,
`EffectMaager` (sic), `ShakeCameraFeel`, `GoodTriggerHandler`,
`PerfectTriggerHandler`, `BeatMapData`, `SongData`,
`LevelDataSO`; the presentation shell is a shader/glitch stack
(`BWEffect`, seven `ShaderEffect_*`, `GlitchEffect`,
`Kino.AnalogGlitch`, `Kino.DigitalGlitch`, `Reaktion.*Motion`,
`EyeSiluetEffect`, `SimpleSpectrum`) plus `MenuManager`,
`GlobalManager`, `LoadSceneManagement`, `ButtonHandler`,
`SoundFileOpener`, `OutputVolume`. Managed neighbours:
DOTween, UniTask, MoreMountains.Tools, Lofelt.NiceVibrations,
Unity 2D Animation/IK runtimes (presence in `Managed/` listing,
evidence home §0).

## 1. The loop, as behavior

One question, repeated: **is the sweeper inside the zone when
you tap?**

1. A target object (tagged `Player`) sweeps back and forth
   between two points, one traversal segment per beat, on an
   arced path (`ObjectManager`, §4). Two fixed trigger zones
   sit on its path: a wide **Good** zone and a narrow
   **Perfect** zone nested in it (physics overlap flags, §3).
2. The player has exactly one input: a tap (Space — the only
   input call in the game assembly; §3). A tap is judged
   instantly against whichever zone flag is currently set:
   Perfect if the Perfect flag is up, else Good if the Good
   flag is up, else **Miss**.
3. Hits build combo and score (§5); a Miss resets combo to 0.
   There is no per-note bookkeeping: the game counts *taps*,
   not notes (§5.3) — an untapped beat costs nothing, a stray
   tap costs a Miss.
4. Progress is the audio clock over the song length (§7); at
   progress ≥ 1 the game loads the `Menu` scene. Per-song
   parameters (BPM, beat map) come from a ScriptableObject
   (§2); levels are selected from the menu — there is no
   in-song difficulty ramp in code (§8).

What the dynamic *is*, mechanically: **a spatial timing
judgment (physics overlap) wearing a temporal costume** — the
"timing window" is a place, not a duration. Everything else
(character dance, camera zooms, glitch escalation) is
presentation driven off the same beat grid (animation note).

## 2. Song / level data model (MEASURED structure)

- `SongData` (serializable): `songName` (string), `songID`
  (string), `BPM` (int), `songDuration` (float), `battleData`
  (TextAsset). Wrapped by `LevelDataSO` (a ScriptableObject
  holding one `SongData` as field `data`).
  — IL: `SongData`/`LevelDataSO` class definitions.
- **Beat map format**: JSON, parsed by
  `JsonUtility.FromJson<BeatMapData>` where
  `BeatMapData { float bpm; int[] beats }` — a flat list of
  beat indices. `GameManager::GetBeatMap` fails soft: null /
  empty / unparseable / beat-less input logs an error and
  yields an empty list (no crash path).
  — IL: `GameManager::GetBeatMap` IL_0042–00a8.
- `GameManager::Awake` loads the map from
  `levelData.data.battleData`; `Start` computes
  **songLength = beats[last] × (60 / SongData.BPM)** seconds.
  — IL: `GameManager::Start` IL_0045–0079.
- The shipped songs' actual BPM values, beat lists, and level
  count are data-resident (ScriptableObject/scene payloads in
  typetree-less SerializedFiles) — **UNKNOWN**, §9. The
  `BeatMapData.bpm` field is parsed but the code paths
  inspected take BPM from `SongData.BPM` (int) — whether the
  JSON's own bpm is ever used is UNKNOWN (no consumer found in
  the inspected methods; not a proof none exists).

## 3. Input and judgment (MEASURED)

- **Input**: `GameManager::Update` polls
  `Input.GetKeyDown(KeyCode 32)` — Space — and calls
  `OnPlayerTap()`. It is the *only* input poll in
  Assembly-CSharp (grep over the IL export: one `GetKeyDown`,
  zero `GetMouseButton`/touch calls). Button UI exists
  (`ButtonHandler`, 10 instances per the pilot census) but no
  IL path from a button to `OnPlayerTap` was found — **how
  (or whether) mouse/touch taps judge is UNKNOWN** (§9).
- **Zone flags**: `GoodTriggerHandler` / `PerfectTriggerHandler`
  are trigger colliders; `OnTriggerEnter2D` with tag `Player`
  sets `GameManager.goodActive` / `perfectActive`, exit
  clears. No timing arithmetic exists in either handler
  (25-byte methods, flag set/clear only).
  — IL: both handler classes, complete.
- **Judgment order** (`GameManager::OnPlayerTap`, 99 bytes,
  complete): perfectActive → `OnHit(Perfect)`; else
  goodActive → `OnHit(Good)`; else `OnMiss()`. Each branch
  also fires its feedback (hit particle, camera shake —
  animation note §5/§6).
- **Window widths**: the zones are scene colliders; their
  sizes and positions — hence the effective windows in
  seconds (zone width ÷ sweeper speed) — are scene-resident:
  **UNKNOWN** from the IL route (§9). The *shape* of the
  window is DERIVED: window_seconds = zone_width /
  (path_length × curve_slope(t) / secPerBeat) — because the
  sweeper's speed is curve-modulated (§4), the same zone is a
  wider time window where the curve runs slow and narrower
  where it runs fast. A spatial window over an eased sweeper
  is a *non-uniform* temporal window by construction.

## 4. The sweeper (MEASURED)

`ObjectManager` (the moving target's driver):

- Clock: `AudioManager.GetSongTime()` — the **audio clock**
  (`AudioSource.time` of the current BGM source; §7), plus a
  public `songOffset` field applied to it (offset semantics
  MEASURED as "applied in Update"; its shipped value is
  scene-resident, UNKNOWN).
- Segments: `beatTimes[i] = beatSteps[i] × secPerBeat`, with
  `secPerBeat = 60 / BPM`; `beatSteps` is this object's own
  beat list (scene/SO-resident values UNKNOWN).
- Motion: within a segment, `t01 =
  InverseLerp(beatTimes[seg], beatTimes[seg+1], songTime)`,
  eased by a serialized `AnimationCurve moveCurve`
  (code default `EaseInOut(0,0,1,1)` in the .ctor), position =
  arc between `leftPoint` and `rightPoint` with the midpoint
  raised by `arcHeight` (code default 1; `arcDown` default
  true flips it) — a quadratic-arc sweep, ping-pong
  (`moveToRight` flips each segment). A `LineRenderer` draws
  the arc for the player.
  — IL: `ObjectManager::Update` (InverseLerp IL_008e,
  `moveCurve.Evaluate` IL_009b, arc midpoint ×0.5 IL_00e9);
  .ctor defaults.

## 5. Scoring (MEASURED, complete)

### 5.1 Base scores and combo

Code defaults from `GameManager::.ctor` (field initializers;
[SerializeField] scene overrides, if any, are UNKNOWN — §9):

| Constant | Value | IL |
|---|---|---|
| `perfectScore` | **300** | .ctor IL_008e (`ldc.i4 0x12c`) |
| `goodScore` | **100** | .ctor IL_0099 |
| `comboMultiplier` | **1.12** | .ctor IL_00a1 |
| accuracy initial | 100 (%) | .ctor IL_0083 |

`AddScore(base, increaseCombo=true)`:
1. combo += 1 (before scoring — the first hit of a streak
   already scores at combo 1); maxCombo tracks the peak.
2. bonus factor = `Max(0, (combo − 1) × (comboMultiplier − 1))`
   = (combo − 1) × 0.12 at the default multiplier.
3. points = `RoundToInt(base × (1 + bonus factor))`.
   — IL: `GameManager::AddScore` IL_002b–005b, complete.

So at default tuning a Perfect is worth 300 × (1 + 0.12·(combo−1)):
300 at combo 1, 336 at combo 2, 3,300 at combo 101 — linear,
uncapped in code (any cap would be data-side; none found).

### 5.2 Accuracy

`CalculateAccuracy` (complete, IL_0000–0068):

**accuracy % = 100 × (300·perfectHits + 100·goodHits) / (300 · totalNotes)**, clamped [0, 100]; 100 when totalNotes = 0.

The weights are literals (300 / 100) that happen to equal the
default base scores — the formula is score-shaped, not
count-shaped: a Good is worth exactly ⅓ of a Perfect.

### 5.3 What counts as a note (MEASURED, and it is the design)

`totalNotes` increments **once per tap outcome** — in
`OnHit` (both grades) and in `OnMiss`. Nothing else in the
inspected code increments it, and no code path registers a
miss for an untapped beat (the only `OnMiss` caller is
`OnPlayerTap`'s else-branch). Consequences, stated as
behavior:

- Accuracy is *per-tap precision*, not *per-note completion*.
  Playing only the beats you are sure of and skipping the
  rest is a dominant strategy for accuracy (score still
  rewards tapping, via combo).
- Combo likewise only breaks on a *tapped* miss.

This is either a deliberate low-punishment design or a
simplification the shipped game lives with; the remake
adopts it knowingly or changes it knowingly (§11) — it must
not be "fixed" silently in porting, because it *is* the
difficulty curve.

## 6. Feedback timing constants (MEASURED code defaults)

From `GameManager::.ctor`; Ease names resolved against the
shipped `DOTween.dll`'s own enum literals (DOTween IL export
lines 4183–4210), not from memory:

| Field | Value | Ease |
|---|---|---|
| effectShowDuration | 0.3 s | effectShowEase = 27 = **OutBack** |
| effectHoldDuration | 1.0 s | — |
| effectHideDuration | 0.5 s | effectHideEase = 5 = **InQuad** |
| scoreAnimDuration | 0.3 s | scoreEase = 12 = **OutQuart** |
| comboAnimDuration | 0.2 s | comboEase = 27 = **OutBack** |
| accuracyAnimDuration | 0.25 s | accuracyEase = 9 = **OutCubic** |
| scaleMultiplier (UI punch) | 1.15 | — |
| indicator punch (`AnimateIndicator` call sites) | scale 1.2 over 0.3 s, both grades | — |

Hit effects are per-grade GameObject arrays cycled by
`ShowHitEffect` with the show/hold/hide envelope above; the
indicator's color tween runs at half the punch duration
(`AnimateIndicator` IL_00dc: duration × 0.5).

## 7. Audio architecture and the sync question (MEASURED)

`AudioManager`: dual BGM sources (A/B) with UniTask async
crossfade (`CrossfadeBGMAsync`), a dedicated SFX source,
name-keyed clip tables (`bgmClips` / `sfxClips` of
`{name, clip}`). Volume code defaults: master 1.0, BGM 0.5,
SFX 0.5 (`.ctor` IL_0001–001c); effective source volume =
bus × master, clamped [0,1] on set. `PlayBGM(name, BPM,
fadeDuration)` sets the rhythm clock from its BPM argument
(below) then crossfades; `GameManager::Start` calls it with
fade **1.0 s** and the level's `SongData.BPM`. Hit SFX name:
`"Hit"` (both grades — same clip name; `OnHit` IL_001a/0063).

**Three clocks, loosely coupled — the central finding:**

| Clock | Source | Consumers |
|---|---|---|
| Frame clock | `RhythmManager.Update`: `timer += Time.deltaTime`; beat fires when timer ≥ 60/BPM, one beat max per frame (single subtraction, no catch-up loop) | beat-event listeners: camera zoom effects (`EffectMaager`), menu/ambience |
| Audio clock | `AudioSource.time` via `GetSongTime()` (0 when not playing) | sweeper (§4), character dance (animation note §3), progress/end (§1) |
| Tween clock | DOTween (default: scaled game time) | all feedback/UI/character tweens |

Couplings and gaps, all MEASURED from the IL:

- `RhythmManager` BPM default **120** (.ctor IL_0001);
  `SetBPM` clamps ≥ 1, recomputes interval = 60/BPM, and
  **resets beatCount and timer to 0**. Its only caller is
  `AudioManager.PlayBGM` — the frame clock restarts when a
  song is *requested*, while the audio itself ramps in over
  the crossfade (1.0 s from GameManager). The frame-clock
  beat grid and the audible downbeat are therefore offset
  by up to the fade duration at song start, by construction.
- **No DSP scheduling anywhere**: zero `AudioSettings.dspTime`
  / `PlayScheduled` references in the game IL (grep over the
  export). Nothing in the game ever aligns an event to the
  audio hardware clock.
- Judgment does not use either clock directly: it reads
  physics flags set by the sweeper, and the sweeper rides
  the audio clock — so *judgment inherits audio-clock
  accuracy through the sweeper's position*, while the
  beat-*event* system (camera, listeners) rides the frame
  clock and can drift from the audio by frame quantization
  plus the start offset above. At 120 BPM a beat is 0.5 s;
  a 60 fps frame clock quantizes beat events to ±8.3 ms and
  drops the remainder of any long frame's beats entirely
  (one subtraction per Update).
- Progress/end DO use the audio clock (songTime/songLength),
  so song duration is honest even where beat events are not.

**Audio-sync assumptions the original makes** (stated as its
design, not endorsed): (a) the audio clock is truth for
anything the player judges; (b) frame-clock drift is
acceptable for anything merely decorative; (c) a fade-in at
song start is worth more than a sample-aligned first beat.
The Torogao spike (§10) exists to test whether Torogao can
hold a stricter line than (a)–(c), not to re-prove them.

## 8. Difficulty progression (MEASURED shape, data-resident values)

- **Across songs**: per-level `SongData.BPM` + beat-map
  density (`beatSteps` spacing) + zone geometry (scene). All
  values UNKNOWN (§9); the *levers* are exactly these three —
  no other difficulty code exists.
- **Within a song**: none in code. No mid-song `SetBPM`
  caller, no window shrink, no speed ramp — tempo and
  density are flat from first beat to last (grep + caller
  analysis over the IL export).
- **Escalation theatre** (presentation, not difficulty):
  beat-indexed camera zooms and effects via
  `EffectMaager.eventFeedbackEffects` — a list of
  `{beat, UnityEvent}` fired when the frame-clock beat count
  equals the entry's beat (animation note §4).
- **Combo as soft ramp**: score pressure grows linearly with
  streak (§5.1) while judgment never tightens — difficulty
  is front-loaded into song choice, not progression.

## 9. REVIEW / UNKNOWN register

| # | Unknown | What was tried | Route that closes it |
|---|---|---|---|
| U1 | Shipped BPM per song, beat lists, level/song count, `songOffset`, zone sizes/positions, sweeper endpoints, serialized overrides of every §5/§6 default | IL pass (complete for code); strings probe over `sharedassets0.assets` + `resources.assets` found no beatmap JSON | DLL-metadata (AssetRipper-class) rung: reconstruct MonoBehaviour/SO field values from the SerializedFiles — the pilot's named next rung (PILOT-REPORT §5) |
| U2 | Whether mouse/touch UI can judge a tap (Space is the only IL input) | Full-assembly input grep | Runtime observation (REVIEW-grade) or scene wiring extraction via U1's rung |
| U3 | `BeatMapData.bpm` — parsed, no consumer found | Caller/field-use grep over inspected methods | Exhaustive field-use pass over all 39 classes |
| U4 | Crossfade curve and exact first-audible-sample time vs frame-clock zero (§7's start offset, bounded ≤ fade 1.0 s but not measured) | IL read of PlayBGM/Crossfade state machines (structure MEASURED; per-frame volume stepping is UniTask-yield driven) | Runtime measurement on the original (Owner-gated play session) or the §10 spike's comparative harness |
| U5 | Haptics: Lofelt.NiceVibrations ships in `Managed/` — whether/when the game calls it | Assembly presence only; no call search run against the game IL for Lofelt types | Targeted IL grep for `Lofelt` references (cheap; not run — noted honestly) |
| U6 | `OutputVolume` / `SoundFileOpener` roles (audio import? volume UI?) | Class names + fields only | Method-level read (same IL artifact) |

## 10. The Torogao audio-sync spike — requirements (UNPROVEN, not assumed)

Torogao's fitness for a rhythm loop is **unproven**. This
spec does not port the loop; it defines what the spike must
measure before any port is commissioned. The original's own
weaknesses (§7) set the bar the spike must *beat*, not meet:

1. **Clock authority**: one clock for judgment — the audio
   output clock (DSP-time class), with the sweeper's position
   a pure function of it. Measure: position error of the
   judged object vs the audio clock over a full song at
   120 BPM and at 180 BPM, on Linux and on Android (S6b),
   reported as max/RMS ms.
2. **Start alignment**: time from "song requested" to first
   audible sample, and the offset between the beat grid's
   zero and that sample. The original ships ≤ 1.0 s of
   fade-class slop (U4); the spike's target must be a stated
   number, not an adjective — proposed gate: grid zero
   within one output buffer of the first sample.
3. **Judgment equivalence**: re-implement §3/§5 as pure
   functions (zone flags from sweeper position; per-tap
   scoring) and show the trajectory hash matches a reference
   implementation on shared fixtures (D-212 cross-platform
   determinism gate) at both target tempos.
4. **Long-frame behavior**: inject 100 ms and 250 ms frame
   stalls; the original drops beats (frame clock) — the
   spike must show the audio-clock systems lose none.
5. **Latency budget**: input-to-judgment latency on both
   targets (tap timestamp vs audio clock at judgment),
   measured, since Android audio latency is the known
   dual-target risk.
6. **Fail condition**: if (1) or (2) cannot be met with
   Torogao's current audio path, the spike reports the gap
   as a Torogao engine requirement (scheduled playback /
   DSP-clock exposure) — it does not quietly adopt the
   original's three-clock looseness.

## 11. Best practices — what Torogao adopts, adapts, avoids

(Folded here per commission; the intake-level practices are
in `hypno-rhythm.md` and stand.)

1. **ADOPT — judgment as a pure function of position.**
   The zone-flag design makes judgment trivially testable
   and deterministic: position in, verdict out. Torogao's
   version keeps the purity and drops the physics engine:
   zone membership computed analytically from the sweeper
   function (§3's shape, §10.3's hash gate).
2. **ADOPT — one input, honestly tuned.** Space-only, one
   question. The estate's one-verb pattern (Magic Coin)
   again: small loop, tuning *is* the design.
3. **ADOPT — fail-soft data loading.** `GetBeatMap`'s
   log-and-empty-list on every bad input (§2) is the right
   posture for user/data-driven content; mirror it in the
   RON loader family (typed refusal + named log, never a
   panic).
4. **ADAPT — per-tap accounting.** §5.3's tap-counted
   accuracy is a real design choice (low punishment,
   dominant skip strategy). A remake must choose it
   deliberately per title, not inherit it by accident —
   record the choice in the title's S4 data file.
5. **AVOID — three clocks.** §7 is the cautionary tale:
   frame-clock beat events + audio-clock judgment objects +
   tween clock, aligned by hope and a fade-in. Torogao:
   one authoritative clock per system class, named in the
   spec before code (§10.1).
6. **AVOID — beat loss on long frames.** One subtraction
   per Update silently eats beats under load. Any
   accumulator in Torogao carries a catch-up loop with a
   stated bound.
7. **ADOPT — data as JSON-shaped tables.** The beat map is
   `{bpm, beats[]}` — the S4 RON table for a remake is a
   transliteration, not a redesign (schema-versioned per
   PIPELINE §1 S4).
8. **NOTE — serialized defaults are a trap for readers.**
   Every §5/§6 number is a *code default*; Unity scenes can
   override silently. Estate specs must keep the two grades
   separate forever (this spec's §9 discipline), because a
   remake tuned to code defaults may not match the shipped
   feel — the U1 rung is what closes that gap.

## Provenance

- Fathom routes: typetree-independent MonoBehaviour census
  (pilot title, PILOT-REPORT §3); ikdasm IL route per
  COIN-DYNAMIC.md §5 precedent.
- Extraction commands of record (primo, light, nice'd):
  `ikdasm Assembly-CSharp.dll` ×2 (byte-identical, §0);
  `ikdasm Assembly-CSharp-firstpass.dll`; `ikdasm DOTween.dll`;
  `strings` probe (§9 U1). Artifacts in the evidence home.
- Bloch candidate: `../bloch/hypno-rhythm-three-clocks.md`.
- Skill-trust: recorded at close (fathom-game-extract,
  game-intake).
