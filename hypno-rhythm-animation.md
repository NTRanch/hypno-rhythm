# ANIMATION STUDY NOTE — Hypno_Rhythm (S2b)

Per-title animation note for the Lane D programme (PIPELINE.md
stage **S2b**, D-1094 research-study framing), on the
ANIMATION-STUDY-TEMPLATE shape.

**Clean-room law, restated for animation:** this note records
*techniques and measurements* — rig shapes, state graphs,
timing numbers, easing shapes described as parameters — never
copied assets, curve files, keyframe dumps shipped verbatim,
or code. What crosses into the remake is a principle Torogao
re-implements in its own terms (§9). Verbatim material stays
RE-side, cited by evidence pointer only.

**Grade vocabulary**: MEASURED (read from source/data — here,
the ikdasm IL export) > DERIVED (computed from measured facts,
derivation shown) > REVIEW (observed at runtime or by
impression only; never presented as measured).

---

## 0. Header

| Field | Value |
|---|---|
| Title / version(s) studied | *HypnoRhythm* (RoukieStudio), the primo build — one version held |
| Engine + version | Unity 6000.0.32f1, Mono (CENSUS.md §1) |
| Original language | UNKNOWN — not recorded at intake; no shipped text read (mechanics/technique only) |
| Study date / student | 2026-11-08 / Lane D queue #3 commission |
| Dynamics spec (sibling) | `games/hypno-rhythm-dynamics.md` |
| Census | `CENSUS.md §1` / PILOT-REPORT.md §3 (fathom pilot title) |
| Evidence home | primo `~/.local/share/pax/hypno-rhythm-20261108/il/` — `run1.il` (Assembly-CSharp IL, two runs byte-identical, sha256 `78dbdba1…`), `dotween.il` (DOTween IL, Ease literals at lines 4183–4210), `firstpass.il` |
| Note grade overall | MEASURED for code-resident technique; scene-resident parameters are UNKNOWN per §10 — overall **DERIVED** (the shipped *feel* depends on scene overrides this route cannot read) |

## 1. Animation architecture at a glance

Almost nothing in this game is keyframed. The character, the
judged sweeper, the camera, and the UI are all **procedural
DOTween/Update motion driven off two clocks**: the sweeper and
the character dance poll the **audio clock**
(`AudioSource.time`), while camera-zoom events ride a
**frame-clock** beat counter, and every tween itself runs on
DOTween's game-time clock (dynamics spec §7 has the full
three-clock analysis). Animation *data* lives in code
constants + serialized scene fields — there are no
AnimationClips in the game assembly at all, and no Animator
controller is referenced by any game class.

- Summary: a sprite-part character (head / body / eye /
  pupil transforms) pulses on each beat with tween durations
  computed as fractions of the current beat segment; a target
  object sweeps an eased arc between two points, one segment
  per beat; the camera zooms at authored beat indices and
  shakes on judgment events; a shader/glitch stack (BW,
  CRT, scanner, glitch) escalates the image. Everything is
  code — the "clips" are tween recipes (§4).
- Evaluation rate + source: per-frame `Update` polls in
  `ObjectManager`, `CharacterController`, `RhythmManager`,
  `GameManager` — **Evidence:** `run1.il`, those classes'
  `Update` methods.
- Animation data location: serialized fields on scene
  MonoBehaviours + `.ctor` code defaults (the readable half)
  — **Evidence:** `run1.il` .ctor bodies; census counts in
  PILOT-REPORT §3.

## 2. Rig / puppet structure

### 2a. Skeletal / puppet rigs

| Field | Entry |
|---|---|
| Rig kind + part count | **Transform-hierarchy sprite puppet**, 5 driven parts in code: `headTransform`, `bodyTransform`, `eyeTransform`, `pupilTransform`, `cameraTransform` (fields of `CharacterController`). No bone/deformer system is driven by game code. |
| Hierarchy shape | head / body / eye / pupil as separate scene transforms under the character root (parenting is scene-resident — UNKNOWN); pupil is position-driven relative to its own origin, eyes scale-driven. |
| Parameter set | Per-part: amplitude (height / range / scale factor), duration multiplier against the beat segment, one DOTween Ease each — the table in §5 is the rig's full control vocabulary as coded. |
| Mesh/deformer notes | None in code. Unity's 2D Animation + 2D IK runtime DLLs ship in `Managed/`, but **no game class references them** (IL grep) — whether scene rigs use them is UNKNOWN (§10). |
| Physics on the rig | None — no springs, pendulums, or verlet in game code. Secondary motion is faked with eased counter-tweens (§6). |

