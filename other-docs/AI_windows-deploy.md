# Manual or Semi-Automated **Offline** Windows Deployment & Repair (WinPE + DISM + BCDBoot + SFC)

You are about to do low-level system work.
Assume **drive letters in WinPE are not what you expect**, and that an incorrect `diskpart`/`dism` target can wipe data.
Make backups and validate targets before every destructive command.

-   This guide prefers **primary sources (Microsoft Learn/Docs)** and uses third-party posts only for color.

    -   Commands are confirmed at the original Microsoft pages; no hearsay chains.
    -   Where there are multiple options, this shows the **safest defaults** and explains tradeoffs.

---

## What you will build or fix (TL;DR)

-   **Deploy Windows to a blank disk offline**:
    Create WinPE → partition the disk → apply `install.wim` → make it bootable with `bcdboot` → optional servicing (drivers, features, updates) → reboot. ([Microsoft Learn][1])

-   **Repair an existing, unbootable Windows offline**:
    Boot WinPE → mount the offline Windows → `DISM /RestoreHealth` with a clean source → `SFC /scannow` offline → rebuild boot files with `bcdboot` → (re)enable WinRE. ([Microsoft Learn][2])

---

# Part 0 — Prereqs & sanity checks

-   **WinPE media** (USB or ISO) from **Windows ADK + WinPE add-on**.
    Build with `copype`, write with `MakeWinPEMedia`. ([Microsoft Learn][3], [6])

-   **Matching Windows ISO** (same major version/edition) for `sources\install.wim` or `install.esd` as the **deploy/repair source**.

    -   Find the **edition index** you need with `DISM /Get-WimInfo`. ([Microsoft Learn][4])

-   **Know your firmware**: UEFI/GPT vs Legacy BIOS/MBR. Partitioning and `bcdboot /f` differ. ([Microsoft Learn][5])

> Jargon cheat-sheet: **WinPE** (mini Windows for deployment/repair), **DISM** (image capture/apply/repair), **WIM/ESD** (image formats), **ESP** (EFI System Partition).

---

# Part 1 — Create WinPE media (once, on a tech PC)

Open **Deployment and Imaging Tools Environment** as Admin:

```bat
:: Create working set
copype amd64 C:\WinPE_amd64

:: Create a bootable USB (replace F: with your USB)
MakeWinPEMedia /UFD C:\WinPE_amd64 F:
```

([Microsoft Learn][6])

---

# Part 2 — Boot target into WinPE and identify volumes

In WinPE (`X:\>`), use `diskpart` to **list disks/volumes** and confirm letters:

```bat
diskpart
list disk
sel disk 0
list vol
exit
```

**Do not assume** the Windows partition is `C:` in WinPE; it might be `D:` or `W:`.

---

# Part 3 — Partition the disk

## A) UEFI/GPT layout (recommended)

Windows’ default UEFI layout: **ESP (FAT32) → MSR → Windows (NTFS) → WinRE (NTFS)**. ([Microsoft Learn][5])

```bat
diskpart
rem *** DANGER: wipes disk 0 ***
sel disk 0
clean
convert gpt

rem 1) ESP ~260MB (FAT32)
create partition efi size=260
format fs=fat32 quick label="System"
assign letter=S

rem 2) MSR 16MB (no format)
create partition msr size=16

rem 3) Windows (use most of disk, leave ~1024MB at end for WinRE if you want a bigger WinRE)
create partition primary
shrink desired=1024
format fs=ntfs quick label="Windows"
assign letter=W

rem 4) WinRE ~1024MB (NTFS) — modern updates often need ~1GB
create partition primary size=1024
format fs=ntfs quick label="WINRE"
assign letter=T
exit
```

> Why 1 GB WinRE? Recent servicing can bloat WinRE; this avoids “not enough space” when enabling/updating WinRE later.

## B) Legacy BIOS/MBR layout (only if you must)

```bat
diskpart
sel disk 0
clean
convert mbr

rem System Reserved ~100MB
create partition primary size=100
format fs=ntfs quick label="SystemReserved"
active
assign letter=S

rem Windows (rest)
create partition primary
format fs=ntfs quick label="Windows"
assign letter=W
exit
```

Later use `bcdboot /f BIOS`. ([Microsoft Learn][8], [10])

---

# Part 4 — Apply Windows from install.wim (or .esd)

Identify the edition index you want (e.g., Pro might be 6):

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim

