# Goal — short (simple version)

Create an **EFI System Partition (ESP)** and a **Microsoft Reserved Partition (MSR)** on an existing disk using a Linux USB (GParted + small CLI helpers), then boot WinPE and point Windows tools (DiskPart / BCDBoot) at those partitions so you can continue with an offline image deploy/repair workflow.

* Jargon (simple version):

  * **ESP** — a FAT32 partition that UEFI firmware reads for bootloaders. (simple version)
  * **MSR** — a small *unformatted* GPT partition Windows reserves for future internal use; it must exist on GPT disks for Windows. (simple version)

I will give a safe, fully reproducible sequence (GUI + CLI) with verification steps and recovery/rollback advice. I quote authoritative docs (Microsoft, GParted manual, gdisk/Arch, mkfs manual) so you can independently verify each claim. ([gparted.org][1])

---

# High-level plan (what we will actually do)

1. Boot a Linux USB (GParted Live or Ubuntu live).
2. Confirm target disk (do not touch the wrong disk).
3. Create a GPT partition table (if disk is MBR and you want UEFI/GPT).
4. Create & format the **ESP** (FAT32, 100–260 MiB), set GParted flags `boot, esp`.
5. Create a small **unformatted** partition for the **MSR** (16 MiB for modern Windows 10/11; larger for older versions if needed).
6. Use `gdisk` (GPT fdisk) to set the MSR partition’s GUID/type (gdisk code **0C01** → MSR).
7. Verify partitions in Linux. Export safe notes.
8. Reboot into **WinPE**, use `diskpart` to *format/assign* ESP (if needed) and assign a letter to it; leave MSR unlettered.
9. Continue with the DISM/Apply/bcdboot steps from the earlier guide.

I’ll cover exact commands and GUI clicks for each step and include verification commands.

---

# Warnings & preflight checks (read this carefully)

* **Back up** any important data. These operations change partition tables and can erase data.
* **Identify the disk**: confirm device node (e.g., `/dev/sda`, `/dev/nvme0n1`) with `lsblk` and `blkid`. Never assume `sda` is the target.
* Decide if you want to convert to GPT. Converting from MBR → GPT will destroy partition layout unless you take additional steps.
* Windows expects MSR after the ESP and before the Windows data partition; adhering to that order avoids subtle problems. ([Microsoft Learn][2])

---

# Detailed, step-by-step procedure

> Example variables I’ll use in examples below: target disk = **/dev/sda** (replace with your disk), ESP size = **260MiB**, MSR size = **16MiB**, Windows partition will later be whatever you choose (e.g., `/dev/sda3`). Adjust sizes if you have special needs.

## A — Boot Linux USB and open GParted

1. Boot the machine from a Linux USB (GParted Live or Ubuntu Live — *Try Ubuntu*).
2. Open **GParted** (`gparted` GUI). If not present, run `sudo apt update && sudo apt install gparted` on a live distro with networking.
3. In GParted, select the **target disk** from the top-right device menu (very important). Verify with `lsblk` in a terminal if unsure.

*Why GParted?* — GParted makes creating a FAT32 ESP easy and sets the `boot`/`esp` flags through a GUI. The manual documents the Manage Flags dialog. ([gparted.org][1])

## B — Create GPT table (if needed)

* **If the disk already uses GPT**, skip this step.
* **If it is MBR and you want GPT** (required for UEFI boot): In GParted → Device → Create Partition Table → choose **gpt** → Apply.

> **WARNING:** Creating a partition table wipes the partition table (possible data loss). Back up first.

## C — Create and format the ESP (FAT32)

1. In GParted, create a **new partition** in the beginning area of the disk:

   * Size: **260 MiB** (100–512MiB is OK; 100MiB minimum, 200-260MiB recommended).
   * File system: **fat32**.
   * Label (optional): `EFI` or `System`.
2. After it’s created, **right-click the partition → Manage Flags → enable `boot` and `esp`** (check both). Then click Apply (the green ✓).
3. If GParted didn’t format as FAT32 for any reason, you can format from a terminal:

   ```bash
   sudo mkfs.fat -F32 -n EFI /dev/sda1
   ```

   (replace `/dev/sda1` with your ESP partition). Use the `mkfs.fat` manual for flags and details. ([Arch Linux Man Pages][3])