- **Evidence:** `run1.il` `CharacterController` fields +
  `TriggerBeatAnimation`; `Managed/` listing in dynamics §0.

### 2b. Sprite / frame rigs

| Field | Entry |
|---|---|
| Sheet organisation | UNKNOWN — sprite assets are scene/bundle-resident (typetree-less census counts objects, not sheets). |
| Swap model | The judgment indicator is a `SpriteRenderer` whose **color** is tweened per verdict (normal / perfect / good / miss `Color32` fields on `GameManager`) — state shown by tint + scale punch, not frame swap, in code. |
| Expression system | Eyes are a *beat instrument*, not an expression set: eye Y-scale pulses ×0.8 per beat segment and pupils shuttle left/right (§5). No face-set/overlay expression system exists in game code. |

- **Evidence:** `run1.il` `GameManager::AnimateIndicator`,
  `CharacterController::TriggerBeatAnimation` IL_00ef–01d7.

### 2c. 3D rigs

N/A — 2D title; no 3D rig classes or model census applies.

## 3. Animation state machine

There is no Animator graph. The motion "machine" is three
small code machines running in parallel:

| State | Entered from | Trigger / condition | Exits to | Evidence |
|---|---|---|---|---|
| Sweeper segment *k* (moving L→R or R→L) | segment *k−1* | audio clock crosses `beatTimes[k]` | segment *k+1* (direction flips) | `ObjectManager::Update` |
| Character pulse (per part) | rest pose | audio clock crosses `beatTimes[segIndex]` → `TriggerBeatAnimation()` | auto: tween up over ½ duration, return tween on complete | `CharacterController::Update` + `TriggerBeatAnimation` |
| Camera zoom Normal / Medium / Close | any | frame-clock beat == an authored `EventFeedbackEffect.beat`, whose UnityEvent invokes the zoom method | next authored beat event | `EffectMaager::OnRhythmEvent`, `CameraZoom*` |
| Camera shake (trauma) | 0 | judgment events add trauma (Perfect .3 / Good .2 / Miss .15 / combo milestone .5 / boss drop .8, code defaults) | decays at 1.5/s; offset ∝ trauma² | `ShakeCameraFeel` .ctor + event methods |
| Hit-effect envelope | hidden | any judged tap | show (0.3 s) → hold (1.0 s) → hide (0.5 s), then inactive | `GameManager::ShowHitEffect` + .ctor |

- Machine owner: code enums/fields only — `segIndex` +
  `beatTimes[]` (audio-clock machines), `RhythmManager.
  beatCount` (frame-clock machine). — **Evidence:** `run1.il`
  as cited per row.
- Transition mechanics: sweeper motion is continuous
  (curve-evaluated, no transitions); character parts use
  paired DOTween tweens (out over duration×0.5, back on
  `OnComplete`); camera zooms are single DOTween moves of
  position + orthographic size over 0.5 s (code default).
- Diagram (text form):
  `audio clock ─┬→ sweeper segments (ping-pong arc)`
  `             ├→ character pulses (4 parts + camera drift)`
  `             └→ progress / song end`
  `frame clock ──→ beat count ──→ authored camera/effect events`
  `tap events ───→ indicator punch, hit effects, trauma shake, UI counters`

## 4. Clip inventory

No AnimationClips exist in the game assembly; the inventory
is the **tween recipe inventory** — each row is one reusable
motion the game actually performs. Durations are in beat
segments (seg = the current `beatTimes` interval, §5) or
seconds where coded so.

