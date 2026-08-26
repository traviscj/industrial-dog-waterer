# CIP — clean-in-place procedure

Applies to Tier 3. Tier 1/2 is "open the drain valve and hose it out," which is the point of
Tier 1/2. Target cadence: **monthly**, or when `bowl_tds` or `filter_dp` trends say otherwise.

## Chemistry selection

| Agent | Use | Copper compatible | 304/316 SS | Notes |
|---|---|---|---|---|
| **PBW** (percarbonate + metasilicate) | soil removal / cleaning | yes at ≤ 60 °C, short contact | yes | will darken copper on long soaks; don't leave it overnight |
| **Peracetic acid, 100–200 ppm** | sanitising | yes at use dilution | yes | dairy/brewery standard; decomposes to acetic acid + O₂ + H₂O; no residue |
| Caustic (NaOH) CIP | — | **NO** — attacks copper | yes | do not use |
| Bleach / hypochlorite | — | **NO** — corrodes copper, pits SS (chlorides) | no | do not use |
| StarSan (phosphoric + DDBSA) | acceptable, short contact | acid dwell → copper leaching | yes | rinse it; do not leave it in the loop |
| Hot water ≥ 77 °C, 15 min | sanitising, zero chemicals | yes | yes | best option if you have a way to heat the loop |

**Never mix.** Alkaline cleaner and acid sanitiser are separate steps with a rinse between.

"No-rinse" claims are written for human food equipment and are legitimate — but for the dogs,
rinse anyway. Water is cheap and it removes the entire argument.

## Procedure (~45 min, mostly unattended)

1. **Lock out.** Service mode in firmware: pump off, makeup solenoid closed, UV off, blowdown
   closed. Confirm the stack light shows service (amber steady).
2. **Drain.** Open blowdown, let the bowl and loop gravity-drain. Confirm reservoir mass is
   unchanged — if it drops, the makeup valve is passing and needs service.
3. **Strainer + bowl.** Pull the SS strainer basket, rinse. Release the bowl (tri-clamp), run it
   through the dishwasher on the hot cycle.
4. **Filter.** Pull the pleated cartridge, hose it from the clean side out, inspect for tears,
   reinstall. Replace the housing O-ring annually and lube with food-grade silicone grease.
5. **UV sleeve.** Power down, GFCI off, pull the quartz sleeve, wipe with isopropyl and a soft
   cloth (never abrasive). **This is the step that keeps UV working** — output loss is almost
   always fouling, not lamp age. Reset `uv_lamp_hours` only when you actually change the lamp.
6. **Clean cycle.** Reinstall everything, close blowdown, fill the bowl with warm PBW at label
   dilution. Firmware `CIP` mode: pump to 100% (≈10 L/min → 1.1 m/s → Re ≈ 15,000, turbulent —
   this is the mechanical shear that removes biofilm; a slow rinse does not). Recirculate 15 min.
7. **Rinse.** Drain fully. Fill with potable water, recirc 5 min at 100%, drain. Repeat twice.
8. **Sanitise.** Fill with 100 ppm peracetic acid. Recirc 10 min at normal flow. **Verify with a
   PAA test strip** — mixing by volume and hoping is not verification.
9. **Final rinse.** Drain. One full fill/recirc/drain with potable water. Do not leave acid in
   the copper.
10. **Return to service.** Exit service mode. Watch the first fill complete and confirm
    `bowl_level_high` asserts before the 90 s makeup timeout. Green light.

## Schedule

| Interval | Task |
|---|---|
| Daily (automatic) | bowl blowdown + refill |
| Weekly | rinse strainer basket; wipe bowl waterline; visual leak check under the drip tray |
| Monthly | full CIP above |
| Quarterly | hose the pleated cartridge; wipe UV sleeve; check reservoir tare drift |
| 6 months | copper test strip on a first-draw sample — target < 0.3 mg/L, act at > 0.5 |
| 12 months | UV lamp; all gaskets/O-rings; silicone flex section; reservoir deep clean |

## Commissioning (once, before any dog drinks from it)

1. Solder all copper. Flush hot, three fill/dump cycles — flux residue is acidic and is the main
   cause of early copper leaching in new work.
2. Full CIP as above.
3. **Passivation soak: 72 h of recirculation on tap water, dumping and refilling daily.** Lets
   the protective patina form on your schedule instead of in your dogs.
4. Copper test strip. Must read < 0.3 mg/L before the system goes into service.
5. Leak test: run 24 h with the leak rope armed and nothing drinking. Reservoir mass loss should
   equal blowdown volume plus evaporation, and nothing more. Any excess is a leak — find it now.
