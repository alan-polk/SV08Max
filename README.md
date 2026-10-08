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