| Recipe | Used by (state) | Duration | Loop? | Evidence |
|---|---|---|---|---|
| Head bob (Y +0.2, up & back) | character pulse | 0.3 seg (½ up, ½ back) | per beat | `TriggerBeatAnimation` IL_0054–009a |
| Body jump (Y +0.15) | character pulse | 0.25 seg | per beat | IL_00a8–00ee |
| Eye scale (Y ×0.8) | character pulse | 0.2 seg | per beat | IL_00fc–015f |
| Pupil shuttle (X ±0.1, alternating) | character pulse | 0.4 seg | per beat, direction flips | IL_016d–01d7 |
| Camera drift (range 0.15) | character pulse | 0.4 seg | per beat | `CharacterController` fields/.ctor |
| Sweeper arc traversal | sweeper segment | 1 segment | continuous ping-pong | `ObjectManager::Update` |
| Indicator punch (scale 1.2 + tint) | judgment | 0.3 s (tint 0.15 s) | per tap | `GameManager::OnHit` IL_0038/0081 |
| Hit-effect envelope | judgment | 0.3 + 1.0 + 0.5 s | per tap | dynamics §6 |
| Score/combo/accuracy counters | UI | 0.3 / 0.2 / 0.25 s + scale punch 1.15 | per score change | `GameManager::Animate*` |
| Camera zoom (3 sizes: 5 / 4 / 3 ortho) | authored beat events | 0.5 s | per event | `EffectMaager` .ctor + `CameraZoom*` |
| BGM crossfade (presentation-adjacent) | song start | 1.0 s from GameManager | per song | dynamics §7 |

- Total clip count: 0 AnimationClips in code; 11 procedural
  recipes above. Scene-side clips, if any, are UNKNOWN (§10).
- Naming pattern: recipes are method-named by body part +
  verb (`headBob`, `bodyJump`, `eyeScale`, `pupilMove`,
  `cameraMove` — each with Height/Range, DurationMultiplier,
  Ease field triplets). The **uniform field triplet per part**
  is itself the convention worth copying (§8.2).

## 5. Timing & easing evidence

All values are `.ctor` **code defaults** — serialized scene
values override them in the shipped game and are UNKNOWN
(§10). Ease names are resolved against the shipped
`DOTween.dll` enum literals (`dotween.il` lines 4183–4210).
Character durations are **multipliers of the current beat
segment**: `GetCurrentBeatInterval()` returns
`beatTimes[seg] − beatTimes[seg−1]` (the audio-clock segment
length), and each part's tween runs
`segment × multiplier`, split ½ out / ½ back (the ×0.5 at
`TriggerBeatAnimation` IL_0072 and siblings). At the default
120 BPM a segment is 0.5 s, so e.g. the head bob is a
0.15 s pulse — DERIVED from the measured multiplier at the
measured default tempo.

| Motion | Duration / rate | Easing shape | Parameters | Evidence |
|---|---|---|---|---|
| Head bob | 0.3 seg | **OutBounce** (Ease 30) | height 0.2 | `CharacterController` .ctor pairs |
| Body jump | 0.25 seg | **OutQuad** (6) | height 0.15 | .ctor pairs |
| Eye scale Y | 0.2 seg | **InOutSine** (4) | factor 0.8 | .ctor pairs |
| Pupil shuttle | 0.4 seg | **InOutQuad** (7) | range ±0.1, alternates | .ctor pairs + IL_016d |
| Camera drift | 0.4 seg | **InOutQuad** (7) | range 0.15 | .ctor pairs |
| Sweeper traversal | 1 seg | serialized `AnimationCurve`, code default **EaseInOut(0,0,1,1)** | arc height 1 (default), arcDown true | `ObjectManager` .ctor + `Update` |
| Camera zoom | 0.5 s fixed | **OutCubic** (9) | sizes 5 → 4 → 3 | `EffectMaager` .ctor |
| Indicator punch | 0.3 s | (scale tween; ease field-driven) | scale 1.2 | dynamics §6 |
| Hit-effect show/hide | 0.3 s / 0.5 s | **OutBack** (27) / **InQuad** (5) | hold 1.0 s | dynamics §6 |
| Score counter punch | 0.3 s | **OutQuart** (12) | scale 1.15 | dynamics §6 |
| Combo counter | 0.2 s | **OutBack** (27) | scale 1.15 | dynamics §6 |
| Accuracy counter | 0.25 s | **OutCubic** (9) | scale 1.15 | dynamics §6 |

- Frame-level timing patterns: the character's anticipation
  *is* the beat — every part pulses symmetrically (out and
  back in equal halves), so the motion reads as breathing
  with the tempo rather than reacting to events. No hold
  frames or smear equivalents exist in code; overshoot is
  supplied by OutBack/OutBounce easings on accents.
- Gameplay timing cross-reference: dynamics spec §3
  (judgment), §4 (sweeper), §7 (clocks).
- Pointer/input → animation mapping: input does not drive
  character animation at all — taps drive only judgment
  feedback (indicator, effects, shake). The character dances
  to the song, not to the player. (Contrast Magic Coin's
  pointer-scale 1.2, where input *is* the motion.)