:: If your ISO uses ESD:
dism /Get-WimInfo /WimFile:D:\sources\install.esd
```

Apply the image to your Windows partition (`W:`): ([Microsoft Learn][4])

```bat
dism /Apply-Image /ImageFile:D:\sources\install.wim /Index:6 /ApplyDir:W:\
```

> **Start with a clean Windows partition (no leftover Windows/Users/Program Files).**
> Stray folders—especially `\Users` and `\Program Files\WindowsApps`—often contain **NTFS reparse points** (junctions/symlinks). During specialize/OOBE, **windeploy** tries to retarget links and can fail with `STATUS_ACCESS_DENIED (c0000022)` in `setup.etl` (look for `Sclp*ReparsePoint*` and “Failed to retarget links”).
> **Best practice:** format the Windows partition before `/Apply-Image`. If you must reuse a volume, enumerate and remove reparse points deliberately:
>
> ```bat
> :: List reparse points on W:
> dir W:\ /AL /S
>
> :: Inspect an item
> fsutil reparsepoint query "W:\path\to\item"
>
> :: Remove a junction/symlink (removes the link, not its target)
> rmdir "W:\path\to\item"
> ```
>
> **Removing `WindowsApps` (TrustedInstaller-owned) offline:**
>
> ```bat
> takeown /f "W:\Program Files\WindowsApps" /r /d y
> icacls "W:\Program Files\WindowsApps" /grant administrators:F /t
> rmdir /s /q "W:\Program Files\WindowsApps"
> ```
>
> If a full `\Users` tree exists on the **target**, don’t mass-delete entries from `DIR /AL /S`. **Stop and format** (or move user data elsewhere) to avoid orphaned ACLs and the same failure later. After cleanup, run `chkdsk W: /f`.

---

# Part 5 — Make it bootable with **BCDBoot**

```bat
:: UEFI
bcdboot W:\Windows /s S: /f UEFI

:: Legacy BIOS
bcdboot W:\Windows /s S: /f BIOS
```

`bcdboot` copies boot files to the **ESP** (UEFI) or **System Reserved** (BIOS) and creates BCD. Using both `/s` and `/f` avoids WinPE guessing wrong. ([Microsoft Learn][10])

---

# Part 6 — Optional offline servicing **before first boot**

## 6.1 Repair the component store and system files

```bat
:: Use a clean, matching source (points to your ISO/USB)
dism /Image:W:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:6

