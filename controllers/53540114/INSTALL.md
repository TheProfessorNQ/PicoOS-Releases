# Keyboard controller bench procedure

This procedure is for the photographed **ClockworkPi CPI2.0 mainboard with a CKS32F103RBT6 and 128 KiB of flash**. The Pico UF2, controller combined BIN and controller application PCU serve three different programming paths. Compare the receiving unit's markings and capacity before choosing an image. A shared ROM device ID alone cannot distinguish the 64 KiB and 128 KiB parts. 

## First installation through mainboard USB-C

1. Start with a functioning factory controller. Check power-on, arrows and F5. Keep the display connected. Power down, disconnect both USB cables, remove batteries and photograph the original switches. Set **DIP 1 ON, DIP 2 OFF**. Connect **only mainboard USB-C** to the PC using a data cable and **press the normal power button**. On Windows identify the CH340 COM port. Leave Pico USB disconnected for programming.
2. Open PowerShell 7 in the extracted **controller-source.zip** directory. Download the upstream programmer from the URL in the dependency lock. Bootstrap handles the compilers separately. The following commands check the project's pinned executable and make a fresh evidence folder.

   ```powershell
   $rom = Read-Host 'Mainboard CH340 port (e.g. COM12)'
   $archive = 'stm32flash-0.7-binaries.zip'
   $pin = (Get-Content .\dependencies.lock.json -Raw | ConvertFrom-Json).local_dependencies.stm32flash
   Invoke-WebRequest $pin.archive_url -OutFile $archive
   if ((Get-FileHash $archive).Hash -ne $pin.archive_sha256) { throw 'Programmer archive differs' }
   $dir = '.\.deps\stm32flash-0.7'
   if (!(Test-Path "$dir/$($pin.executable)")) { Expand-Archive -LiteralPath $archive -DestinationPath $dir }
   $flash = (Resolve-Path "$dir/$($pin.executable)").Path
   if ((Get-FileHash $flash).Hash -ne $pin.executable_sha256) { throw 'Programmer executable differs' }
   $work = (New-Item -ItemType Directory -Path ("build/controller-" + (Get-Date -Format yyyyMMdd-HHmmss)) -Force).FullName
   function Run-Rom { & $flash @args; if ($LASTEXITCODE) { throw "Programmer failed ($LASTEXITCODE)" } }
   Run-Rom -b 57600 $rom
   Run-Rom -c -b 57600 -S 0x1FFFF7E0:2 -r "$work/size.bin" $rom
   if ([BitConverter]::ToString([IO.File]::ReadAllBytes("$work/size.bin")) -ne '80-00') { throw 'Expected 128 KiB part' }
   ```

3. Compare the part marking, board and ROM report. The reference bench reported ROM 0x22 and ID 0x0410. Stop on an unexpected identity, protection setting or capacity. Take two independent complete reads. Each command performs a separate ROM transaction.

   ```powershell
   Run-Rom -c -b 57600 -S 0x08000000:131072 -r "$work/factory-a.bin" $rom
   Run-Rom -c -b 57600 -S 0x08000000:131072 -r "$work/factory-b.bin" $rom
   ```

   Create `$work/device.json` from observations for **this** device. The verifier requires `controller_model`, `controller_marking`, `device_id`, `flash_bytes`, `board_revision`, `tool_version`, `port` and Boolean `independent_reads_confirmed`. Example values for the matching model are `CKS32F103RBT6`, `0x0410` and `131072`. Record the full marking and actual COM port. Verify both reads and copy the archive to a second storage device before the first write.

   ```powershell
   .\scripts\stm32-backup-verify.ps1 -ReadA "$work/factory-a.bin" -ReadB "$work/factory-b.bin" -DeviceRecord "$work/device.json" -ArchiveDirectory "$work/verified-original"
   ```

4. With the owner ready for the factory-restore check, write **this unit's verified original**, verify it, then independently read the whole chip. The explicitly bounded 128-page operation here is for restoring this model's full factory backup.

   ```powershell
   Run-Rom -c -b 57600 -w "$work/verified-original/factory-read-a.bin" -v -n 0 -s 0 -e 128 $rom
   Run-Rom -c -b 57600 -S 0x08000000:131072 -r "$work/factory-restored.bin" $rom
   .\scripts\stm32-backup-verify.ps1 -ReadA "$work/factory-a.bin" -ReadB "$work/factory-b.bin" -Readback "$work/factory-restored.bin" -DeviceRecord "$work/device.json" -ArchiveDirectory "$work/verified-restore"
   ```

5. Disconnect mainboard USB-C, set both DIP switches OFF, reconnect mainboard USB-C and press Power. Confirm the factory controller boots with launcher, arrows and F5 working. Keep the case open for the combined installation. Power down and re-enter programming mode exactly as in step 1. Run `Run-Rom -b 57600 $rom` for the new ROM session.
6. Install **initial-install.bin**. It includes the resident updater, controller application and commit record. The write is deliberately limited to the lower 64 KiB. Verify the lower half against the combined image and the upper half against this unit's original read.

   ```powershell
   $image = (Resolve-Path 'initial-install.bin').Path
   if ((Get-Item $image).Length -ne 65536) { throw 'Wrong combined image length' }
   Run-Rom -c -b 57600 -w $image -v -n 0 -s 0 -e 64 $rom
   Run-Rom -c -b 57600 -S 0x08000000:131072 -r "$work/combined-readback.bin" $rom
   $read = [IO.File]::ReadAllBytes("$work/combined-readback.bin")
   $expected = [IO.File]::ReadAllBytes("$work/verified-original/factory-read-a.bin")
   [IO.File]::ReadAllBytes($image).CopyTo($expected, 0)
   if (![System.Collections.StructuralComparisons]::StructuralEqualityComparer.Equals($read,$expected)) { throw 'Controller readback differs' }
   ```

7. Disconnect mainboard USB-C, return **both DIP switches OFF**, reconnect mainboard USB-C and press Power. Check launcher, arrows and F5. Connect Pico USB, open resident Diagnostics and confirm **STOCK+ CONTROLLER 53540114** or capture the identity in the boot log. Reassemble after this grouped check. Keep the verified factory archive and photos with the unit.

## Return this unit to factory firmware

If ROM recovery becomes necessary, use the original archive verified for this exact unit. Re-enter the ROM loader as in step 1, select the factory image and perform step 4's bounded 128-page restore and independent full readback. Compare the readback to the preserved originals, then perform step 5's normal boot check. Keep the originals intact. Clear any queued custom PCU from the SD card before the factory boot, retaining a copy for diagnosis.


## Later updates on the Calc
Use PicoOS Preview 5 or later with System Updater 0.2.0 or later. Open Settings → Update Status → F1. In System Updater, F3 chooses Controller and F2 selects the Preview channel. Press Enter to check and again to download. Escape returns to Update Status. Use F2 there to select Controller, compare the installed and downloaded build IDs, then press Enter and confirm installation. Keep power and SD connected during writing and verification. After an interruption restore power with the same SD card. The resident updater resumes from the retained package. Confirm the expected controller identity in Diagnostics and check keyboard input.

For manual SD staging on a controller already equipped with the PU/1 updater, place the verified controller-update.pcu at /shared/controller-update.pcu and restart with the card inserted. The network download and manual SD methods both use that resident installer.