## 6. Secondary motion

| Element | Driven by | Mechanism | Parameters (as shipped, code defaults) | Evidence |
|---|---|---|---|---|
| Pupils | beat segment | alternating L/R shuttle tween | ±0.1, 0.4 seg, InOutQuad; direction flag flips per trigger | `TriggerBeatAnimation` IL_016d–01d7 |
| Camera drift | beat segment | positional tween, range-limited | 0.15 range, 0.4 seg, InOutQuad | `CharacterController` .ctor |
| Camera shake | judgment events | **trauma model**: events add trauma; offset = noise × trauma², smooth-damped back to origin | decay 1.5/s, power 2, max rot 3°, max pos X 0.2 / Y 0.15, noise speed 25, damping 0.1 s; noise offsets seeded from a 0–1000 random range (at least one axis, .ctor) | `ShakeCameraFeel` .ctor pairs + `Update` |
| Eye pulse | beat segment | scale tween (also §5 — listed here as the "alive" layer over the static face) | Y ×0.8, 0.2 seg | `TriggerBeatAnimation` |
| Shader/glitch layer | authored events / parameters (wiring scene-side) | full-screen material effects: BW, CRT, scanner, tint, unsync, corrupted-VRAM, bleeding-colors + Kino glitch + `SimpleSpectrum` audio-reactive class present | `ShaderParams` + per-effect material `SetFloat` calls in `BWEffect` | class inventory, dynamics §0 |

- Blink / breathing cycles: **no blink system exists in
  code** — the eye-scale pulse is beat-locked, not a blink.
  Breathing is the aggregate of the §5 pulses (head/body at
  different multipliers = polymetric breathing: 0.3 / 0.25 /
  0.2 / 0.4 seg phases over the same segment grid).
- What has NO secondary motion (the honest negative): hair,
  cloth, soft-body — nothing spring- or physics-driven
  exists in the game code; all life is tween illusion.
  Haptics (Lofelt.NiceVibrations ships in `Managed/`) would
  be a non-visual secondary channel, but whether the game
  calls it is UNKNOWN (dynamics §9 U5).

## 7. Engine animation plumbing

| Engine feature used | How this title uses it | Evidence |
|---|---|---|
| DOTween (DG.Tweening) | The universal motion layer: every character part, UI counter, indicator, camera zoom, hit-effect envelope | `run1.il` passim; firstpass.dll is DOTween module shims |
| AnimationCurve (single) | Sweeper easing only — one curve, evaluated per frame from audio-clock t01 | `ObjectManager::Update` IL_009b |
| UniTask | Async BGM crossfade (frame-yield volume stepping) | dynamics §7 |
| UnityEvent (authored) | Beat-indexed effect triggers (`EventFeedbackEffect { beat, onEventTriggered }`) | `EffectMaager::OnRhythmEvent` |
| Physics 2D triggers | Judgment zones (animation-adjacent: they gate feedback) | dynamics §3 |
| Shader/material stack | Full-screen escalation effects driven by material floats | `BWEffect` + `ShaderEffect_*` classes |

- Update-order facts: all polls are plain `Update` — the
  game makes no use of `FixedUpdate`/`LateUpdate` in the
  inspected classes, so sweeper position, character pulses,
  input judgment, and frame-clock beats all sample within
  the same frame in engine-defined component order
  (unspecified in code). — **Evidence:** method inventory of
  the six core classes.
- Determinism-relevant facts: judged motion (sweeper) is a
  pure function of the audio clock + serialized parameters
  (deterministic given the same parameters); frame-clock
  listeners are deltaTime-quantized (nondeterministic
  across frame rates); shake noise offsets are randomly
  seeded (visual only, never feeds judgment).

## 8. Best practices worth adopting

