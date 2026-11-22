# Sovol SV08 Max – config backups & notes

Working repo for a Sovol SV08 Max (Klipper). Holds config/macro backups, notes for future agents, and misc printer files.

## Printer access
- Host `SPI-XI`, IP `10.88.20.44` (DHCP – may change), SSH user `sovol` (factory default credentials; **change them** and prefer key auth).
- Mainsail/Moonraker: `http://10.88.20.44` / `:7125`. Config dir: `~/printer_data/config/`.
- `sudo` works with the same password. No `sshpass`-free access yet.
- Tip: `tar ... | tar` over `sshpass ssh` returned nothing here; use `rsync -e ssh` instead.

## Layout
- `backup/YYYY-MM-DD/printer_data_config/` – full `~/printer_data/config` (printer.cfg, Macro.cfg, plr.cfg, buffer_stepper.cfg, chamber_hot.cfg, moonraker.conf, crowsnest.conf, …). Symlinks (mainsail.cfg, timelapse.cfg, obico macros) are preserved as symlinks and dangle locally.
- `backup/.../patch/` – Sovol's factory patch dir. **`factory_resets.sh` copies `~/patch/config/*.cfg` over `printer.cfg` etc. and restarts firmware** – that is the factory-reset source of truth.
- `backup/.../pyhelper/`, `home_scripts/`, `systemd_env/` – Sovol helper scripts, PLR (power-loss recovery) state, env files.
- `backup/.../system/system_info.txt` – rc.local, fstab, apt sources, service unit files, nginx, package list, upgradable list, network, sshd config.
- `bique feeder files/` / `btt_feeder*.cfg` – BTT feeder config files supplied by the owner.
- **Not backed up on purpose:** `~/printer_data/database/moonraker-sql.db` (contains API key + password hashes), logs, gcodes, git working trees of klipper/moonraker/etc.

## Baseline state (2026-10-05)
- OS: SPI-XI 2.3.3, Debian 11 bullseye (Armbian-derived), kernel 5.16.17-sun50iw9 (Allwinner H616), aarch64. Root fs 29G, 4.5G used.
- Klipper `d4031b3-dirty` (Sovol fork, remote is an internal `http://192.168.1.233/root/klipper.git` – not reachable; 'dirty' = tracked .pyc changes only). Moonraker `v0.9.3-1-g4e00a07` (upstream). KlipperScreen, crowsnest, Mainsail, moonraker-obico, moonraker-timelapse installed.
- MCUs: main `stm32h750xx` fw `aeb4421-dirty-20250402`; `extra_mcu` `stm32f103xe` fw `cc8afd8-dirty-20250310`. Host is a CAN bridge (CANBUS_BRIDGE).
- Moonraker update manager is not configured (`/machine/update/status` 404) – updates are Sovol-OTA/manual only.

## OS update log
- **2026-10-05:** `apt-get upgrade` applied (123 security packages: libc, sudo, curl, gnutls, git, polkit, wpa_supplicant, etc.), then rebooted. Kernel unchanged (5.16.17-sun50iw9); Klipper, Moonraker, nginx, KlipperScreen, crowsnest all came back `active`, printer `ready`, 0 failed units, 0 upgradable.
- **Debian 11 LTS has ended** (Aug 2026). Security packages now 404 on `deb.debian.org/debian-security`; they live on `archive.debian.org/debian-security`. `/etc/apt/sources.list` was changed accordingly (`bullseye-backports` commented out, security line -> archive.debian.org; original saved as `/etc/apt/sources.list.bak-20261005` on the printer; current copy in `backup/2026-10-05/system/sources.list.after-upgrade`). Expect **no further security updates** – the OS is effectively frozen. Do not attempt a release upgrade (vendor-customised image); rely on network isolation instead.
- Harmless warning during upgrade: `ln: failed to create hard link /boot/initrd.img-...dpkg-bak` because `/boot` is FAT. initrd/uInitrd regenerated fine.
- Still TODO: change default `sovol` password, add SSH key auth.