:: Then verify system files against the repaired store
sfc /scannow /offbootdir=W:\ /offwindir=W:\Windows
```

Run **SFC after DISM**. ([Microsoft Learn][2], [15])

## 6.2 Inject drivers (storage/NIC/model-specific)

```bat
:: Add all .inf drivers from a folder, recursively
dism /Image:W:\ /Add-Driver /Driver:D:\Drivers /Recurse
```

([Microsoft Learn][11])

## 6.3 Enable features offline (example: .NET 3.5)

```bat
:: Point /Source to \sources\sxs on a matching ISO (or local SxS repo)
dism /Image:W:\ /Enable-Feature /FeatureName:NetFx3 /All /Source:D:\sources\sxs
```

([Microsoft Learn][12])

## 6.4 Offline registry edits (advanced)

```bat
reg load HKLM\WimREG W:\Windows\System32\Config\SOFTWARE
:: ...make your changes under HKLM\WimREG...
reg unload HKLM\WimREG
```

([Microsoft Learn][13])

## 6.5 Windows Recovery Environment (WinRE)

```bat
:: If you created a dedicated tools partition (T:)
reagentc /setreimage /path T:\Recovery\WindowsRE /target W:\Windows
reagentc /enable
reagentc /info
```

> **Enable WinRE (`reagentc /enable`) after deployment or repair.**
> WinRE isn’t strictly required to complete Setup, but features like **Reset this PC**, **Startup Repair**, and **BitLocker recovery** depend on it. We’ve seen first-boot/OOBE issues clear up once WinRE was provisioned correctly.
> **Partition size tip:** modern updates often need ~**1 GB** WinRE. If `reagentc /enable` reports insufficient space, shrink OS and create a dedicated NTFS WinRE partition:
>
> ```bat
> diskpart
> sel disk 0
> sel vol W
> shrink desired=1024
> create partition primary size=1024
> format fs=ntfs quick label="WINRE"
> assign letter=T
> exit
>
> md T:\Recovery\WindowsRE
> copy W:\Windows\System32\Recovery\Winre.wim T:\Recovery\WindowsRE\
> reagentc /setreimage /path T:\Recovery\WindowsRE /target W:\Windows
> reagentc /enable
> reagentc /info
> ```

([Microsoft Learn][14], [17])

---

# Part 7 — Reboot and first-boot checks

-   Eject the WinPE USB, boot from disk, complete OOBE.
-   Verify Device Manager, run Windows Update, confirm `reagentc /info` and `bcdedit /enum`.

---

## Troubleshooting

### 1) **Reparse-point failures & “Windows could not finish configuring the system”**

**Symptoms you saw**

-   OOBE error: _“Windows could not finish configuring the system. To attempt to resume…”_
-   `C:\Windows\Panther\UnattendGC\setupact.log`: `setupcl has pending operations; blocking deployment…`
-   `setup.etl`:

    -   `SclpLinkProcessReparsePoint ... (c0000022)`
    -   `Failed to enumerate reparse points` / `Failed to retarget links`.

**Root cause (common)**
Leftover junctions/symlinks from an old install—most often preexisting `\Users` or `\Program Files\WindowsApps`—on the **target** partition.

**Safe fix order**

1. **Prefer a clean partition**: quick-format the Windows volume before `dism /Apply-Image`.
2. If reusing:

    - Enumerate: `dir W:\ /AL /S`
    - Inspect: `fsutil reparsepoint query "<path>"`
    - Remove only the **links** with `rmdir "<path>"`.
    - For `WindowsApps`, take ownership + ACL reset, then delete (see Part 4 callout).
    - `chkdsk W: /f`

3. If Setup already failed once, clear pendings:

    ```bat
    dism /Image:W:\ /Cleanup-Image /RevertPendingActions
    ```

    Then recheck step 2 and reboot.

4. If boot records got muddled, rebuild:

    ```bat
    bcdboot W:\Windows /s S: /f UEFI
    ```

> **Do not** mass-delete everything from `DIR /AL /S`. Some entries are legitimate system junctions created by Windows during apply.

### 2) **“I deleted \Users trying to fix it—now what?”**

Avoid hand-crafting system junctions (e.g., `All Users`, `Default User`). They’re special and owned by TrustedInstaller.
**Best options**:

-   If you’re still offline **before first boot**: simply **re-apply** the image (`/Apply-Image`) to `W:` (fastest, safest).
-   If you must salvage data on the same volume: move user data out (e.g., `Users\Alice\Documents`) to another drive, format `W:`, re-apply, then bring data back.
-   As a last resort on an existing install: run the DISM+SFC sequence (Part 6.1). Do not try to recreate `All Users`/`Default User` junctions manually.

### 3) **Wrong letters / ambiguous targets**

Always `diskpart → list vol` and use explicit letters in **every** `dism`, `sfc`, and `bcdboot` command. Never rely on defaults in WinPE.

### 4) **Boot files not found / “No bootable device”**

Re-run `bcdboot` **explicitly** (don’t rely on `bootrec` auto-detection):

```bat
bcdboot W:\Windows /s S: /f UEFI
```

([Microsoft Learn][10])

### 5) **UEFI vs BIOS mismatch**

If firmware is UEFI but disk is MBR (or vice-versa), it won’t boot. Match partition scheme and `bcdboot /f` accordingly. ([Microsoft Learn][5], [10])

### 6) **DISM /RestoreHealth can’t find source**

Confirm the **index** matches target edition and the media path is correct:

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim
dism /Image:W:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:<Index>
```

([Microsoft Learn][4])

### 7) **SFC offline syntax errors**

Use **both** `/offbootdir` and `/offwindir`, or SFC may refuse to run:

```bat
sfc /scannow /offbootdir=W:\ /offwindir=W:\Windows
```

([Microsoft Learn][15])

### 8) **Stuck mounted WIM**

If you closed a window before unmounting:
`dism /Cleanup-Wim` or PowerShell `Dismount-WindowsImage -Path <mount> -Discard`. ([Microsoft Learn][16])

---

## Full end-to-end example (UEFI/GPT, new blank disk)

