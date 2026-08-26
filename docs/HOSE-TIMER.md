# Automating the Cowboy's self-cleaning waterer

The 5 gal Cowboy's waterer has no pump, no filter, and no electronics. It is a bucket with an
inlet that does two things when you turn the water on:

- a jet scours the bottom, lifting settled debris;
- the rising level pushes the surface film — hair, pollen, slobber, mosquito larvae — over an
  overflow and out.

So "self-cleaning" means **flush-on-fill**. There is no cleaning cycle to trigger; there is only
"water is flowing" and "water is not flowing." Everything you automate is therefore just: *turn
the hose on for one minute, N times a day.* A hose timer is genuinely the whole solution.

A built-in check valve prevents backflow into the hose and lets you disconnect and use it as a
plain pail — which matters more than it sounds like, see freezing below.

## What people actually run

Straight from the reviews on the product page, because real schedules beat theory:

| Reported schedule | Result reported |
|---|---|
| 1 min every 2 h (12×/day) | "keeps the water looking fresh and cool"; still does a full manual dump every other day |
| every 4 h (6×/day), 3 large dogs, Georgia | "dogs are happy… I don't worry they'll be out of water" |
| 2 min × 3/day | "water stays clean and cool" |
| 1 min × 2/day | "cleans itself very very well" |
| 3 min × 3/day | **still got algae** |

Two things fall out of that. First, **1-minute runs are the norm**, so minimum run time is the
spec that decides which timer you can buy. Second, the person who ran the *most* water per day
(9 min/day) is the one who got algae, and the person running 12 min/day didn't. Duration isn't
the variable — **light is**. Algae needs photons. Put it in shade before you turn the schedule up.

## Buy on three specs, not on brand

1. **Minimum run time ≤ 1 minute.** Non-negotiable. Some timers floor at 5 minutes, which
   quintuples your water bill for no benefit.
2. **Interval scheduling — "every N hours" — not "N start times per day."** This is the one that
   quietly bites. Several popular smart timers cap at ~4 start times per program, which maxes you
   at 4 flushes/day unless you stack programs. A dumb interval dial does every-2-hours natively.
3. **Working pressure range covers your bib.** Typical hose timers want roughly 10–100 psi;
   some newer ones go down to 5. Municipal supply is fine. **Gravity or a rain barrel is not** —
   these are diaphragm valves and they need line pressure to seat.

Then, in rough order of how much you'll enjoy owning it:

| | What | ~$ | Why / why not |
|---|---|---|---|
| **A** | **Mechanical/digital interval dial** (Orbit two-dial class): set duration, set "every N hours", done | 30–40 | Does exactly the one job. No app, no account, no firmware. **Start here.** |
| **B** | Smart Wi-Fi/BT timer (Orbit B-hyve XD, Rachio) | 50–100 | Remote schedule changes and a "did it run" log. Check the start-times cap before buying. Rachio uses a piston valve rather than a ball valve, which passes more flow. |
| **C** | **DIY: ESP32 + irrigation solenoid** | ~35 | The only option that tells you it *actually opened*. Details below. |

## Option C — the one that fits the rest of this repo

A hose timer's failure mode is silent: the battery dies, the valve stops opening, and you find
out when the bucket is empty. Adding $20 of parts fixes that and drops straight into the ESPHome
config already in this repo.

```
  hose bib ──[ vacuum breaker ]──[ ¾" NC irrigation solenoid ]──[ flow sensor ]── waterer
                                          ▲                            │
                                 24 VAC via relay                      │ pulses
                                          └──────── ESP32 ─────────────┘
```

- **Valve:** any ¾" jar-top irrigation solenoid (Orbit/Rain Bird, ~$12). Normally closed, so a
  power cut or a crashed ESP leaves it shut. These are the same valves that sit in a million
  buried sprinkler boxes for a decade.
- **Drive:** 24 VAC transformer + relay. Needs mains at the bib. If you'd rather run on
  battery/solar outdoors, use a 12 V **latching** solenoid (the kind inside commercial hose
  timers) with a DRV8833 H-bridge — it draws current only during the ~50 ms open/close pulse.
- **Verification:** the flow sensor is the whole point. Command the valve open, and if the
  pulse counter doesn't move within 5 seconds, that's a fault: no supply, dead valve, kinked
  hose, closed bib. Now the failure is loud instead of silent.
