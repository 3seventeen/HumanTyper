# Human Typer - Realistic Typing Simulator

A Tampermonkey/Greasemonkey userscript that simulates realistic human typing, complete with typos, corrections, natural pauses, fatigue, and burst patterns. All timing parameters are derived from peer-reviewed keystroke dynamics research.

## Installation

1. [Install here](https://update.greasyfork.org/scripts/566022/Human%20Typer%20-%20Realistic%20Typing%20Simulator.user.js)
4. The panel appears on every page (toggle with **Ctrl+Shift+H**)

## Usage

1. Navigate to any page with a text field
2. Click on the target text field (the panel shows which element is targeted)
3. Paste your text into the Human Typer panel
4. Set your desired WPM range
5. Click **Start Typing**

### Controls

| Control | Description |
|---------|-------------|
| **Start Typing** | Begins typing into the targeted field |
| **Stop** | Gracefully stops after the current character |
| **Speed (WPM)** | Set min-max range for words per minute |
| **Ctrl+Shift+H** | Toggle panel visibility |
| **–** button | Minimize to header only |
| **x** button | Hide panel (reopen with Ctrl+Shift+H) |
| **Drag header** | Reposition the panel |

## Features

### Timing Engine

- **Ex-Gaussian distribution** for inter-key intervals (IKI), matching the right-skewed timing observed in real keystroke data
- **WPM-compensated**: overhead from pauses, modifiers, and distribution tails is factored into base IKI so actual throughput matches your target WPM
- **Per-character modifiers**: common letters (e, t, a, o, i, n, s, h, r) type faster; uncommon letters (z, x, q, j, k, v) type slower; capitals and symbols are slowest
- **Bigram-aware**: hand-alternating pairs (e.g. "th", "en") are fastest, same-hand different-finger pairs are medium, same-finger repeats (e.g. "ed", "de") are slowest

### Key Dwell Time

Real keyboards register how long each key is held down (keydown → keyup). Synthetic events without dwell time are a detection vector.

| Key Type | Hold Duration |
|----------|--------------|
| Normal letter keys | 65-115ms |
| Pinky-operated keys (q, a, z, p, ;, etc.) | 85-135ms |
| Spacebar (thumb) | 55-95ms |
| Per-keystroke jitter | ±12ms |

- Dwell scales with WPM (faster typists hold keys more briefly)
- Keyup fires asynchronously after the dwell period (non-blocking)
- Shift key dispatched with correct hand side (left shift for right-hand keys, vice versa) and proper `location` field

### Realistic Keyboard Events

Every keystroke dispatches events with accurate metadata:

- **Physical key codes** (`KeyA`, `Digit1`, `Semicolon`, `BracketLeft`, etc.) matching real hardware
- **Legacy keyCode/which** values for older event APIs
- **Shift modifier** with correct `shiftKey` flag and separate shift keydown/keyup events
- **Shift hand selection**: touch typists use opposite-hand shift (left shift for right-hand letters)
- **charCode** on keypress events

### Burst Typing

Real typing follows a chunk-and-pause rhythm, not a constant stream:

- **Word-aware burst sizing**: burst length adapts to word structure instead of fixed random ranges
  - Short remaining-in-word extends into space + next word
  - Medium words (≤7 chars) often typed in one burst
  - Long words broken into 4-8 char chunks
  - Common words get longer bursts (muscle memory)
- **180-600ms pauses** between bursts
- Errors and corrections reset the burst flow

### Error System

Four error types weighted by real-world frequency:

| Error Type | Weight | Example |
|-----------|--------|---------|
| Substitution | 40% | "the" → "thr" (adjacent key on QWERTY) |
| Insertion | 35% | "the" → "thre" (extra adjacent key) |
| Transposition | 15% | "the" → "teh" (swapped pair) |
| Omission | 10% | "about" → "abot" (skipped a letter) |

- **QWERTY adjacency map** ensures substitutions hit physically neighboring keys
- **~6% base error rate** (matches trained typist data)
- **Context-dependent error rate**:
  - Longer words have higher error rates (+0.8% per char beyond length 4)
  - Common words (top ~120 English words) have 50% fewer errors (muscle memory)
  - Middle-of-word characters are more error-prone (bell curve peaking at center)
  - Post-error frustration: 30% error rate boost that decays over 5 characters

### Habitual Typo Personality

Each typing session generates a unique "typist personality":

- **3-6 habitual letter swaps** randomly chosen from adjacent key pairs
- These pairs trigger at **15% rate** when encountered (vs 6% base)
- Creates consistent, repeatable error patterns per session — not purely random
- Mimics how real people have specific letters they always mix up

### Error Correction

Corrections follow a realistic detection distribution:

| Detection Speed | Probability | Behavior |
|----------------|-------------|----------|
| Immediate | 70% | Backspace right away |
| Short delay | 20% | Notice 2-5 chars later, backspace to error |
| Long delay | 8% | Notice 6-15 chars later, backspace all the way back |
| Uncorrected | 2% | Typo left in the text |

Delayed corrections erase everything back to the error point and replay the text correctly through the main typing loop.

### Backspace Behavior (WPM-independent)

Backspace timing is separate from typing speed — people mash backspace at a consistent rate regardless of how fast they type:

| Phase | Timing |
|-------|--------|
| First backspace | 80-150ms (finger moves to key) |
| Consecutive backspaces | 35-65ms, accelerating by 5ms each |
| Floor speed | 30ms minimum |
| "Oh crap" hesitation | 120-350ms before starting (scales with error distance) |
| Post-correction pause | 80-250ms (re-reading before resuming) |
| Re-typing after correction | 15% slower than normal (being more careful) |

### Natural Pauses

Pauses follow the hierarchy observed in writing research:

| Context | Pause Duration |
|---------|---------------|
| Between letters (IKI) | 60-200ms |
| Between words (space) | Slightly longer than IKI |
| After comma/semicolon | 150-400ms |
| After period/!/? | 300-900ms |
| Paragraph break | 1.2-3.0s |
| Thinking pause (3% chance per word) | 2.0-5.5s |

### Fatigue Simulation (Nonlinear)

Real fatigue is messy — not a straight line. The fatigue model combines multiple overlapping signals:

- **Overall upward trend**: +0.2% per minute (caps at +20%)
- **Attention wave**: sinusoidal 3-5 minute cycle (±3% speed variation)
- **Micro-oscillations**: two overlapping faster frequencies simulating irregular focus fluctuations
- **Second wind dips**: occasional brief recovery moments
- **Error rate oscillates** in sync with attention waves
- Floor at 92% ensures the model never makes you faster than rested speed

### Entropy Layers

Adds chaotic human noise that pure mathematical distributions can't replicate:

| Layer | Effect |
|-------|--------|
| **Micro-tremor jitter** | ±8ms random per keystroke (hand/finger instability) |
| **Attention drift** | ±4% speed variation over 20-45 second cycles |
| **Zone states** | 0.2% chance per char of entering "the zone" (15% faster) or "brain fog" (20% slower) for 8-25 characters |

Each session randomizes the phase and period of all oscillations, so no two runs produce the same timing signature.

### Compatibility

- Works on **all websites** (`@match *://*/*`)
- Standard text inputs and textareas
- ContentEditable elements (rich text editors)
- Panel z-index set to maximum (`2147483647`) to stay above all UI


## Research Sources

The typing parameters in this script are based on the following peer-reviewed studies:

1. **Dhakal, V., Feit, A. M., Kristensson, P. O., & Oulasvirta, A. (2018).** "Observations on Typing from 136 Million Keystrokes." *ACM CHI Conference on Human Factors in Computing Systems.*
   [doi.org/10.1145/3173574.3174220](https://dl.acm.org/doi/10.1145/3173574.3174220)
   — IKI distributions, error rates, burst patterns, and dwell times from 168,000 participants.

2. **Pinet, S., Zielinski, C., Alario, F.-X., & Longcamp, M. (2022).** "Typing Expertise in a Large Student Population." *Cognitive Research: Principles and Implications.*
   [PMC9356123](https://pmc.ncbi.nlm.nih.gov/articles/PMC9356123/)
   — IKI by typist speed class, hand alternation vs same-finger timing ratios.

3. **Van der Linden, D., Tops, M., & Bakker, A. B. (2020).** "Dynamics in Typewriting Performance Reflect Mental Fatigue." *PLOS One.*
   [doi.org/10.1371/journal.pone.0239984](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239984)
   — Fatigue effects: IKI increase (+5-10%/hour), backspace rate increase (4.2% → 5.4%), delayed error detection.

4. **Wobbrock, J. O., & Myers, B. A. (2008).** "TrueKeys: Identifying and Correcting Typing Errors for People with Motor Impairments." *ACM International Conference on Intelligent User Interfaces.*
   [faculty.washington.edu](https://faculty.washington.edu/wobbrock/pubs/iui-08.01.pdf)
   — Error type distribution: 60% insertions, 17% substitutions, 15% omissions, 6% transpositions.

5. **DataGenetics. (2012).** "Sloppy Typing."
   [datagenetics.com](http://datagenetics.com/blog/november42012/index.html)
   — Per-key error likelihood rankings, adjacency-based substitution patterns.

6. **Medimorec, S., & Risko, E. F. (2017).** "Pauses in Written Composition: On the Importance of Where Writers Pause." *Reading and Writing.*
   [doi.org/10.1007/s11145-017-9723-7](https://link.springer.com/article/10.1007/s11145-017-9723-7)
   — Pause hierarchy: within-word < between-word < sentence < paragraph < thinking.

7. **Baaijen, V. M., & Galbraith, D. (2018).** "Constructing Theoretically Informed Measures of Pause Duration in Writing." *Reading and Writing.*
   [doi.org/10.1007/s11145-022-10284-4](https://link.springer.com/article/10.1007/s11145-022-10284-4)
   — Pause threshold categories: <300ms motor, 300-999ms word planning, 1000-1999ms sentence planning, >2000ms cognitive.

8. **Baus, C., Strijkers, K., & Costa, A. (2016).** "Keystroke Dynamics as Signal for Shallow Syntactic Parsing." *COLING.*
   [aclanthology.org/C16-1059](https://aclanthology.org/C16-1059.pdf)
   — Burst length characteristics and between-burst pause distributions.

## License

All Rights Reserved. This software may not be modified, redistributed, or used in derivative works without explicit written permission from the author (3seventeen).
