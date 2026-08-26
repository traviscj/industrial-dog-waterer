# Industrial dog waterer — Tier 3 design

Reference unit being replaced: Kastty FS55. 2 gal (7.6 L), ABS, 13.2×12.4×5.5 in,
sub-30 dB pump, "5-stage" = PP cotton + coconut carbon pad + hair pre-filter bag,
LED low-water indicator, ~470 mL of dead-power reserve. $39.99.

Everything below is the delta from that.

---

## 0. Design intent, stated as requirements

| # | Requirement (yours) | Engineering translation | Where solved |
|---|---|---|---|
| R1 | copper pipes | 1/2" Type L copper for the visible loop + spout | §3, §9 |
| R2 | better cleaning | CIP-able loop, no dead legs, drain valve, continuous blowdown | §5, docs/CIP.md |
| R3 | larger tank | 57 L (15 gal) reservoir → ~11-day interval | §2 |
| R4 | less frequent filter changes | delete carbon; washable pleated + UV; zero monthly consumables | §4 |
| R5 | surplus tank, refillable | stainless keg on load cells, top-fill, or hard-plumbed autofill | §2, §6 |
| R6 | email when surplus low | Prometheus + Alertmanager, incl. `predict_linear` lead time | §7 |
| R7 | (added) blinking light, cloud-free | 3-colour stack light driven locally by the ESP32, no network needed | §7.0 |

---

## 1. Architecture

Two hydraulically independent circuits. This is the single most important decision.

**Circuit A — makeup (gravity, intermittent).** Reservoir → shutoff → level control → bowl.
Carries municipal water *with its chlorine residual intact*. Slow, cold, stored.

**Circuit B — recirculation (pumped, continuous).** Bowl → strainer → pump → sediment filter
→ UV → copper riser → spout → bowl. Carries bowl water only. Fast, warm, organically loaded.

Keeping them separate means the pump can never run the reservoir dry, the filter never sees
57 L it doesn't need to, and a fault in B cannot drain A into your floor.

```
        [ 57 L SS reservoir ]  <-- top fill (or autofill, §6)
             | on 4× load cells (HX711)  -> reservoir_liters
             | 1/2" SS braid, full-port ball valve (service isolation)
             v
        [ makeup valve ]  NC solenoid  (Variant A)
             |            or mech. float valve + NC solenoid backstop (Variant B)
             v
   +---> [ BOWL, 304/316 SS, ~2 L working ] ---> [ overflow standpipe ] --> DRAIN
   |          |            ^                                                  ^
   |          | bottom     | spout (copper)                                   |
   |     [ SS strainer ]   |                                              [ blowdown ]
   |          |            |                                               motorized
   |          v            |                                                ball valve
   |      [ PUMP ]---->[ 10" pleated ]---->[ UV-C ]---->[ Cu riser ]-----------+
   |       12V BLDC        20 µm washable    9-18 W       1/2" Type L
   |       PWM 2->10 L/min
   +--------------------------------------------------------------------------+
```

### Variant A vs B — the one decision that matters

- **Variant A (a drain is reachable):** overflow standpipe to drain. You *cannot* flood the
  room, because the failure mode of every valve is "water goes down the standpipe." Electronic
  level control is now safe. Blowdown is automatic. **Choose this if you can reach a utility
  sink, washer standpipe, floor drain, or condensate pump within ~15 ft.**
- **Variant B (no drain):** mechanical float valve is primary level control (no firmware in the
  flood path), NC solenoid in series as an electronic backstop, drip tray under the whole
  footprint sloped to a leak rope that latches both valves closed. Blowdown goes to a 20 L
  greywater jug — *on its own load cell*, so it alerts when it's full. You empty it weekly.

Variant A is strictly better and cheaper. Variant B exists so this isn't blocked on plumbing.

---

## 2. Reservoir sizing

Consumption model:

```
drinking      ≈ 50 mL/kg/day        (normal range 40–70; >100 = see a vet, §7.4)
evaporation   ≈ 0.4–0.8 L/day       open bowl + falling aerated stream, indoor
blowdown      ≈ 1.0–1.5 L/day       (§5, dumping the bowl)
```