- **Bonus:** the totalizer gives you actual gallons per day instead of a guess, which is what
  you need for the water-bill decision below. And a stuck-*open* valve — the expensive failure —
  shows up immediately as flow that doesn't stop.

```yaml
# add to esphome/dogfountain.yaml
switch:
  - platform: gpio
    name: "flush valve"
    id: flush_valve            # NC solenoid: de-energised = closed
    pin: GPIO19
    restore_mode: ALWAYS_OFF

script:
  - id: flush_cycle
    mode: single
    then:
      - switch.turn_on: flush_valve
      - delay: 5s
      - if:                     # verify the actuator, not the command
          condition:
            lambda: 'return id(flush_flow).state < 0.5;'
          then:
            - switch.turn_off: flush_valve
            - lambda: 'id(fault_latched) = true; ESP_LOGE("flush","valve commanded, no flow");'
      - delay: 55s
      - switch.turn_off: flush_valve

interval:
  - interval: 4h
    then:
      - if:
          condition: {lambda: 'return !id(fault_latched);'}
          then: [script.execute: flush_cycle]
```

## The gotchas

### Freezing — the big one outside the Sun Belt

Every reviewer raving about this is watering dogs in Georgia. A hose bib timer is a plastic body
full of standing water; Orbit states plainly that **freeze damage is not covered under warranty**,
and a split valve body at 3 a.m. in January is a bad night. In a cold climate this is a
**seasonal system**:

- **April–November:** hose connected, timer running, as above.
- **First hard freeze:** shut the bib, drain and store the timer indoors, disconnect the hose.
  The built-in check valve means the waterer keeps working as an ordinary pail — this is exactly
  what that feature is for.
- **Winter:** fill by hand, or move to a heated bowl. If the dogs are outdoors in freezing
  weather anyway, you need a heated bowl regardless; a flushing waterer doesn't solve ice.

If you want year-round automation in a cold climate, that's a different build: a frost-free
hydrant, a buried supply line below the frost depth, and a heated bowl — real money and a trench.

### Backflow

The waterer has a check valve, and that is not the same thing as a backflow preventer for a
cross-connection. A hose sitting in an animal watering vessel is a textbook cross-connection, and
most plumbing codes already require a **hose bib vacuum breaker** on every outdoor spigot. It's
$8 and it threads on in ten seconds. Fit one.

### Water use — measure it once, then decide

Nobody publishes the flow rate, and it depends on your line pressure, so measure: put a bucket
under the overflow, run one minute, weigh or measure what comes out. Then:

| Flush flow | 1 min × 2/day | 1 min × 6/day | 1 min × 12/day |
|---|---|---|---|
| 1.5 GPM | 90 gal/mo | 270 gal/mo | 540 gal/mo |
| 3.0 GPM | 180 gal/mo | 540 gal/mo | 1,080 gal/mo |
| 5.0 GPM | 300 gal/mo | 900 gal/mo | 1,800 gal/mo |

Not free, and in many places **sewer is billed off the water meter**, so outdoor use can get
charged twice unless your utility offers an irrigation deduct. **Start at 1 min every 6 h and
tune up** only if the water actually looks bad. Every-2-hours is a hot-Georgia-afternoon setting,
not a default.

### Where the overflow goes

It runs onto the ground, continuously, at the volumes above. One reviewer routes it to their
garden beds, which is the right instinct. Put the waterer where the discharge helps something —
and **not within 10 ft of the foundation**, especially over clay. A few hundred gallons a month
against a basement wall is a genuinely bad idea.

### Algae, again

3 min × 3/day still grew algae for one owner. Flushing controls *film and debris*; it does not
control algae, because algae is a light problem. In order of effect: move it to full shade, keep
the bucket opaque, then increase flush frequency. If it's in direct sun, no schedule will save it.

### The failure mode is a silent one

A dead battery or a stuck valve means an empty bucket, and 5 gal of buffer means you may not
notice for a day. This is precisely what the Tier 2 build in this repo is for: load cells under
the waterer, a stack light, and no cloud in the path. Option C above merges the two into one
controller — it opens the valve *and* watches the mass, so "the flush ran but the bucket didn't
fill" is a detectable state rather than a surprise.
