# Kuramoto EEG Demo

> [!WARNING]
> **All files** in this repository have been vibe-coded using Claude Sonnet 4.6 without human manual edits to the code itself at all, except this very set of sentences in `README.md`, written by hand by Paul. One major conceptual flaw was noticed during the CoE meeting by A. Danielsen in the simulated EEG signals, showing an entrainment pattern in contradiction with the theory. The purpose of this exercise was to get familiarized with the use of agentic AI. 

Interactive demo showing how coupled neural oscillators synchronise to a musical beat.
Built for the CoE meetup hackathon 2026.

**Live page:** https://claude.ai/artifact/PY5SfEwXh4n8JJ5odvuTJD

---

## What it does

Three panels update in real time:

- **Phase circle.** Each dot is one of 19 oscillators at its current phase angle. When they cluster, the brain is in sync. The amber dot marks the beat phase target.
- **Brain map.** 10-20 EEG layout. Nodes pulse green (firefly style) each time their oscillator crosses zero phase. Amber nodes are the auditory cortex (T7, T8, P7, P8), the ones directly driven by the beat.
- **EEG traces.** T7 (auditory, left) and Fz (frontal midline). The vertical red line marks each beat.

Three controls:

| Control | Range | Effect |
|---|---|---|
| Coupling K | 0–10 | How strongly oscillators pull each other toward a shared phase |
| Tempo | 40–240 BPM | Beat rate. At 50/75/150 BPM the beat is a harmonic of the 2.5 Hz oscillators (3:1, 2:1, 1:1) |
| Beat strength | 0–1 | Forcing amplitude on auditory nodes |

---

## The model

Standard Kuramoto network plus Arnold-tongue entrainment for auditory nodes:

```
dθᵢ/dt = ωᵢ + (K/N) Σⱼ sin(θⱼ − θᵢ) + ε · sin(n·Ω_beat·t − θᵢ)   [auditory only]
```

- N = 19 oscillators, natural frequency ωᵢ ~ 2.5 Hz (delta/theta)
- n is chosen so that n · f_beat ≈ 2.5 Hz, putting the forcing at the oscillator's resonance
- Order parameter R = |Σ eⁱθ| / N, shown live in the phase panel
- Euler integration, dt = 5 ms

The forcing term is continuous, not a discrete phase reset. This is the Arnold tongue entrainment mechanism: the oscillator is pulled toward the beat phase smoothly each cycle.

---

## Interesting parameter combinations

| Setting | What to watch |
|---|---|
| K=0, beat strength=0 | Oscillators drift independently, R stays near 0 |
| K=5, beat strength=0 | Spontaneous synchrony, R climbs toward 1 |
| K=2, tempo=150 BPM, beat strength=0.8 | 1:1 locking, auditory nodes lock first, then pull frontal via coupling |
| K=2, tempo=75 BPM, beat strength=0.8 | 2:1 locking, T7 fires twice per beat |
| K=2, tempo=50 BPM, beat strength=0.8 | 3:1 locking |
| Sweep tempo slowly | Watch R collapse and rebuild at each resonant BPM |

---

## How we built it

This demo grew out of a one-session conversation with Claude Code. The process was:

**1. Design interview (`/grill-me`)**
Before writing a line of code, Claude ran a structured interview, one question at a time, working through every design branch: audience (cognitive neuroscience researchers, familiar with EEG but not Kuramoto), format (demo not tutorial), inputs (music as phase-entraining forcing on auditory subset), outputs (phase circle + brain map + EEG traces), controls (coupling, tempo, beat strength), and feedback (metronome click + beat dot, no explanatory text on screen).

**2. First build**
Single-file HTML, all inline CSS and JS, Canvas 2D rendering, Web Audio API for the click. No dependencies except Google Fonts. The first version used discrete phase resets on each beat, which produced sawtooth artifacts in the EEG trace.

**3. Iterative fixes via `/criticize`**
After publishing, Claude ran a self-critique, ranking issues by impact. The main findings:

- **Physics bug.** Discrete phase resets created unrealistic sawtooth jumps visible in EEG traces. Fixed by replacing the interrupt with continuous Arnold-tongue forcing added to the ODE derivative.
- **n:m locking absent.** Forcing was always at f_beat regardless of tempo. Fixed by computing beatN = argmin_n |n·f_beat − 2.5 Hz| and using n·Ω_beat in the forcing term.
- **Activity gating.** Nodes were lit for half the cycle (half-wave rectification). Fixed to a Gaussian pulse exp(−2(1−cos θ)) so nodes flash briefly at zero crossing, like fireflies.
- **Beat flash was frame-rate dependent.** Fixed to simTime-based 100ms window.
- **Performance.** Order parameter was computed twice per frame. Cached once in animate().

**4. Visual polish**
Head outline was near-invisible against the dark background. Legend added for auditory (amber) nodes. Phase circle got a beat-phase marker and "cluster = sync" label. Edge legend renamed from "phase coherence" to "phase alignment".

**Key takeaway for the hackathon:** the `/grill-me` interview before building saved more time than any subsequent fix. Getting the physics right (Arnold tongue vs. discrete reset) required one back-and-forth after seeing the artifact in action. The model caught most of its own bugs when prompted with `/criticize`.

---

## Data and state

No data leaves the browser. The simulation runs entirely in JavaScript:

- Oscillator phases (`theta[]`) and natural frequencies (`omega[]`) are plain JS arrays, recomputed each frame by Euler integration.
- EEG trace buffers (`trT7`, `trFz`) are fixed-length circular arrays overwritten continuously.
- Nothing is written to localStorage, sent to a server, or saved to disk.

Reload the page and it starts from scratch with new random initial phases.

---

## Files

```
kuramoto-eeg.html           self-contained demo, open in any browser
README.md                   this file
conversation-transcript.md  the Claude Code session that built the demo
```

`conversation-transcript.md` is a readable rendering of the Claude Code session log. It covers the whole session in order: the design interview, the first build, the self-critique and fixes, and the GitHub setup. User prompts and Claude's replies are verbatim, with a UTC timestamp per turn. Each tool call (file edit, shell command, publish) is one line. Tool output, model thinking and harness metadata are left out. Email addresses are redacted. The raw log is a 2.5 MB JSONL file kept locally by Claude Code under `~/.claude/projects/`.
