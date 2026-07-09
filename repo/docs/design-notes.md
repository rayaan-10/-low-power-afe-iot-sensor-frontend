# Design Notes

Worked-through rationale for each stage, written so you (future you, in an interview) can
reconstruct the "why" without having to re-derive everything from the schematic cold.

## 1. Single-supply biasing

The circuit runs off a single 3.3 V rail — no negative supply. But sensor signals are
AC (they swing above and below their resting value). A single-supply op-amp can't output
negative voltage, so the trick is to **artificially shift the whole AC signal up** so it
sits entirely within the 0–3.3 V window.

That's what the R1/R2 (100 kΩ / 100 kΩ) divider off the 3.3 V rail does — it creates a
mid-supply reference (~1.65 V) that the signal path is biased around. C2/C3 decouple that
reference so it stays stable (a "AC ground" for the bias node). Everything downstream then
swings around 1.65 V instead of around 0 V.

## 2. Input protection (BAT54 clamp)

The sensor connects through a 100 Ω series resistor into node N1, which is clamped by a
pair of BAT54 diodes to the two supply rails (Vcc / Vgnd side). If the sensor ever
outputs something outside the safe input range (fault condition, static discharge,
disconnected sensor picking up noise), the diodes turn on and clamp the node instead of
letting it damage or saturate the op-amp input. The 1 MΩ resistor to the bias node
(`IN_BIAS`) sets the DC operating point of that same node when nothing else is driving it.

**Trade-off to be upfront about:** diode clamps do add a small amount of leakage current
and nonlinearity right at the clamp threshold. For a low-level sensor signal that never
approaches the clamp voltage in normal operation, this is invisible — but it's worth
naming as a design trade-off rather than treating the diodes as "free."

## 3. Amplification stage — gain math

The high-gain stage uses a feedback pair sized for roughly:

```
Gain = Rf / Rg = 1 MΩ (R5) / 10 kΩ (R3) ≈ 100×  (≈ 40 dB)
```

A separate stage uses a 90 kΩ / 10 kΩ pair for a lower, secondary gain step. Splitting gain
across two stages instead of hitting 100× in one shot keeps each op-amp comfortably inside
its gain-bandwidth product and reduces the chance of one stage slamming into the rail
before the signal even reaches the filter.

## 4. Active low-pass filter — cutoff math

The filter stage uses matched 16 kΩ resistors (R6, R7) and matched 10 nF capacitors
(C4, C5) in a two-pole low-pass configuration:

```
f_c ≈ 1 / (2π × R × C) = 1 / (2π × 16 kΩ × 10 nF) ≈ 995 Hz
```

Two matched RC legs in this arrangement give a steeper (2nd-order, −40 dB/decade) roll-off
than a single RC stage (−20 dB/decade), which matters because the noise this stage is
meant to reject (switching noise, EMI pickup) is broadband and needs to be knocked down
hard, not just gently attenuated.

## 5. Output buffer

A final MCP6002 stage is wired as a unity-gain voltage follower (output tied directly to
the inverting input). Its only job is impedance transformation: the filter stage has a
finite output impedance, and if you connect an ADC input (or long wiring, or a second
board) directly to it, that load can pull current and distort the filtered signal. The
buffer presents a high input impedance to the filter and a low output impedance to
whatever comes next, so the filtered waveform is preserved unchanged.

## 6. Power budget derivation

```
P = V_CC × N_amp × I_q
```

- `V_CC` = 3.3 V (single supply rail)
- `N_amp` = 4 (four op-amp stages used: bias buffer / gain / filter / output buffer,
  spread across two MCP6002 dual packages)
- `I_q` ≈ 100 µA per amplifier (MCP6002 typical quiescent current from its datasheet)

```
P ≈ 3.3 × 4 × 100 µA ≈ 1.32 mW
```

Compare against the referenced state-of-the-art ECG AFE at 6.55 µW — roughly 200× lower.
That gap is expected and worth explaining honestly: that reference is a custom 0.18 µm
CMOS IC design with sub-threshold biasing and IC-level power tricks that aren't accessible
when you're stitching together commercial op-amp ICs on a breadboard. The project's
contribution isn't "beat the state of the art" — it's "understand what a low-power AFE
has to do, build a working version discretely, and know exactly where the power actually
goes so you can reason about what a custom IC would need to do differently."

## 7. Simulated vs. measured gap

Simulated final-stage amplitude (`V(fifth)`) was ≈156 mV; measured RMS on hardware was
146.53 mV. A few mV difference here can come from:

- Resistor tolerance (5% carbon-film resistors vs. ideal simulation values)
- Breadboard parasitic capacitance/inductance affecting the filter corner slightly
- Loading from the oscilloscope probe itself (10:1 vs. 1:1 probe changes input
  capacitance seen by the circuit)
- Op-amp input offset voltage / bias current not perfectly matching the SPICE model

None of these are large enough to suggest something is *wrong* — they're the normal,
expected delta between a SPICE model and a hand-built breadboard prototype, and being able
to name the likely sources of that gap is itself a useful thing to be able to say out loud
in an interview.