*Authority:* UEFI/ESP requirements and common practice described in ArchWiki and other distro docs. The `boot/esp` flags in GParted are how GParted marks an EFI partition. ([ArchWiki][4])

## D — Create the MSR partition (GUI + gdisk)

> Important: GParted can create a raw partition of the right size, but it does not set the Windows MSR partition GUID by default. Use `gdisk` (GPT fdisk) to set the MSR type code to **0C01** (gdisk shorthand for Microsoft Reserved). The MSR must be unformatted (no filesystem).

1. In GParted create a **small unformatted partition** immediately after the ESP:

   * Size: **16 MiB** (Windows 10/11). Use **128 MiB** if you are preparing for older Windows or you prefer the extra slack (Windows historically used larger sizes).
   * File system: leave as **unformatted** / "unknown".
   * Do **not** set flags on the MSR; leave it unflagged. Apply changes.

2. Now use `gdisk` to set the partition type (you can also create the partition entirely with `gdisk` if you prefer command line):

   ```bash
   # Inspect current partitions
   sudo gdisk -l /dev/sda

   # Edit the partition type
   sudo gdisk /dev/sda
   ```

   Inside `gdisk` interactive session:

   * press `t`  ← change a partition’s type code
   * type the partition number of the MSR (e.g. `2`)
   * enter the type code: `0C01`  ← this is gdisk’s shorthand for Microsoft Reserved (maps to GUID `E3C9E316-0B5C-4DB8-817D-F92DF00215AE`).
   * press `w` to write changes and exit (confirm with `y`).

   Example:

   ```
   Command (? for help): t
   Partition number (1-128): 2
   Hex code or GUID (L to show codes): 0C01
   Command (? for help): w
   ```

   Use the gdisk docs/Arch manual as reference for the `0C01` code. `gdisk` internal code `0C01` maps to the Microsoft Reserved GUID. ([Arch Linux Man Pages][5])

3. Verify the partition table again:

   ```bash
   sudo gdisk -l /dev/sda
   sudo sgdisk -p /dev/sda   # prints partition table
   lsblk -f                 # quick FS view (MSR will be unformatted)
   ```

*Why use gdisk?* — `gdisk` exposes the shorthand codes (like `0C01`) for special Windows GPT types and is the common tool to set them correctly from Linux. ([Arch Linux Man Pages][5])

## E — Final Linux verification

* Check that:

  * The ESP partition is FAT32, size ≈ 100–260MiB and has flags `boot, esp`. (`GParted` shows the flags; `lsblk -f` shows filesystem.)
  * The MSR partition is unformatted and has type `0C01` / Microsoft Reserved in `gdisk`/`sgdisk` output.
* If anything looks wrong, **do not proceed**; stop and ask or restore from backup.

---

## F — Reboot into WinPE and use DiskPart to prepare ESP for Windows tools

After the Linux side is done, shut down, remove the Linux USB and boot into **WinPE** (your WinPE USB made earlier).

### In WinPE: identify partitions and assign a letter to the ESP

1. Open `cmd` → `diskpart`:

   ```
   diskpart
   list disk
   select disk 0              <-- pick the target disk carefully
   list partition
   select partition 1        <-- the ESP partition number from previous step
   ```

2. If the ESP is already FAT32 and formatted, you can **assign** a drive letter (do not reformat unless required):

   ```
   assign letter=S
   exit
   ```

   If the ESP is not formatted or you prefer to format in WinPE, do:

   ```
   select partition 1
   format fs=fat32 quick label="System"
   assign letter=S
   exit
   ```

3. **Do not assign a drive letter** to MSR. Leave MSR unlettered and unformatted. If Windows tooling complains you can create the MSR inside WinPE instead, using DiskPart:

   ```
   diskpart
   select disk 0
   create partition msr size=16    <- creates MSR correctly (if needed)
   ```

   The Microsoft doc for `create partition msr` describes this Windows-side option. If you created MSR from Linux with `gdisk`, skip this `create` command. ([Microsoft Learn][6])

