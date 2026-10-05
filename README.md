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