| Dogs | Drinking | + evap | + blowdown | **Total/day** | 19 L (corny keg) | 57 L (15 gal) |
|---|---|---|---|---|---|---|
| 2 × 20 kg | 2.0 L | 2.5 | 3.7 | **3.7 L** | 5.1 d | 15.4 d |
| 2 × 30 kg | 3.0 L | 3.5 | 4.7 | **4.7 L** | 4.0 d | 12.1 d |
| 2 × 40 kg | 4.0 L | 4.5 | 5.7 | **5.7 L** | 3.3 d | 10.0 d |
| 3 × 30 kg | 4.5 L | 5.0 | 6.2 | **6.2 L** | 3.1 d | 9.2 d |

**57 L is the sweet spot** — it puts the refill on a "whenever, roughly biweekly" cadence
instead of a chore. Below ~40 L you have not actually escaped the problem; a 19 L corny keg
buys you only 2× over the Kastty.

Vessel options:

| Option | Vol | $ | Notes |
|---|---|---|---|
| **15 gal Sanke keg** | 58.7 L | $110–160 | 304 SS, sanitary, needs a spear-removal tool once |
| 15 gal HDPE drum, 2" bung | 57 L | $70 | lightest, easiest to fill, opaque, cheapest |
| 2× ball-lock corny keg | 38 L | $180 | modular, best fittings ecosystem, but 2× the cleaning |
| 15 gal SS brew kettle + lid | 57 L | $130 | wide open top = trivial to clean & fill, but not sealed |

Recommended: **Sanke or the brew kettle.** Stainless does not harbour biofilm the way scratched
HDPE does, and you can hit it with hot PBW without worrying.

Mount it **1.0–1.3 m above the bowl rim.** That is 0.10–0.13 bar (1.4–1.9 psi) of static head —
enough to gravity-feed 1/2" line at several L/min, and enough for a low-pressure float valve.
Note: many float valves (Fluidmaster-type) need ≥10 psi to seal. Buy one **rated from 0 psi**
(Kerick MA052 / Hudson low-pressure / Jobe livestock valve). This trips people up.

Fill port: 2" tri-clamp or the kettle lid. Put a funnel and a 5 µm mesh screen on it so you're
not decanting sink grit into a system you just spent a weekend sanitizing.

---

## 3. Copper — do it, but do it right

Copper is defensible here, not just aesthetic: it is oligodynamic. Copper surfaces measurably
suppress biofilm formation, which is exactly the failure mode of every consumer pet fountain.
It's also stiff, solderable, cleanable, and it looks like the thing you wanted to build.

**Use:** 1/2" Type L (0.545" ID) for the pump discharge, filter/UV interconnect, riser, and spout.
**Do not use:** copper for the reservoir, or anywhere water sits stagnant >8 h.

### Leaching — the actual risk, quantified

Copper-associated hepatopathy is real and breed-linked: Bedlington Terriers (COMMD1), and to a
lesser and more variable degree Labradors, Dobermans, Westies, Dalmatians. Dietary copper is the
dominant exposure, but water is additive.

- EPA action level (humans): **1.3 mg/L**.
- NRC recommended allowance, 30 kg dog: ≈ 0.2 × 30^0.75 ≈ **2.6 mg/day**.
- A 30 kg dog drinking 1.5 L/day at 1.0 mg/L picks up **1.5 mg/day** — i.e. water alone could add
  ~60% on top of the dietary allowance. Not acutely toxic, well under any safe upper limit, but
  it is not nothing, and if you have a predisposed breed it's the wrong direction.

Six mitigations, in order of effect:

1. **Never stagnate.** Leaching is driven by contact time. Continuous recirculation means water
   is in copper for seconds, not overnight. This alone is worth an order of magnitude. If you
   duty-cycle the pump, flush the riser to drain on restart rather than serving first-draw water.
2. **Check whether your water is favourable — this varies enormously by utility.** Pull your
   annual Consumer Confidence Report and look for pH, alkalinity, and (if published) the
   Langelier Saturation Index. Water that is alkaline and moderately hard is *scale-forming*:
   it passivates copper rather than dissolving it. Soft, low-alkalinity, or acidic water
   (pH < 7, alkalinity < 60 mg/L as CaCO₃) is aggressive to copper and should push you toward
   stainless. Example of a favourable supply: Great Lakes surface water typically runs
   pH 7.8–8.0, alkalinity ~110 mg/L as CaCO₃, LSI positive. Example of an unfavourable one:
   soft well water or acidified mountain supply. **Do not assume; look it up.**
3. **Passivate before service.** Run the loop for 72 h on tap water, dumping and refilling daily,
   before a dog ever drinks from it. Let the patina form on your time, not theirs.
4. **No acid dwell.** StarSan/phosphoric sitting in copper overnight is a leaching spike. Use
   peracetic acid at short contact and rinse (docs/CIP.md), or hot-water sanitize.