4. Confirm `list vol` shows an `S:` volume (your ESP). You can `dir S:\` to validate it’s a FAT32 ESP.

*Why assign S:?* — `bcdboot` later needs an explicit target to copy boot files into, and `/s S:` is deterministic. Microsoft docs recommend using DiskPart to determine volumes then run `bcdboot`. ([Microsoft Learn][7])

---

## G — Use `bcdboot` (from WinPE) to populate the ESP and create BCD

After you have applied Windows to the Windows partition (for example `W:\Windows`) using your DISM steps:

```cmd
bcdboot W:\Windows /s S: /f UEFI
```

* This copies the firmware boot files into `S:` (ESP) and creates a BCD store pointing at the Windows install. If the firmware is legacy BIOS and you used an MBR layout, use `/f BIOS` instead. ([Microsoft Learn][7])

---

# Quick recovery tips & common gotchas

* If you accidentally formatted the wrong disk, stop and **do not** write further data. Use `dd`, `testdisk`, or professional recovery if data is critical.
* If Windows refuses to boot after `bcdboot`, re-boot to WinPE and re-run `diskpart` → `assign` to ensure the correct ESP letter, then re-run `bcdboot W:\Windows /s S: /f UEFI`. The `bcdboot` doc shows using DiskPart to confirm letters first. ([Microsoft Learn][7])
* If Windows tools still complain about MSR, you can let Windows create it: in WinPE `diskpart` → `create partition msr size=16`. But **do not** create a filesystem for MSR. ([Microsoft Learn][6])

---

# Verification checklist (before leaving WinPE / booting to disk)

* [ ] ESP present, formatted FAT32, letter assigned `S:` and contains `EFI\Microsoft\Boot\` after `bcdboot`.
* [ ] MSR present as unformatted GPT partition (no drive letter).
* [ ] Windows partition contains `Windows` folder and you used DISM to apply image or repair as needed.
* [ ] You ran `bcdboot` with correct `/s` and `/f` switches.

---

# WHY these steps / source trustworthiness

* **Microsoft documentation** is authoritative on MSR and `bcdboot` usage; follow Windows docs for sizes and parameters for MSR and `bcdboot`. ([Microsoft Learn][2])
* **GParted manual** documents how to set flags (Manage Flags) which is the correct GUI action to mark an ESP. Use it for the FAT32 GUI steps. ([gparted.org][1])
* **gdisk (GPT fdisk)** is the standard Linux tool to set GPT partition type codes (0C01 for MSR, EF00 for ESP in gdisk shorthand); using it guarantees the correct partition type GUID is written to the GPT table. Arch/gdisk docs confirm shorthand codes. ([Arch Linux Man Pages][5])
* **mkfs.fat / mkfs.vfat** manual describes FAT creation from Linux; use `sudo mkfs.fat -F32 /dev/sdXN` to format an ESP when needed. ([man7.org][8])

I favored **primary sources** (Microsoft, GParted docs, gdisk/Arch, man pages). Where community Q&A clarified practical behavior (e.g., using `gdisk` for MSR), I used it to explain how tools behave in practice — but the key facts (MSR GUID, size recommendations, `bcdboot` usage) come from Microsoft or official tool manuals.

---

# If you want, next steps I can do right now (pick any)

* Produce a single **copy-pasteable checklist / script** for your exact disk layout (you must tell me the device name and desired Windows partition size). I will not run anything — I will only produce the script.
* Walk you interactively through the terminal commands (explain each command as you run it).
* Show how to *create ESP + MSR entirely with `sgdisk`/`gdisk`* (no GParted GUI).

Tell me which of the three you want and give me the target disk device (e.g., `/dev/sda`) and preferred ESP size (if different from 260MiB) and MSR size (16MiB recommended for Windows 10/11).

[1]: https://gparted.org/display-doc.php%3Fname%3Dhelp-manual?utm_source=chatgpt.com "GParted Manual"
[2]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions?view=windows-11&utm_source=chatgpt.com "UEFI/GPT-based hard drive partitions"
[3]: https://man.archlinux.org/man/mkfs.fat.8.en?utm_source=chatgpt.com "mkfs.fat(8) - Arch manual pages"
[4]: https://wiki.archlinux.org/title/EFI_system_partition?utm_source=chatgpt.com "EFI system partition - ArchWiki"
[5]: https://man.archlinux.org/man/gdisk.8.en?utm_source=chatgpt.com "gdisk(8) — Arch manual pages"
[6]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-msr?utm_source=chatgpt.com "create partition msr"
[7]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di?view=windows-11&utm_source=chatgpt.com "BCDBoot Command-Line Options"
[8]: https://man7.org/linux/man-pages/man8/mkfs.vfat.8.html?utm_source=chatgpt.com "mkfs.vfat(8) - Linux manual page"
