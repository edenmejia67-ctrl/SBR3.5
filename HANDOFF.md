# Soil Buddy RE: handoff for adjusting sensitivity

State as of 2026-11-09. Sketch v3.5, app v3.1 (note text updated). Everything below is edited in `firmware/SOIL_BUDDY_3.5/SOIL_BUDDY_3.5.ino` unless it says "app".

## Current calibrated values

| What | Constant | Now | Meaning |
|---|---|---|---|
| Soil probe, dry | `SOIL_ADC_DRY` | 4095 | probe in open air = 0 % |
| Soil probe, wet | `SOIL_ADC_WET` | 2095 | probe fully in water = 100 % |
| Water sensor, empty | `WATER_ADC_AT_EMPTY` | 0 | dry sensor = 0 cm |
| Water sensor, full | `WATER_ADC_AT_FULL` | 460 | fully dipped = 4.0 cm (read 454 to 466) |
| Tank height | `RESERVOIR_HEIGHT_CM` | 4.0 | tank is about 4 cm high x 15 cm long |
| Low-water cutoff | `WATER_LOW_ADC_THRESHOLD` | 140 | AUTO blocked below this (about 1.2 cm) |
| Low-water recovery | `WATER_RECOVERY_ADC_THRESHOLD` | 200 | guard clears above this (about 1.7 cm) |

## How to re-measure (any time)

1. Connect the board (app Terminal tab, serial monitor or Dabble Terminal).
2. **Soil:** send `soil`, read "Raw ADC". Probe in air = dry value. Probe fully in water = wet value. Put them in `SOIL_ADC_DRY` and `SOIL_ADC_WET`.
3. **Water level:** send `water`, read "HW-038 ADC". Sensor dry = `WATER_ADC_AT_EMPTY`. Tank filled to the 4 cm mark = `WATER_ADC_AT_FULL`.
4. Re-upload the sketch. Send `water` and `soil` again to confirm the % and cm look right.

## "Sensitivity" knobs, and what each one really does

- **The trimmer on the soil probe module does nothing here.** It only sets the module's digital output (DO). The board reads the analog output (AO) on GPIO35.
- **Make soil react over a narrower or wider range:** move `SOIL_ADC_DRY` and `SOIL_ADC_WET` closer together (more sensitive) or further apart (less sensitive). Measure the probe in your real pot soil, dry and just watered, and use those two numbers instead of air and water.
- **When AUTO starts and stops watering:** `SOIL_START_INDEX` (now 30 %, a placeholder) starts the pump and `SOIL_STOP_INDEX` (now 50 %) stops it. Real dry soil may read anywhere from 0 to 50 %, so set the start value just above your measured dry-soil reading. The soil is looked at every `SOIL_CHECK_MS` (10 s).
- **How long it pumps (v3.5, continuous):** there are no pulses any more. `MAX_RUN_MS` (120 s) is the longest single run, `REST_MS` (60 s) the rest after it. `NO_EFFECT_MIN_RISE` (3 soil points) and `MAX_NO_EFFECT_RUNS` (2) pause AUTO if the soil never responds.
- **Low-water guard:** `WATER_LOW_ADC_THRESHOLD` and `WATER_RECOVERY_ADC_THRESHOLD`. Keep the cutoff a little above the height of the pump intake, and the recovery value higher than the cutoff. The tank is checked every `TANK_CHECK_MS` (5 s) and trips after `WATER_LOW_CONFIRM` (2) low readings in a row.
- **App alerts** (Setup tab, saved on the phone): "Dry alert below" (now 30 %) and "Tank low below" (now 1.5 cm, keep it above the guard cutoff of about 1.2 cm).

## Known facts and open items

- The water sensor (HW-038) reads about 0 when dry and only about 460 when fully dipped, so its cm scale is short. A resistive sensor is not linear with depth.
- The 140 / 200 cutoffs are a guess. **Open:** measure the pump intake height above the tank floor and set the cutoff from it.
- **Open:** for the cm accuracy test (SOP 3), read the water ADC at about 1, 2, 3 and 4 cm. If the points are not on a straight line, replace the straight-line map with a lookup table.
- **Open:** set `SOIL_START_INDEX` and `SOIL_STOP_INDEX` from real readings in the chili pot (dry soil and just-watered soil).
- Soil reading exactly 4095 (0 %) can also mean the probe is unplugged.
- Water ADC exactly 0 while dipped means a wiring or sensor fault (it was a bad sensor once).
- `pump test` runs 30 s and skips the water guard. `pump off` also turns AUTO off. AUTO is ON at every boot (`AUTO_AT_BOOT`). A pump test hands AUTO back when it ends, a low-water trip only pauses it until the tank is refilled, and only `pump off` turns it off until the next `pump auto` or reboot.
- Upload the sketch and the app together. The data line is `#S,soilIndex,soilAdc,waterCm,waterAdc,mode,pump,demand,guard`.
- Web Bluetooth needs Chrome or Edge on HTTPS (GitHub Pages). The board allows one phone at a time, so close the Dabble app first.

## After editing the app on GitHub

Upload the changed file to the same repo. If a phone still shows the old version, bump the cache name in `sw.js` (`sb-re-11`) to the next number.