5. **Lead-free everything.** Sn/Ag or Sn/Cu solder (**never** 50/50 Sn-Pb), and NSF/ANSI-61
   "no-lead" brass (≤0.25% Pb) for any valve body. Ordinary yellow brass fittings are ~2% lead.
   Cheapest way to avoid the question entirely: solder joints + stainless valves, zero brass.
6. **Measure it.** Bicinchoninate colorimetric test, ~$15 for 50 strips, 0.05–3 mg/L range. Pull a
   first-draw sample at 30 days and 180 days. **Target < 0.3 mg/L, act at > 0.5.** Log it.

If you have a Bedlington/Lab/Doberman: build the wetted path in 316L SS and use copper as a
non-wetted decorative cladding or as a tin-lined line. You lose the antimicrobial benefit, you
keep the look, you eliminate the argument.

---

## 4. Filtration — why the stock design is wrong and yours should delete most of it

The Kastty's "5-stage" is: hair bag, PP cotton, coconut carbon, more PP, sponge. In a closed
recirculating loop this is close to actively harmful:

- **Activated carbon is exhausted within days** for anything it actually adsorbs, and there is
  nothing left to adsorb in a loop you already dechlorinated.
- **Carbon then becomes a bacterial substrate.** Warm, dark, huge specific surface area, organic
  load from saliva, zero disinfectant residual. It is a bioreactor with a pet-safe label. This is
  the real reason those pads "need" monthly replacement — not exhaustion, colonisation.
- **The carbon strips the chlorine** that was the only thing keeping the water microbiologically
  stable in the first place.

Replace the whole stack with three things that don't wear out:

| Stage | Part | Purpose | Service interval |
|---|---|---|---|
| S1 | 304 SS mesh strainer basket at the bowl drain, 800 µm | hair, kibble, leaves | rinse weekly, lasts forever |
| S2 | 10" Big Blue **washable pleated polyester**, 20 µm | fines, turbidity | hose off quarterly; replace ~3 yr |
| S3 | Inline UV-C, 9–18 W | disinfection | **lamp 12 mo**, sleeve wipe quarterly |

That is **one scheduled consumable per year** (a UV lamp, ~$25) versus twelve carbon pads.

UV sizing sanity check: loop volume ≈ 8 L (2 L bowl + ~5 L filter housing + ~1 L pipe).
At 2 L/min recirc that's 360 turnovers/day. Even a cheap 9 W pond clarifier delivering a modest
per-pass dose accumulates an enormous cumulative dose. **You are not dose-limited, you are
maintenance-limited** — the #1 UV failure is a fouled quartz sleeve, not lamp output. Hence:
wipe the sleeve at every CIP, and instrument the lamp (§7.3).

Deliberately **not** included, with reasons:
- *Carbon* — see above. Optional exception: a carbon block on a hard-plumbed autofill *only*
  if your tap tastes bad enough that the dogs care. It costs you the reservoir's chlorine residual.
- *RO* — strips minerals, wastes 3:1, produces aggressive low-TDS water that attacks copper.
- *Ozone* — corrodes copper aggressively, and residual management is a project unto itself.
- *Silver/ion cartridges* — unregulated, unmeasurable, and you already have copper.

---

## 5. Cleaning — CIP and continuous blowdown

Two mechanisms. The first is the one everyone skips and it's the one that matters.

### 5.1 Blowdown (this is the "better cleaning" answer)

A consumer fountain recirculates the same 2 L of saliva-water for a week. That is the whole
problem. The industrial answer is the cooling-tower answer: **continuously bleed and replace.**

Evaporation removes pure water and leaves everything else behind, concentrating the bowl.
Cycles of concentration `C = TDS_bowl / TDS_makeup`, held by blowdown `B = E / (C − 1)`.

With E ≈ 0.5 L/day and a target of **C = 1.5** (very conservative; towers run 3–7):

```
B = 0.5 / (1.5 − 1) = 1.0 L/day
```

So ~1 L/day of blowdown holds bowl TDS at ≤1.5× tap, forever, with no filter involved.

Implementation: motorized 1/2" ball valve at the bowl's low point (2-wire CR-05 class, $25;
prefer over a solenoid — no coil heat, holds position on power loss, full-port so it doesn't
become a crevice). Firmware opens it, the bowl gravity-drains, level control refills.

