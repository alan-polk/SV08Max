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

## OS update assessment
- `apt-get update` was run (package lists only; **no upgrades installed**). 123 packages upgradable, all from `bullseye-security` (libc, curl, gnutls, git, glib, gnupg, avahi, bind9 libs, etc.). No kernel/Armbian/Sovol packages are in the list (only `linux-libc-dev` headers), so security updates should not touch the kernel/boot.
- Debian 11 is past regular support (LTS only, apt calls it `oldoldstable`). `bullseye-backports` repo is dead (`apt-get update` errors on it) – remove/comment that line in sources.
- Recommendation: after this backup, `sudo apt-get upgrade` (not `dist-upgrade`/`full-upgrade`) is low risk; reboot afterwards. Do not do a release upgrade – the image is vendor-customised.
