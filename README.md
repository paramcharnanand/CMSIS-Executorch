# Sentinel — learned "normal state" monitor on the Alif Ensemble E8

Sentinel runs on the Alif Ensemble E8 DevKit. It learns how one machine
normally sounds, keeps that baseline in on-chip MRAM, and shows on the LCD
whether the machine still matches it.

- **Demonstrated:** a desk fan changing from its learned medium speed to
  high speed (CHECK), and back (OK).
- **Not demonstrated:** fault diagnosis.

Built on Arm's CMSIS-ExecuTorch pico-faces example (Apache-2.0). The original
guide is in [docs/pico-faces-guide.md](docs/pico-faces-guide.md). Team: Param
Charnanand, Arya Varia.

## Hardware actually used (Cortex-M55 HP core)

| Peripheral | Use |
|---|---|
| PDM microphones (channels 4/5) | 16 kHz audio into a 4 x 100 ms interrupt-fed ring |
| BMI323 accelerometer (I3C) | Read at the driver's fixed rate of about 50 Hz |
| MIPI-DSI LCD | Landscape text status |
| MRAM | Two baselines (FAN, PHONE) in the last 256 bytes of user MRAM |
| SW2 joystick | Select a profile and learn |
| UART4 (115200 8N1) | One status line per second |

The live Sentinel mode uses **statistical scoring on the M55 CPU**: under
0.2 % of one core. It does **not** run ExecuTorch or the Ethos-U85 NPU. The
pico-faces ExecuTorch/NPU code remains in the repository but is not linked in
this mode.

## How it works

1. **Windows:** every 1 s window combines ten 100 ms audio blocks with about
   50 accelerometer reads.
2. **Features:** four per window:
   - log sound power of each microphone channel;
   - a high/low-frequency ratio;
   - log variance of the acceleration magnitude.
3. **Learning:** SW2 up records 30 windows (30 s) and stores each feature's
   mean and standard deviation in MRAM. The save is checksummed and read back.
4. **Scoring:** each window gives a z-score per feature against that baseline.
   - A sensor counts as changed after 3 windows in a row with |z| > 4, and as
     normal again after 3 windows with |z| < 2.5.
   - No sensor changed = **OK**, one = **CHECK**, both = **ALERT**.

## Run the FAN demo

1. Build and flash. A fresh clone first needs the "Create AI layer" step from
   the [pico-faces guide](docs/pico-faces-guide.md):
   ```
   cbuild cmsis-executorch.csolution.yml --active DevKit-E8 --rebuild
   JLinkExe -device AE822FA0E5597LS0_M55_HP -if swd -speed 4000   # LoadFile the .hex, then reset
   ```
2. Set up:
   - fan guard on, oscillation off;
   - board beside the fan, out of its airflow;
   - both positions fixed;
   - fan on medium for about 10 s, room quiet.
3. **SW2 left** selects FAN. **SW2 up** learns: blue LEARNING for 30 s, then
   green **OK**.
4. Switch the fan to **high**: amber **CHECK**, `SOUND: CHANGED`, about 2 s
   after the sound changes.
5. Switch back to **medium**: green **OK** again.

## What was demonstrated (2026-10-17/18, on the board)

- Medium → high gave CHECK 2 s after the sound changed, and back gave OK,
  repeatedly. An operator confirmed it on the LCD.
- Baselines saved to MRAM and reloaded after full power cycles.
- 0 audio FIFO errors and 0 ring-full stops in every log.
- About 0.2 ms of CPU per 100 ms audio block at worst.
- Logs can be analysed offline with `tools/sentinel_log_report.py`.

## Main limitations

- **Fault diagnosis:** not shown. It reports a *change* from the learned
  normal state; OK is not a health verdict.
- **Fan vibration:** not validated. The installed BMI323 driver gives about
  50 Hz samples, and fan speeds stay at the noise floor.
- **Phone vibration:** inconclusive. No loom or real machine fault was tested.
- **Sensitivity to the room:** voices or bumps near the board cause short
  CHECKs, and moving the board after learning needs a relearn.
- **Power:** not measured.

## More

- [docs/sentinel-details.md](docs/sentinel-details.md): architecture,
  measured performance and memory, build modes, test records, troubleshooting.
- `board/DevKit-E8/`: Sentinel sources
  - `sentinel.c` (scoring and LCD), `mic_probe.c`, `imu_probe.c`
  - `sentinel_profile_store.c` (MRAM profiles)
- `tests/`: host tests of the profile store
- `tools/`: offline log report