Control law:
- **Adaptive:** trigger blowdown when `bowl_tds > 1.5 × makeup_tds_baseline`.
- **Floor:** force a full bowl dump every 24 h regardless.

The floor is not optional, because **conductivity is a poor proxy for saliva and biofilm** —
organic load barely moves TDS. Treat the conductivity trigger as an *extra* dump when the water
is genuinely concentrating, not as permission to skip the daily one. Being honest about that
limitation is the difference between an instrument and a decoration.

### 5.2 CIP (clean-in-place) — full procedure in docs/CIP.md

Design rules that make CIP possible; get these wrong at build time and no procedure saves you:

- **No dead legs.** Every branch/tee stub < 6 pipe diameters (< 3" on 1/2" line). Dead legs are
  where biofilm lives, because CIP flow never reaches them.
- **Self-draining.** Slope all horizontal runs ≥ 1/8" per ft toward the blowdown valve. No traps,
  no low points that hold 50 mL of last month's water.
- **Full-port ball valves only.** No gate, no globe — both have unswept internal cavities.
- **Minimise threads.** Threaded joints are crevices. Solder, tri-clamp, or compression.
- **Scouring velocity during CIP.** Sanitary practice wants ≥1.5 m/s and turbulent flow to
  mechanically shear biofilm. At 2 L/min in 1/2" line you get 0.22 m/s — far too slow. So:
  **PWM the pump.** Run 2 L/min normally (quiet, gentle for the dogs); command 100% for CIP:

  ```
  1/2" Type L:  ID 13.8 mm,  A = 1.51 cm²
  10 L/min  ->  v = 1.10 m/s
  Re = vD/ν = 1.10 × 0.0138 / 1.0e-6 ≈ 15,000   -> fully turbulent ✓
  ```
  This is why the pump spec below says ≥10 L/min even though you'll never serve water that fast.
- **Bowl is quick-release** — tri-clamp or gasketed bayonet — and goes in the dishwasher.
- **No PVC or vinyl tubing.** Plasticiser migration, and vinyl surface roughens and colonises.
  Hard-pipe everything; one short platinum-cured silicone section for vibration isolation only,
  replaced annually.
- Bowl in **316 SS, electropolished if you can get it (Ra < 0.8 µm)** — smoother surface, an
  order of magnitude less biofilm adhesion than scratched ABS.

---

## 6. Optional: hard-plumbed autofill (kills the refill chore entirely)

If there's a supply line within reach, tee off it and the reservoir never needs manual filling:

```
house cold --> [ full-port ball valve, manual ] --> [ NSF-61 backflow preventer / RPZ or AVB ]
           --> [ NC solenoid, 24VAC or 12VDC ] --> [ reservoir, top inlet above flood rim ]
```

Non-negotiables:
- **Backflow prevention.** An air gap (inlet terminating above the reservoir flood rim) is the
  simplest and most reliable, and satisfies most codes for a non-pressurised receptacle. Do not
  submerge the fill line. Ever.
- **Solenoid is normally-closed**, so a power cut closes it.
- **Independent hardware timeout.** Firmware bug = flooded house. Put a maximum-fill-duration
  watchdog in firmware *and* a mechanical float valve downstream, *and* the overflow standpipe.
  Three layers, at least one of them not made of software.
- Leak rope on the floor latches the solenoid closed and requires a manual reset.

Keep the manual fill port anyway. You want to be able to run the thing with the supply valve shut.

---

## 7. Instrumentation, control, alerting

### 7.0 Cloud-free mode (per your follow-up: yes, just a blinking light)

The entire alerting stack is optional. Tier 0 of notification is a **3-colour 12V stack light**
(or a single RGB beacon), driven directly by the ESP32 with no network in the path:

| State | Light | Meaning |
|---|---|---|
| Normal | green, steady | reservoir > 30%, bowl wet, pump & UV healthy |
| Advisory | amber, 0.5 Hz | reservoir < 30% (~3 days left) — refill when convenient |
| Warning | amber, 2 Hz | reservoir < 10%, or UV lamp > 8000 h, or filter ΔP high |
| Fault | red, 2 Hz | **bowl dry**, leak detected, pump stalled, or valve timeout |

This works with the router unplugged. Everything in §7.1–7.4 is additive telemetry on top.
If you build only this, you have already beaten the Kastty's blue→red LED by a lot.

### 7.1 Sensors