## Eddy sensor failure -> nozzle-into-bed crashes (Oct 2026, warranty)
**Symptom:** during start-of-print Z-offset calibration, `G28 Z` or QGL the nozzle is driven into the bed; the four Z motors keep trying to move and torque the gantry until the e-stop. Intermittent (about 4 incidents in roughly 30 prints). Printer was bone stock. Damaged the PEI sheet.
**Cause (from `klippy.log`):** the toolhead eddy sensor (LDC1612, `probe_eddy_current eddy`) stops producing valid data. The first status line is `report ldc1612 i2c register: cr1_data=1025 cr2_data=36 sr1_data=512 sr2_data=2 err_code=36` (once also preceded by `cr1_data=1 sr1_data=1024 err_code=34`), then `cr1_data=257 sr1_data=0 err_code=36` repeats thousands of times. On this machine the eddy sensor **is** the Z endstop (`endstop_pin: probe:z_virtual_endstop`), so with no data it never triggers and the gantry keeps descending (only `position_min: -10` stops it). Healthy sessions log no `ldc1612 i2c register` lines at all. Related Klipper errors: `Unable to obtain 'sensor_bulk_status' response`, `probe_eddy_current sensor outage`.
**Timeline (device clock, UTC; printer has no RTC so times are approximate):**
- ~2026-10-07: start-of-print calibration probed `z=510.31` then `z=499.99` at bed centre, then the sensor error loop (log since rotated off the printer; only chat excerpts remain).
- 2026-10-10 14:47:45: manual `G28` with the toolhead 400+ mm up; sensor error loop 1,589 lines, e-stop (`Klippy has shutdown`, `Homing failed due to printer shutdown`).
- 2026-10-10 (later): bed lowered to ~35 mm, power cycled, homed again, jammed again (no klippy log survived).
- 2026-10-13 ~13:49:57: failure during QGL, 15,052 error lines, `probe_eddy_current sensor outage`. Nozzle target 225 C, bed target 0 (cold bed ~25.8 C). Forum posts report similar crashes with a cold bed; unproven here.
**Ruled out:** ribbon cable reseated at both ends (user-verified), gantry level/mechanics (log points at sensor data), homing height (lowering the bed did not prevent a repeat).
**Evidence (kept outside the repo, large):** `/Volumes/4TB-NVME/comma/Claude Artifacts/SV08Max-warranty-logs-2026-10-12/` (+ `.zip`) and `.../SV08Max-warranty-logs-2026-10-13-QGL-failure/` (+ `.zip`); email draft: `.../SV08Max-sovol-support-email-draft.html`.
**Outcome (2026-10-13):** Sovol support confirmed they will ship a replacement eddy sensor and a PEI sheet. Keep the old sensor/cable in case Sovol wants it back.
**After the new sensor is fitted:**
- Redo eddy calibration per Sovol's replacement instructions (drive current + Z offset; Klipper commands are `LDC_CALIBRATE_DRIVE_CURRENT` / `PROBE_EDDY_CURRENT_CALIBRATE`, verify before use).
- First Z home + QGL with a hand on the e-stop, then check `klippy.log` for any `ldc1612 i2c register` lines (there should be none) and sane probe values (a few mm near the bed, not hundreds).
- Re-check first layer / Z offset (new PEI thickness may differ).
- **Until then:** no unattended Z home or print start; pull `klippy.log` (it rotates at ~15 MB, roughly daily) before any restart if it fails again.
- Upgrade-phase idea (not applied): a firmware guard that aborts when the probe reads absurd heights or Z homing travels too far without a trigger, so a dead sensor errors instead of crashing.

