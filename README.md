# PicoOS releases

Compiled PicoOS firmware and apps for the ClockworkPi PicoCalc with a standard Raspberry Pi Pico 2 W. [Download a release](https://github.com/TheProfessorNQ/PicoOS-Releases/releases). Releases marked **Pre-release** are hardware-test candidates. Stable is published after on-device acceptance.

## First installation

1. Back up your SD card and retain the last working UF2.
2. Put the Pico 2 W in USB BOOTSEL mode and copy **first-install-pico2w.uf2** to the RP2350 drive.
3. Install **system-updater.capp** anywhere under `/apps` on the FAT32 card. Use **sd-apps.zip** to populate a fresh card. When updating an existing card by hand, replace packages at their current locations to avoid duplicate IDs. Keep your `/data` directory.
4. Boot the Calc, connect a network, and set or synchronize the clock.

## Updating from the Calc

Open **Settings → Update Status → F1**. Enter checks Stable. F2 changes to Preview. After you confirm the download, the updater verifies the firmware and apps and stages them on SD. Press Escape to return to Update Status, then Enter and confirm to install. Keep the card inserted and power connected until the launcher returns.

If power is interrupted during installation, restore power with the same card in place. If the card is unreadable, restore the signed **picoos.pup** to `/picoos/armed.pup` on a readable FAT32 card and cold-start the Calc, or use your USB rescue image. A normal update includes the PicoOS system and apps. The keyboard controller has a separate update procedure.

Each release includes third-party license notices. No GitHub account is needed on the device.