| Signal | Device | Why this one |
|---|---|---|
| `reservoir_liters` | 4× 50 kg half-bridge load cells + HX711 under the reservoir | **No wetted parts.** Mass→volume is geometry-independent, immune to foam/condensation, and it doubles as leak detection (mass loss with no dispense). 24-bit → ~10 g ≈ 10 mL practical resolution. |
| `bowl_level_low` / `_high` | 2× optical prism level switches | No current in the water. See rejected options below. |
| `makeup_liters_total` | Hall turbine flow sensor on makeup line | Cross-check against reservoir mass loss — divergence = leak. |
| `water_temp_c` | DS18B20 in a **dry thermowell** | Biofilm risk above ~25 °C; also density-corrects the mass→volume conversion. |
| `bowl_tds_ppm` | Analog TDS probe (AC-excited, e.g. DFRobot) | Blowdown control (§5.1). |
| `pump_current_a` | INA219 on the 12V pump feed | Dry run, clog, impeller failure — all distinguishable from current signature. |
| `uv_current_a` | INA219 or CT on the ballast | **"Relay closed" ≠ "lamp emitting."** Verify the actuator, not the command. |
| `leak_detected` | Water rope at drip-tray low point | Latching. Cuts makeup + pump, requires manual reset. |
| `filter_dp_kpa` | 2× pressure transducers across S2 (optional) | Turns filter service from a calendar into a measurement. |

Rejected: **bare DC conductivity electrodes for level.** They work, they're cheap, and they
electrolyse the water and plate probe metal into your dogs' drinking bowl. If you insist on
electrodes, AC-excite them. Optical is $8 and has none of the argument.

Rejected: ultrasonic reservoir level — condensation on the transducer face, and it can't see
through a stainless lid.

### 7.2 Controller

ESP32-S3 (or WROOM-32). Put it on **the IoT VLAN / `iot`** — it's an IoT device, treat it like one.
Power: 12V 5A brick → buck to 5V/3.3V. UV ballast is the only mains item: **on its own GFCI
outlet**, in its own compartment, physically separated from the low-voltage side.

Suggested pinout (ESP32-S3; avoids strapping pins 0/3/45/46 and USB 19/20):

```
GPIO4  HX711 DOUT        GPIO15 pump PWM (LEDC, N-MOSFET, flyback diode)
GPIO5  HX711 SCK         GPIO16 relay: UV ballast
GPIO6  1-Wire (DS18B20)  GPIO17 relay: makeup solenoid (NC)
GPIO7  flow pulse (pu)   GPIO18 motorized blowdown valve (2-wire, on/off)
GPIO8  TDS analog (ADC1) GPIO21 stack light: green
GPIO9  bowl level LOW    GPIO35 stack light: amber
GPIO10 bowl level HIGH   GPIO36 stack light: red
GPIO11 leak rope         GPIO12 service/reset button
GPIO13/14  I2C SDA/SCL (INA219 ×2)
```

Fail-safe defaults, all enforced in hardware or at boot, not by the happy path:
- Makeup solenoid **NC** — power loss closes it.
- Blowdown valve is a **motorized ball valve** — holds last position, so power loss doesn't
  silently drain the bowl.
- Pump off on watchdog reset; pump interlocked to `bowl_level_low` (no dry run).
- Max-fill-duration timer: if makeup has been commanded open > 90 s without `bowl_level_high`,
  close it, latch a fault, go red. Something is wrong; do not keep pouring.

### 7.3 Metrics & alerting

ESPHome ships a Prometheus exporter — the ESP serves `/metrics` directly, Prometheus on the homelab box
scrapes it, Alertmanager sends the email. See `prometheus/`.

**Split-brain caveat, since it's your day job:** if the homelab box is down, Prometheus cannot alert you
that the homelab box is down, and it also cannot alert you that the dogs have no water. The alert path must
not share fate with the monitored thing. So:
- Rich alerting (trends, prediction, health) → Prometheus/Alertmanager.
- The two alerts that actually matter — **bowl dry** and **leak** — also fire *from the device*,
  via a direct SMTP send or a webhook, plus the red stack light which needs no network at all.
- Device pushes a heartbeat to a dead-man's-switch (healthchecks.io or a local equivalent) so a
  dead ESP is itself an alert.

Headline alert rules (full set in `prometheus/dogfountain.rules.yml`):

```promql
# Threshold: ~3 days left
dogfountain_reservoir_liters < 17

# Better: predicted-empty, so lead time is constant regardless of consumption rate
predict_linear(dogfountain_reservoir_liters[12h], 48*3600) < 0

# The one that actually matters
dogfountain_bowl_level_low == 0   for: 10m   severity: critical

# Verify the actuator, not the command
dogfountain_uv_commanded == 1 and dogfountain_uv_current_a < 0.05
```

