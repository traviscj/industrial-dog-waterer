# Tier 1/2 — the livestock-waterer build (what I'd actually build)

You linked a Cowboy-style 5 gal gravity waterer. You're right, and it's worth saying plainly:
**that plus a drain solves most of the brief for ~$70.**

## What the gravity waterer already gives you

| Your requirement | Gravity waterer | Kastty FS55 |
|---|---|---|
| larger tank | 5 gal (19 L) = 2.6× | 2 gal |
| surplus tank, refillable | that's literally what it is | no |
| less frequent filter changes | **there are no filters** | monthly pads |
| better cleaning | one HDPE part, hose it out, no pump to disassemble | 6 parts + a pump impeller |
| copper pipes | no | no |
| email when low | no | blue→red LED |

It wins on four of six by *deleting components*, which is the correct kind of engineering.

## What it doesn't give you, honestly

1. **No moving water.** Some dogs drink more from a moving source; cats strongly prefer it. If
   yours don't care, this costs you nothing. If they do, you need Tier 3's recirc loop.
2. **No disinfection.** A gravity trough grows biofilm at the waterline — the slimy ring. The
   drain is what fixes this, by making a flush trivial instead of a chore.
3. **5 gal ≈ 4 days** for two 30 kg dogs. Same order as the Kastty, not a step change. Get the
   10 gal version if it exists in that line, or park a second unit.
4. **Opaque tank, no telemetry.** Which is what Tier 2 adds.

## Tier 1: waterer + drain — $70, one afternoon

- Drill/tap the trough at its lowest point, 1/2" NPT bulkhead → full-port ball valve → barb.
- Run reinforced hose to: floor drain, utility sink, washer standpipe, or a condensate pump.
  Slope it. No traps. If you can't reach a drain, run it to a 20 L jug you tip into the sink.
- Weekly, 30 seconds: open the valve, hose the trough out, close it. Done. No filters, ever.
- Optional: put the whole thing on a sloped SS drip tray so splash-out also goes to the drain.

That's it. That's the whole build. It's a genuinely good answer.

## Tier 2: add the blinking light — +$60

Bolt the telemetry on without touching the hydraulics.

| Part | $ | Purpose |
|---|---|---|
| ESP32-S3 devkit or ESP32-C3 | 8 | controller, Wi-Fi to the IoT VLAN / `iot` |
| 4× 50 kg load cell + HX711 | 15 | put the waterer on them → `reservoir_liters` |
| 12V 3-colour stack light (or RGB beacon) | 28 | **the blinking light — no cloud in the path** |
| Optical level switch in the trough | 8 | `bowl_dry` — the alert that actually matters |
| 5V PSU + enclosure + wire | 15 | |

Load cells under the reservoir are the right sensor: nothing touches the water, so nothing
fouls, and mass→volume works regardless of tank shape. You also get consumption rate for free,
which is where the health signal comes from (DESIGN.md §7.4).

Light behaviour — identical to Tier 3, so firmware carries over:

| State | Light |
|---|---|
| > 30% | green steady |
| < 30% (~3 days) | amber, 0.5 Hz |
| < 10% | amber, 2 Hz |
| trough dry, or tank mass dropping with no drinking pattern (leak) | **red, 2 Hz** |

Email is then purely optional — point Prometheus at the ESP's `/metrics` if and when you feel
like it, and the light keeps working whether or not you ever do.

## The upgrade path is not wasted

Tier 2's load cells, HX711, ESP32, stack light, and firmware all move to Tier 3 unchanged.
Build Tier 2 now, live with it for a month, and you'll know from data whether you actually
want the copper loop — including whether the dogs drink more when the water moves.
