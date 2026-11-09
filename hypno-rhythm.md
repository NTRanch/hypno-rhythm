# Hypno_Rhythm — best practices (and an extraction lesson)

Study update (2026-11-08, queue #3): the IL pass landed —
`hypno-rhythm-dynamics.md` (S3 spec, MEASURED code constants;
three-clock audio analysis; the Torogao audio-sync spike
requirements) and `hypno-rhythm-animation.md` (S2b note:
procedural DOTween puppet, tempo-relative pulse table). The
best-practices sections live in those records (§11 / §8);
this file remains the intake-level note.

Evidence: intake census (Unity 6000.0.32f1 Mono, 188 files,
213,584,843 B — the smallest complete Unity Mono title in the tree);
`~/workspace/fathom-unity-mono-pilot-20261103/PILOT-REPORT.md` (it was
the fathom extraction pilot title, picked mechanically: smallest
complete Mono root).

What this game does well:

1. **A rhythm loop with one timing question.** A rhythm game's whole
   contract is "was that on the beat" — a single binary judgment,
   repeated. Like the coin's one verb, the loop is small enough that
   tuning *is* the design; there is nowhere for sloppiness to hide.
2. **Smallness as a feature.** 188 files / 214 MB total for a complete
   shipped game. The estate's own demo doctrine (production-esque,
   ≤10-minute pieces) points the same way: a tight loop shipped whole
   beats a broad loop shipped thin.

The extraction lesson this title taught the estate (the reason it
matters beyond its loop): its SerializedFiles — like every sampled
title in the 17-game Mono corpus — ship with **type trees disabled**
(every type row has node count 0). Typetree readers (UnityPy
`read_typetree()`) therefore hard-fail on effectively every
MonoBehaviour, not because the data is corrupt but because the data
is *not there*. The working route is a typetree-independent census:
fixed-layout MonoBehaviour headers + MonoScript identity via the
externals table, with field expansion deferred to a DLL-metadata
(AssetRipper-class) rung. Filed as a bloch candidate
(../bloch/unity-typetree-absence.md).