### 7.4 The feature that justifies the whole build

Once you're measuring reservoir mass continuously and subtracting evaporation and blowdown, you
have **daily water intake per household**. Sustained polydipsia is one of the earliest owner-
detectable signs of chronic kidney disease, diabetes mellitus, and hyperadrenocorticism in dogs —
often months before anything else is visible.

```promql
# >50% sustained increase in weekly intake vs the prior week
  increase(dogfountain_drink_liters_total[7d])
> 1.5 * increase(dogfountain_drink_liters_total[7d] offset 7d)
```

Route this one to email at `severity: info`, not to anything that pages. It is a **prompt to
book a vet appointment, not a diagnosis** — and with multiple dogs it's a household aggregate,
so it tells you *someone* changed, not who. Still: it is a real early-warning signal that a
$40 plastic bowl cannot give you, and it's the reason to instrument rather than just enlarge.

---

## 8. BOM (Tier 3, Variant A)

| Qty | Item | Est. $ |
|---|---|---|
| 1 | 15 gal Sanke keg or SS brew kettle + lid | 130 |
| 1 | 316 SS bowl, 13", electropolished if available | 35 |
| 1 | 12V BLDC magnetic-drive pump, ≥10 L/min @ 3 m, PWM | 45 |
| 1 | 10" Big Blue housing + washable pleated 20 µm | 62 |
| 1 | Inline UV-C 9–18 W + spare lamp | 65 |
| 10 ft | 1/2" Type L copper + fittings, lead-free solder & flux | 80 |
| 1 | Motorized ball valve 1/2" CR-05 (blowdown) | 25 |
| 1 | NC solenoid valve, NSF-61, 12V (makeup) | 35 |
| 1 | Low-pressure float valve, 0 psi rated (Variant B / backstop) | 20 |
| 2 | Full-port SS ball valves (service isolation) | 30 |
| 1 | SS strainer basket + tri-clamp bits, gaskets, silicone flex | 85 |
| 1 | 4× 50 kg load cells + HX711 | 15 |
| 2 | Optical level switches | 16 |
| 1 | Hall flow sensor | 12 |
| 1 | DS18B20 + SS thermowell | 14 |
| 1 | Analog TDS probe | 15 |
| 2 | INA219 current monitors | 12 |
| 1 | Leak rope + controller | 18 |
| 1 | ESP32-S3 devkit | 12 |
| 1 | 12V 5A PSU + buck converters + MOSFETs + relay board | 45 |
| 1 | 3-colour stack light, 12V | 28 |
| 1 | IP54 enclosure, DIN rail, terminal blocks, glands | 50 |
| 1 | Frame: 80/20 extrusion or welded SS square tube | 130 |
| 1 | SS drip tray, sloped, with lip | 35 |
| — | Copper test strips, PAA, PBW, gaskets | 45 |
| | **Total** | **≈ $1,059** |

Realistically $850–1,150 depending on how much you already have. Against a $40 Kastty. The
honest framing: you are not buying water delivery, you are buying **zero monthly consumables,
a two-week refill cadence, a bowl that is genuinely clean, and a health telemetry stream.**

---

## 9. Build notes / gotchas

- **Pump must be flooded-suction.** Magnetic-drive centrifugals are not self-priming. Mount the
  pump *below* the bowl water line or it will sit there spinning air.
- **Solder the copper before you mount anything.** Torch near a load cell, a TDS probe, or a
  silicone gasket will ruin your day.
- **Flush hard after soldering.** Flux residue is acidic and it's the #1 cause of early copper
  leaching in new work. Three fill/dump cycles, hot, before the passivation soak (§3.3).
- **Bowl at floor level.** Do not elevate it. Evidence in large/giant breeds associates elevated
  feeders with *increased* GDV risk. The aesthetics are not worth it.
- **Noise.** The pump isn't the problem, the splash is. Use a low weir and a laminar spout, not a
  jet. Rubber-isolate the pump and break structure-borne path with the silicone section.
- **Dogs will chew, lean on, and stand in this.** Everything under 24" must survive 40 kg of
  enthusiastic quadruped. No exposed low-voltage wiring, no glass sight tubes, cable in SS flex.
- **The reservoir mass reading drifts with temperature.** Auto-tare during a known-zero-flow
  window (e.g. 03:00, pump off, valves closed) and use the DS18B20 for density correction.
