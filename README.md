# Soil Buddy RE

Soil moisture and water-level monitor for an ESP32 board, with a phone web app (Web Bluetooth) and an optional ESP32-CAM view.

- **App** (`index.html`): installable web app. Shows the soil moisture index, the tank water level in cm, pump status, alerts, a terminal and the camera.
- **Firmware** (`firmware/SOIL_BUDDY_3.5/SOIL_BUDDY_3.5.ino`): v3.5 sketch for ESP32-WROOM. It reads the sensors, runs the pump and talks to the app over Bluetooth LE (Dabble Terminal protocol).

## Repo layout

```
index.html                    the whole app (one file)
sw.js                         service worker (offline cache, notification buttons)
manifest.webmanifest          install info
icon-*.png, apple-touch-icon.png, badge-96.png, logo-*.svg
firmware/SOIL_BUDDY_3.5/         Arduino sketch (folder name must match the .ino name)
.nojekyll                     tells GitHub Pages to serve files as they are
```

## Put the app online (GitHub Pages)

1. Create a repository and upload **everything in this folder** (drag the files and the `firmware` folder into "Add file > Upload files"). `index.html` must sit at the top level.
2. Open **Settings > Pages**. Under "Build and deployment" pick **Deploy from a branch**, branch **main**, folder **/ (root)**, then Save.
3. After a minute the app is at `https://<your-username>.github.io/<repo-name>/`.
4. Open that link in **Chrome or Edge** (Android, Windows, Mac, ChromeOS). Web Bluetooth needs HTTPS, which Pages provides. iPhone and Safari cannot use Web Bluetooth.
5. On Android, use **Setup > Install app** or the browser menu, Add to Home screen.

When you change a file, upload it again to the same place. The service worker is network first, so a refreshed page shows the new version. Bump the cache name in `sw.js` (`sb-re-11`) if an old copy sticks.

## Flash the board

1. Install the **DabbleESP32** library in the Arduino IDE and add ESP32 board support.
2. Open `firmware/SOIL_BUDDY_3.5/SOIL_BUDDY_3.5.ino`, pick an ESP32 Dev Module style board (ESP32-WROOM) and upload.
3. The board advertises over **Bluetooth LE** as "Soil Buddy RE". It does not show in the phone's normal Bluetooth list. Use the app's Connect button, the Dabble app or nRF Connect.
4. The board allows one phone at a time. Close the Dabble app before connecting from the web app.

Pins: soil sensor AO on GPIO35, HW-038 water sensor on GPIO34, pump through GPIO26 > 330 ohm > TIP120 base. The pump is OFF at boot and AUTO is ON at boot (`AUTO_AT_BOOT`); AUTO looks at the tank and soil right away.

## Water level in cm

The tank is about **4 cm high and 15 cm long**. The sketch turns the HW-038 ADC reading into a level in cm with a two-point map:

```
level_cm = (adc - WATER_ADC_AT_EMPTY) / (WATER_ADC_AT_FULL - WATER_ADC_AT_EMPTY) * 4.0
```

These are the measured values for this tank. To recalibrate:

1. Send `water` with the sensor dry and note `HW-038 ADC`. Put it in `WATER_ADC_AT_EMPTY`.
2. Fill the tank to the 4 cm mark, send `water` again, and put that ADC in `WATER_ADC_AT_FULL`.
3. Re-upload the sketch.

If the sensor reads lower when wetter, swap the two numbers and set `WATER_ADC_INCREASES_WITH_LEVEL` to `false` (that flag only feeds the low-water guard).

On this tank the sensor reads about 460 when fully dipped, so `WATER_ADC_AT_FULL = 460`. The AUTO low-water guard uses a fixed raw ADC cutoff (`WATER_LOW_ADC_THRESHOLD = 140`, about 1.2 cm, clears at 200). Make sure the pump intake is submerged above that level.

## Soil calibration

The soil index uses a two-point map set in the sketch: `SOIL_ADC_DRY = 4095` (probe in air, 0 %) and `SOIL_ADC_WET = 2095` (probe fully in water, 100 %). The small trimmer on the probe module only changes its digital output, not the analog reading the board uses. To recalibrate, send `soil` in air and in water and update the two constants.

## Commands (Terminal tab, Dabble Terminal or serial)

`help`, `soil`, `water` (or `cm`), `status`, `chat on` / `chat off`, `pump test`, `pump off`, `pump auto`, `threshold`

- `pump test` runs the pump for 30 s and **skips the water guard**. Only run it with the pump submerged and you beside it.
- `pump off` stops the pump and turns AUTO off.
- The app polls with the short command `s`. The board answers with one line:
  `#S,soilIndex,soilAdc,waterCm,waterAdc,mode,pump,demand,guard`
  (mode 0 = off, 1 = auto, 2 = timed test; pump, demand and guard are 1 or 0).

## How AUTO works (v3.5)

1. AUTO is on at boot and looks right away.
2. Soil index at or below `SOIL_START_INDEX` (30 %) and the tank is OK: pump ON, continuous, no pulses.
3. It stays on until soil reaches `SOIL_STOP_INDEX` (50 %), the tank interlock trips, or `MAX_RUN_MS` (120 s) runs out.
4. The tank is checked every 5 s and the soil every 10 s. The tank interlock trips after 2 low readings in a row (about 10 s), stays blocked until the HW-038 reads above 200, and then AUTO restarts by itself.
5. After a 120 s run the pump rests 60 s. If two full runs in a row raise the soil index by less than 3 points, AUTO pauses itself and says so. `pump auto` clears the pause.

Every PUMP ON / PUMP OFF line in the Terminal tab gives the reason and the soil % / tank ADC at that moment.

## Known limits

- Soil % is a relative ADC index, not a calibrated moisture value. Water cm is an estimate from a resistive sensor, which is not truly linear with depth.
- The soil thresholds (30 start, 50 stop), the 120 s max run / 60 s rest and the 140 / 200 ADC cutoffs are pilot settings. Validate them on your own plant and soil.
- Alerts only work while the app is open and connected. Closing it disconnects Bluetooth.
- The camera tab expects an ESP32-CAM stream at `http://<ip>:81/stream`. A page served over HTTPS may be blocked from loading an `http://` stream. If so, open the stream link shown under the camera field, or run the app from `http://localhost`.
- Web Bluetooth works in Chrome and Edge only (not iPhone, not Safari).