1. **Partition** (finish with `S:` ESP and `W:` Windows, optional `T:` WinRE ~1 GB)
2. **Apply**:

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim
dism /Apply-Image /ImageFile:D:\sources\install.wim /Index:6 /ApplyDir:W:\
```

([Microsoft Learn][4])

3. **Boot files**:

```bat
bcdboot W:\Windows /s S: /f UEFI
```

([Microsoft Learn][10])

4. **Optional servicing**:

```bat
dism /Image:W:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:6
sfc /scannow /offbootdir=W:\ /offwindir=W:\Windows
```

([Microsoft Learn][2])

5. **Enable WinRE** (if you made `T:`):

```bat
reagentc /setreimage /path T:\Recovery\WindowsRE /target W:\Windows
reagentc /enable & reagentc /info
```

([Microsoft Learn][14])

6. **Reboot** and complete OOBE.

---

## Repair-only example (existing Windows will not boot)

1. Boot WinPE → `diskpart` to find your offline Windows (say it’s `C:`).
2. **Health repair** with a known-good ISO mounted as `D:`:

```bat
dism /Image:C:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:6
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows
```

([Microsoft Learn][2])

3. **Rebuild boot files** (set your ESP as `S:` first if needed):

```bat
bcdboot C:\Windows /s S: /f UEFI
```

([Microsoft Learn][10])

4. **WinRE** (optional but recommended): `reagentc /enable` then `reagentc /info`. ([Microsoft Learn][17])

---

## Appendix — PowerShell equivalents & handy one-liners

-   Mount/service/commit a WIM:

```powershell
Mount-WindowsImage -ImagePath D:\sources\install.wim -Index 6 -Path C:\Mount
# ... servicing ...
Dismount-WindowsImage -Path C:\Mount -Save
```

([Microsoft Learn][23])

-   Create WinPE working files and media (PowerShell-friendly contexts also in docs). ([Microsoft Learn][24])

---

[1]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11&utm_source=chatgpt.com "Create bootable Windows PE media"
[2]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11&utm_source=chatgpt.com "Repair a Windows Image"
[3]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro?view=windows-11&utm_source=chatgpt.com "Windows PE (WinPE)"
[4]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/take-inventory-of-an-image-or-component-using-dism?view=windows-11&utm_source=chatgpt.com "Take Inventory of an Image or Component Using DISM"
[5]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions?view=windows-11&utm_source=chatgpt.com "UEFI/GPT-based hard drive partitions"
[6]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/copype-command-line-options?view=windows-11&utm_source=chatgpt.com "Copype Command-Line Options"
[7]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-efi?utm_source=chatgpt.com "create partition efi"
[8]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/bcdboot?utm_source=chatgpt.com "bcdboot (overview)"
[9]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism---deployment-image-servicing-and-management-technical-reference-for-windows?view=windows-11&utm_source=chatgpt.com "DISM Technical Reference"
[10]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di?view=windows-11&utm_source=chatgpt.com "BCDBoot Command-Line Options"
[11]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/add-and-remove-drivers-to-an-offline-windows-image?view=windows-11&utm_source=chatgpt.com "Add/Remove Drivers (Offline)"
[12]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-net-framework-35-by-using-deployment-image-servicing-and-management--dism?view=windows-11&utm_source=chatgpt.com "Deploy .NET Framework 3.5 with DISM"
[13]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/reg-load?utm_source=chatgpt.com "reg load"
[14]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-re?view=windows-11&utm_source=chatgpt.com "Deploy Windows RE"
[15]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sfc?utm_source=chatgpt.com "SFC"
[16]: https://learn.microsoft.com/en-us/powershell/module/dism/dismount-windowsimage?view=windowsserver2025-ps&utm_source=chatgpt.com "Dismount-WindowsImage (DISM)"
[17]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options?view=windows-11&utm_source=chatgpt.com "REAgentC options"
[18]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--generalize--a-windows-installation?view=windows-11&utm_source=chatgpt.com "Sysprep (Generalize)"
[19]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/capture-and-apply-windows-system-and-recovery-partitions?view=windows-11&utm_source=chatgpt.com "Capture/Apply System & Recovery Partitions"
[20]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-using-full-flash-update--ffu?view=windows-11&utm_source=chatgpt.com "Deploy Windows using FFU"
[21]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-image-management-command-line-options-s14?view=windows-11&utm_source=chatgpt.com "DISM Image Management Options"
[22]: https://www.windowscentral.com/microsoft/windows-help/how-to-use-dism-command-tool-to-repair-windows-10-image?utm_source=chatgpt.com "Windows Central: DISM + SFC order (secondary)"
[23]: https://learn.microsoft.com/en-us/powershell/module/dism/mount-windowsimage?view=windowsserver2025-ps&utm_source=chatgpt.com "Mount-WindowsImage (DISM)"
[24]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/makewinpemedia-command-line-options?view=windows-11&utm_source=chatgpt.com "MakeWinPEMedia Options"