## Wi-Fi reconnect loop fix (2026-10-13)
**Symptom:** Wi-Fi dropped and re-joined roughly every 13 s (`wpa_supplicant` `CTRL-EVENT-DISCONNECTED reason=3 locally_generated=1`), Mainsail kept disconnecting, and Moonraker's detected IP flipped between `.44`, `.48` and 169.254.x.x. Changing 5 GHz -> 2.4 GHz SSID made no difference; it is not a signal/antenna/AP problem.
**Cause (a Sovol image bug, not RF):**
- Both NetworkManager and `dhcpcd` run DHCP on `wlan0`, so it gets two addresses (NM `.44`, dhcpcd `.48`). When dhcpcd adds `.48`, the `wlan0 proto kernel` route disappears.
- `/boot/scripts/connect_wifi.sh` (root watchdog, `check_interval=5`) treats "no `wlan0 proto kernel` route" as "no network", logs `==== No network connection...` to `/boot/scripts/wifi.log`, and runs `nmcli device connect wlan0`, which drops the link. The loop repeats.
- Sovol's touchscreen app (`zhongchuang_klipper`, source `/home/sovol/zhongchuang/src/mks_wpa_cli.cpp`) runs `system("dhcpcd wlan0")` on every `CTRL-EVENT-CONNECTED`. That explicit call **bypasses `denyinterfaces`**, so `denyinterfaces wlan0` alone did not work.
**Fix:** appended to `/etc/dhcpcd.conf` (a `denyinterfaces wlan0` line plus a per-interface block `interface wlan0` / `noipv4` / `noipv6`), then `systemctl restart dhcpcd` and `ip addr del 10.88.20.48/24 dev wlan0`. Original saved on the printer as `/etc/dhcpcd.conf.bak-20261013`.
**Files:** `backup/2026-10-13/system/dhcpcd.conf.original`, `dhcpcd.conf.fixed`, and `connect_wifi.sh.sovol-stock` (unmodified Sovol watchdog, for reference). **If a Sovol OTA or reflash wipes the fix**, re-apply `dhcpcd.conf.fixed` to `/etc/dhcpcd.conf` and restart `dhcpcd`.
**Who is likely affected (hypothesis, untested):** not specific to mesh/multi-AP networks (the loop has no roaming component; extra BSSIDs only made it look like roaming). NM and dhcpcd send different DHCP client IDs (dhcpcd uses a DUID), so a DHCP server that keys leases on client ID (UniFi here handed out `.44` and `.48`) gives two addresses and triggers the loop. A router that keys on MAC only would give both clients the same IP and likely never show the problem. Wired-only use is unaffected. No matching reports found online as of 2026-10-13.
**After the fix (2026-10-13):** no disconnects, one IPv4 address, printer shows up stably in the UniFi portal (alias set), uploads noticeably faster. Optional: reserve `10.88.20.44` in UniFi.
**Verify:** `ip -br addr show wlan0` shows only one IPv4 address, and `tail /boot/scripts/wifi.log` stops adding "No network connection" lines.

## BIQU Panda (BTT) auxiliary feeder upgrade
Replaces the stock Sovol buffer/feeder with a BIQU Panda feeder (single feeder, no hub yet). Mounting STL/3MF files are a **paid kit and are git-ignored** – never commit them. Source: author's Cults3D post (macros updated through 2026-06-26).
- `bique feeder files/` holds the macro versions, named after the kit zips: `DKEU_v1` (oldest; reuses `LOAD_FILAMENT`/`UNLOAD_FILAMENT` names, DKEU-modified printers only), `STOCK_v2` (stock Sovol, `BTT_` names), `STOCK_DKEU_v4` (**newest, unified for stock and DKEU printers – use this**). STOCK = stock Sovol SV08 Max, DKEU = the modified variant.
- Author's install: upload the macro as `btt_feeder.cfg` to the config dir, then in `printer.cfg` replace `[include buffer_stepper.cfg]` with `#[include buffer_stepper.cfg]` + `[include btt_feeder.cfg]`. Bracket choice (single/double/triple) does not change the macros; triple enables auto-refill.
- No two-way comms between printer and feeder: for filament other than PLA/PETG, preheat the nozzle manually before using the feeder's load/unload buttons (this is why `BTT_UNLOAD_FILAMENT` doesn't heat).
- **Status: NOT installed yet** (waiting on author clarification). Known open issues in v4: runout handler calls `_FIL_CHANGE_PARK` (undefined on stock Sovol – should be `PAUSE STATE=filament_change`); filament sensor is `PB2` on the main MCU (unverified wiring). Sovol's `Macro.cfg` still calls `BUFFER_STEPPER`, LED `SET_PIN`, `MANUAL_FEED`, `NOZZLE_CLOG_CHECK`, `variables`, `CHECK_FILAMENT_STATUS`; v4 stubs these – verify after install.

