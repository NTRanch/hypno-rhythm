# hypno-rhythm — measured specification

The clean-room study record for the Torogao remake of **Hypno_Rhythm**
(Unity 6000.0.32f1, Mono), by Nice Thoughts Ranch. This repository
currently holds the **measured specification only — the systems build
has not started.** The build (programme stage S4) is gated on a Torogao
audio-sync spike: the original's judgement is spatial rather than
temporal (Physics2D trigger volumes, not a timing window), and the
remake's sync design must be settled before the crate is written.

## Contents

- `hypno-rhythm-dynamics.md` — the S3 dynamics spec: measured code
  constants, the three-clock audio analysis, and the audio-sync spike
  requirements. Headline facts: sole input is one key; scoring is
  Perfect 300 / Good 100; the accuracy denominator counts taps, not
  notes.
- `hypno-rhythm-animation.md` — the S2b animation note: procedural
  DOTween puppet work and the tempo-relative pulse table.
- `hypno-rhythm.md` — the intake-level note and best-practices record.

## Method

Clean-room: constants and behaviour were measured from the studied
build (including IL records); no assets, text, or code from the
original are included or will ship. When the build starts it will
follow the programme's dual-target law — native Linux and Android from
one crate plus its data file, with a deterministic trajectory fixture
shared by both targets.

The studied game remains the property of its respective developer;
this repository is an independent study and reimplementation record.

## License

BSD 2-Clause. See `LICENSE`.