| # | Practice (as this title demonstrates it) | Evidence | Disposition |
|---|---|---|---|
| 1 | **Tempo-relative durations**: every recurring motion's duration is a multiplier of the current beat segment, so animation stays glued to the music at any BPM with zero per-tempo tuning | §5 (multiplier table; `GetCurrentBeatInterval`) | ADOPT into Torogao's animation timing tables — durations in beat fractions, never bare seconds, for rhythm-synced motion |
| 2 | **Uniform per-part field triplets** (amplitude / duration-multiplier / ease) make a whole puppet tunable from one inspector pattern | §4 naming pattern | ADOPT as the Torogao puppet-parameter schema shape (mirrors the lumen evaluator's parameter discipline) |
| 3 | **Polymetric breathing**: parts pulse at different segment fractions (0.2–0.4) so the aggregate never looks mechanical | §5/§6 | ADOPT for idle/alive layers estate-wide (sprite-acting skill territory) |
| 4 | **Trauma-based shake with per-event intensities** (Perfect < milestone < boss) instead of fixed shakes | §6 (`ShakeCameraFeel`) | ADOPT — the estate game-feel skill already teaches trauma shake; this is a shipped, tuned instance with numbers |
| 5 | **Telegraph the judged path**: a LineRenderer draws the sweeper's arc, so the timing question is readable before it is asked | §1/`ObjectManager` line fields | ADOPT for any Torogao timing-judgment UI |
| 6 | **Authored beat-event table** (`{beat, event}` list) for escalation moments instead of code checks scattered per system | §3 (`EventFeedbackEffect`) | ADAPT — as RON data in S4, not UnityEvents |
| 7 | **Counter-tween illusion over physics**: zero physics secondary motion, all eased returns — cheap, deterministic, shippable | §6 negative finding | NOTE-ONLY as a budget option; Torogao's puppet stack aims higher (real secondary motion) but this is the floor that proves feel doesn't require it |

## 9. What Torogao would emulate

| Original technique (§ ref) | Torogao mechanism | Difference, if any, + reason |
|---|---|---|
| Tempo-relative tween pulses (§5) | Beat-fraction duration fields in the title's systems RON; evaluator advances pulses from the audio clock | Torogao drives from **one** clock (dynamics §10.1) — the original's frame-clock listener split is not emulated |
| Transform-part puppet (§2a) | Lumen puppet evaluator parameters (part transforms + deformers) with the §8.2 triplet schema | Torogao adds real secondary motion (springs) where this title fakes it — parity-or-better (D-456); the triplets remain the authoring surface |
| Eased arc sweeper (§3/§4 of dynamics) | Pure function position(t) over a named curve class; zone membership analytic | No physics engine in the judgment path (dynamics §11.1) |
| Trauma shake (§6) | Trauma accumulator in the session layer; per-event intensity table in RON | Same model, deterministic noise (seeded) so replays hash-match |
| Authored beat events (§3) | Beat-indexed event table in RON consumed by the scene graph | Data, not callbacks — testable headlessly |
| Shader/glitch escalation (§6) | Torogao post stack, parameter-driven from the same beat-event table | Estate-original shaders; the *escalation grammar* (clean → CRT → glitch with progress) is the transferable part, no shader code crosses |

- Feeds: the cross-medium animation pipeline (tempo-relative
  timing + polymetric idle layers are general techniques,
  filed as the bloch candidate below) and Torogao's rhythm
  spike (dynamics §10) — the spike's harness should render
  the §5 pulse table as its visual fixture.

## 10. Gaps & REVIEW register

| # | Unknown | What was tried | Route that would close it |
|---|---|---|---|
| 1 | Serialized overrides of every §5/§6 default (the shipped feel) | IL .ctor read (code defaults only) | DLL-metadata (AssetRipper-class) rung — dynamics §9 U1 |
| 2 | Sweeper `moveCurve` as shipped (code default is plain EaseInOut) | .ctor default read | Same rung (curve keys live in scene payloads) |
| 3 | Whether 2D Animation / IK rigs exist in scenes | IL reference grep (none in code) | Scene object census by class (pilot census extension) |
| 4 | Sprite-sheet organisation, actual character art structure | Out of scope for IL route | Media census (fathom R4-class) over the build |
| 5 | Shader/glitch escalation triggers (which beats/parameters drive the stack) | Class-level read only | Method-level read of `BWEffect`/`EffectMaager` wiring + U1 rung for scene wiring |
| 6 | Blink behaviour at runtime (none in code — possibly sprite-side) | Full IL read of character classes | Runtime observation (REVIEW-grade) |

## 11. Provenance

- Fathom routes used: typetree-independent MonoBehaviour
  census (pilot); ikdasm IL route (COIN-DYNAMIC §5
  precedent).
- Extraction commands of record: dynamics spec §Provenance.
- Bloch candidates filed from this note:
  `../bloch/hypno-rhythm-three-clocks.md` (the clock
  architecture + tempo-relative timing practices).
- Skill-trust records: recorded at close with the dynamics
  spec.