## Board pinouts & filament sensor planning (researched 2026-10-08)
- Sovol publishes the SV08 Max board pinouts: https://github.com/Sovol3d/SV08MAX/tree/main/Motherboard (`Mcu_Pin_definition.pdf` = mainboard, STM32H750 = `[mcu]`; `Extra_Pin_definition.pdf` = toolhead board, STM32F103 = `extra_mcu`). Not copied here; fetch from the link.
- **`PB2` (used by the feeder macro's `filament_sensor`) is not on any connector in either diagram.** Treat that sensor as a placeholder until the author says otherwise.
- Pins already used by `printer.cfg`: X endstop `extra_mcu:PA10`, Y endstop `PD1`, plus motors/fans/heaters/eddy (`PB10`/`PB11` on toolhead = eddy I2C).
- Free candidate inputs for a BTT SFS V2.0 (needs 2 signals: runout switch + motion encoder, each with `^` pull-up, 3.3-5 V supply):
  - Toolhead board header `5V GND PC15 PC14` (`extra_mcu:PC15`/`PC14`) – closest to the extruder. Caution: F103 PC13-15 are not 5 V-tolerant, so power the SFS from 3.3 V or verify its output level before connecting.
  - Mainboard "X-axis limited" header `PD6 GND 5V` (single signal; X endstop is on the toolhead, so it is unused).
- SFS V2.0 manual: https://github.com/bigtreetech/smart-filament-detection-module/blob/master/V2.0/Manual/SFS%20V2.0%20User%20Manual_20231123.pdf – `filament_switch_sensor` + `filament_motion_sensor` (extruder: extruder), `detection_length` start ~3 mm, raise in 1 mm steps on false triggers (farther from the extruder = higher). Runout gcode should call Sovol's `PAUSE`.

### SFS V2.0 decisions (2026-10-08)
- Use the **motion output only** (runout is handled by the Panda feeder). Wire black (GND) + red (VCC) + green (motion) into one 3-pin plug for the mainboard "X-axis limit" header (`PD6 GND 5V`); leave the blue (switch) wire unconnected. Verify pin order against the pinout PDF before powering up.
- Config sketch: `[filament_motion_sensor encoder_sensor]`, `switch_pin: ^PD6`, `extruder: extruder`, `pause_on_runout: False`, runout_gcode `PAUSE`.
- **Must disable the motion sensor** (`SET_FILAMENT_SENSOR SENSOR=encoder_sensor ENABLE=0`, re-enable after) in the feeder load/unload macros and Sovol LOAD_FILAMENT/UNLOAD_FILAMENT, and for any purge/prime with no filament flow, or it will pause mid-load.
- Mount: **outside the enclosure, as close to the enclosure's filament inlet as possible** (SFS rated max 50 C). Design after the enclosure is installed. The tube run to the extruder adds slack, so expect a `detection_length` above the 3 mm default (start ~5-10 mm and tune in 1 mm steps).
- Sequence after the Halloween print jobs: SSH hardening -> Panda feeder install -> SFS install. Do not modify the printer while the big jobs run.

## Enclosure chamber heater (2026-11-22)
- Enabled by uncommenting `[include chamber_hot.cfg]` (line 2 of `printer.cfg`), then `FIRMWARE_RESTART`. Done entirely over SSH/Moonraker; screen firmware >= 2.1.8 is only needed for the touchscreen's chamber button (Mainsail works regardless). Pre-change copies on the printer: `~/printer_data/backup/pre-chamber-20261122/`. Latest `printer.cfg` snapshot: `backup/2026-11-22/`.
- Heater board `hot_mcu` (CAN UUID `58a72bb93aa4`, STM32F103, fw cc8afd8) is on the bus and matches the UUID in `chamber_hot.cfg`. Chamber thermistor read 23.5 C at idle (plausible). Heater: `heater_generic chamber_temp`, max_temp 65, watermark control; macros `M141`/`M191`.
- Enclosure fans: `fan2` (mainboard PB0 = K-FAN1 header) and `fan3` (PE11 = K-FAN2 header). The factory `check_chamber` macro runs `fan2` at 80% while the chamber target is >0 and stops it 30 s after the target returns to 0; `fan3` is manual only.
- **Known issue in Sovol's `chamber_hot.cfg`:** `check_chamber` calls `SAVE_VARIABLE` every second, rewriting `saved_variables.cfg` (which also holds power-loss-recovery data such as `power_resume_z`). A power cut mid-write could corrupt that file. Fix = only save when the target actually changes.
- Not yet done: first live heating test (see chat).
