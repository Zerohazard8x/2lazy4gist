# Manual or Semi-Automated **Offline** Windows Deployment & Repair (WinPE + DISM + BCDBoot + SFC)

You are about to do low-level system work.
Assume **drive letters in WinPE are not what you expect**, and that the wrong `diskpart` or `dism` target can wipe data.
Make backups and validate targets before every destructive command.

* This guide prefers **primary sources (Microsoft Learn/Docs)** and uses third-party posts only for color.

  * Commands are confirmed at the original Microsoft pages; I avoid “X said Y said Z” chain-citations.
  * Where there are multiple options, I show the safest defaults and explain tradeoffs.

---

## What you will build or fix (simple version = “TL;DR”)

* **Deploy Windows to a blank disk offline**:
  Create WinPE → partition the disk → apply `install.wim` → make it bootable with `bcdboot` → optional servicing (drivers, features, updates) → reboot. ([Microsoft Learn][1])

* **Repair an existing, unbootable Windows offline**:
  Boot WinPE → mount the offline Windows → run `DISM /RestoreHealth` with a clean source and `SFC /scannow` offline → rebuild boot files with `bcdboot` → re-enable WinRE if needed. ([Microsoft Learn][2])

---

# Part 0 — Prerequisites and “sanity checks”

* **WinPE media** (USB or ISO) made from the **Windows ADK + WinPE add-on**.

  * Create working files with `copype`, then make boot media with `MakeWinPEMedia`. ([Microsoft Learn][3])
* A **matching Windows ISO** (same major version/edition) so you can use `sources\install.wim` or `install.esd` as a **repair/deploy source**.

  * You will also need the **index number** for the edition you intend to deploy or use as your repair source. Use `DISM /Get-WimInfo`. ([Microsoft Learn][4])
* **Know your firmware**: UEFI/GPT vs Legacy BIOS/MBR. Partitioning and `bcdboot /f` differ. ([Microsoft Learn][5])

> Jargon: **WinPE** (Windows Preinstallation Environment) is a minimal Windows you boot into for deployment and repair. **DISM** is the tool that captures/applies/repairs Windows images. **WIM** is the Windows Imaging format. **ESP** (EFI System Partition) is the small FAT32 partition UEFI firmware boots. (simple version)

---

# Part 1 — Create WinPE media (once, on a technician PC)

Open **Deployment and Imaging Tools Environment** as Administrator:

```bat
:: Create working set
copype amd64 C:\WinPE_amd64

:: Create a bootable USB (replace F: with your USB)
MakeWinPEMedia /UFD C:\WinPE_amd64 F:
```

* `copype` builds a working PE tree; `MakeWinPEMedia` writes it to USB. ([Microsoft Learn][6])

---

# Part 2 — Boot target PC into WinPE and identify volumes

In WinPE (X:>), start `diskpart` to **list disks and volumes** and confirm letters:

```bat
diskpart
list disk
sel disk 0
list vol
exit
```

* You need these letters accurately for later steps.
* In WinPE, the Windows partition might be `D:` or `G:`. Do not assume `C:`. (General caution; no doc needed.)

---

# Part 3 — Partition the disk

### A) UEFI/GPT layout (recommended on modern hardware)

Windows’ default UEFI layout is: **ESP (FAT32) → MSR → Windows (NTFS) → WinRE (NTFS)**.
Use a script like this (sizes are examples; adapt to your disk): ([Microsoft Learn][5])

```bat
diskpart
rem *** DANGER: wipes disk 0 ***
sel disk 0
clean
convert gpt

rem ESP ~260MB FAT32
create partition efi size=260
format fs=fat32 quick label="System"
assign letter=S

rem MSR 16MB
create partition msr size=16

rem Windows partition (rest of disk)
create partition primary
format fs=ntfs quick label="Windows"
assign letter=W

rem Optional: WinRE tools partition ~750MB
shrink desired=750
create partition primary size=750
format fs=ntfs quick label="WINRE"
assign letter=T

exit
```

* `create partition efi` must run on a **GPT** disk; ESP must be **FAT32**. ([Microsoft Learn][7])

### B) Legacy BIOS/MBR layout (only if you must)

```bat
diskpart
sel disk 0
clean
convert mbr
create partition primary size=100
format fs=ntfs quick label="SystemReserved"
active
assign letter=S
create partition primary
format fs=ntfs quick label="Windows"
assign letter=W
exit
```

* BIOS needs an **Active** system partition; later use `bcdboot /f BIOS`. ([Microsoft Learn][8])

---

# Part 4 — Apply Windows from install.wim (or .esd)

First, identify the edition index you want (for example, Pro might be 6):

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim

:: If ESD:
dism /Get-WimInfo /WimFile:D:\sources\install.esd
```

Then apply the image to your Windows partition (here `W:`): ([Microsoft Learn][4])

```bat
dism /Apply-Image /ImageFile:D:\sources\install.wim /Index:6 /ApplyDir:W:\
```

> If your ISO has `install.esd`, DISM can still apply it. Same syntax, different file extension. (From DISM image mgmt.) ([Microsoft Learn][9])

> Please make sure that the target drive does not have a complete Users folder, or WindowsApps folder. This will cause reparse point errors in setup.etl. To see reparse points, do `DIR /AL /S` in the root of the target drive.

---

# Part 5 — Make it bootable with **BCDBoot**

```bat
:: UEFI
bcdboot W:\Windows /s S: /f UEFI

:: Legacy BIOS
bcdboot W:\Windows /s S: /f BIOS
```

* `bcdboot` copies the required boot files to the **ESP** (UEFI) or the **System Reserved** partition (BIOS) and creates the BCD store.
* Using `/s` and `/f` removes ambiguity when WinPE guesses the wrong partition/firmware. ([Microsoft Learn][10])

---

# Part 6 — Optional offline servicing **before first boot**

These are common, safe post-apply steps while the image is still offline.

### 6.1 Repair the component store and system files (when repairing an existing Windows or validating a deployment)

```bat
:: Use a clean source that matches the target (points at your ISO/USB)
dism /Image:W:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:6

:: Then verify system files against the repaired store
sfc /scannow /offbootdir=W:\ /offwindir=W:\Windows
```

* `DISM /RestoreHealth` repairs the **component store** using your clean WIM as source; `SFC` fixes protected system files.
* The **offline** switches for SFC are `/offbootdir` and `/offwindir`. Run SFC **after** DISM. ([Microsoft Learn][2])

### 6.2 Inject drivers (for storage/NIC or model-specific needs)

```bat
:: Add all .inf-based drivers from a folder, recursively
dism /Image:W:\ /Add-Driver /Driver:D:\Drivers /Recurse
```

* Works on offline Windows and WinPE; PowerShell equivalents exist (`Add-WindowsDriver`). ([Microsoft Learn][11])

### 6.3 Enable features offline (example: .NET Framework 3.5)

```bat
:: Point /Source to \sources\sxs on a matching ISO (or local SxS repo)
dism /Image:W:\ /Enable-Feature /FeatureName:NetFx3 /All /Source:D:\sources\sxs
```

* Use a **matching** source for language/edition; for multi-language images, add NetFx3 **before** language packs. ([Microsoft Learn][12])

### 6.4 Offline registry edits (advanced)

```bat
reg load HKLM\WimREG W:\Windows\System32\Config\SOFTWARE
:: ...make your changes under HKLM\WimREG...
reg unload HKLM\WimREG
```

* `reg load` mounts an offline hive under a temporary key; **always** `reg unload` before reboot. ([Microsoft Learn][13])

### 6.5 Windows Recovery Environment (WinRE)

```bat
:: Point WinRE at your tools partition if you created one
reagentc /setreimage /path T:\Recovery\WindowsRE /target W:\Windows
reagentc /enable
reagentc /info
```

* Use `Deploy Windows RE` when you build a custom WinRE; use `reagentc` to set and enable it. ([Microsoft Learn][14])
> Windows appears to usually throw an error in installation if you do not do `reagentc /enable`

---

# Part 7 — Reboot and first-boot checks

* Remove the WinPE USB, boot from disk, complete OOBE, verify device manager and Windows Update.
* If the system does not boot, see **Troubleshooting** below. (General practice.)

---

## Troubleshooting (most common real-world snags)

* **Wrong letters**: In WinPE, your intended `C:` might be `W:` or `G:`. Always `list vol` before running `dism` or `bcdboot`. (General practice.)

* **Boot files not found or “no bootable device”**
  Re-run `bcdboot` **explicitly** targeting the correct partitions and firmware:

  ```bat
  bcdboot W:\Windows /s S: /f UEFI
  ```

  This is safer and more deterministic than hoping `bootrec` finds the right BCD. ([Microsoft Learn][8])

* **UEFI vs BIOS mismatch**
  If firmware is UEFI but you partitioned MBR (or vice-versa), the system will not boot. Match your partition scheme and `bcdboot /f`. ([Microsoft Learn][5])

* **DISM restore health cannot find source**
  Confirm the **index** matches the target edition and that the media is accessible:
  `dism /Get-WimInfo /WimFile:D:\sources\install.wim` → check the index; then use `/Source:wim:D:\sources\install.wim:<Index>`. ([Microsoft Learn][4])

* **SFC offline syntax**
  Use **both** `/offbootdir` and `/offwindir`, or SFC may refuse to start repairs. ([Microsoft Learn][15])

* **Stuck mounted WIM** (you closed the window before unmounting)
  Use `dism /Cleanup-Wim` or PowerShell `Dismount-WindowsImage` with `/Discard` as needed. ([Microsoft Learn][16])

---

## Full end-to-end example (UEFI/GPT, new blank disk)

1. **Partition**
   Use the UEFI script from Part 3-A; end with `S:` (ESP) and `W:` (Windows).

2. **Apply image**

```bat
dism /Get-WimInfo /WimFile:D:\sources\install.wim
dism /Apply-Image /ImageFile:D:\sources\install.wim /Index:6 /ApplyDir:W:\
```

([Microsoft Learn][4])

3. **Make bootable**

```bat
bcdboot W:\Windows /s S: /f UEFI
```

([Microsoft Learn][10])

4. **Optional servicing**

```bat
dism /Image:W:\ /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:6
sfc /scannow /offbootdir=W:\ /offwindir=W:\Windows
```

([Microsoft Learn][2])

5. **Enable WinRE (if you created T: for tools)**

```bat
reagentc /setreimage /path T:\Recovery\WindowsRE /target W:\Windows
reagentc /enable
```

([Microsoft Learn][14])

6. **Reboot** and finish OOBE.

---

## Repair-only example (existing Windows that will not boot)

1. Boot WinPE → `diskpart` → identify the **offline Windows** volume (say it is `C:`).
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

4. **Re-enable WinRE** (optional):
   `reagentc /enable` then `reagentc /info`. ([Microsoft Learn][17])

---

## Capturing and re-using your own image (for semi-automated rollouts)

* **Generalize** the reference machine with **Sysprep** so it is hardware-agnostic.
  Then **capture** with DISM and later **apply** to targets, as in Part 4. ([Microsoft Learn][18])

```bat
:: On reference PC (after Sysprep shutdown), from WinPE:
dism /Capture-Image /ImageFile:N:\Images\MyWin11.wim /CaptureDir:C:\ /Name:"Win11_Ref"
```

* Microsoft also documents capturing **partitions individually** and when to capture or skip ESP/MSR/Recovery. ([Microsoft Learn][19])

* If you need block-level cloning without file-system awareness, consider **FFU** workflow (outside scope here). ([Microsoft Learn][20])

---

## Why these sources and why you should trust them

* **Microsoft Learn / Docs**: Primary, authoritative references for each command and partitioning rule.

  * DISM image management and repair, including `/Apply-Image`, `/Get-WimInfo`, and `/RestoreHealth`. ([Microsoft Learn][21])
  * **BCDBoot** usage and parameters (`/s`, `/f UEFI|BIOS`). ([Microsoft Learn][10])
  * **SFC** options for offline repair (`/offbootdir`, `/offwindir`). ([Microsoft Learn][15])
  * **DiskPart** creation of the EFI partition and the **default UEFI layout** (ESP + MSR + Windows + Recovery). ([Microsoft Learn][7])
  * **WinPE creation** (`copype`, `MakeWinPEMedia`) and customization. ([Microsoft Learn][1])
  * **Drivers / Features** servicing offline (`/Add-Driver`, NetFx3). ([Microsoft Learn][11])
  * **Registry hive load** for offline edits. ([Microsoft Learn][13])
  * **WinRE deployment / reagentc**. ([Microsoft Learn][14])
* **Windows Central** article is recent and aligns with Microsoft guidance on DISM + SFC order; treated as **secondary** context, not a source of truth. ([Windows Central][22])

I leaned on built-in knowledge only to stitch steps together and to warn about common foot-guns; every command and policy decision is linked to a primary Microsoft page so you can verify independently.

---

## Appendix — PowerShell equivalents and useful one-liners

* Mount a WIM, service it, then commit:

```powershell
Mount-WindowsImage -ImagePath D:\sources\install.wim -Index 6 -Path C:\Mount
# ... servicing ...
Dismount-WindowsImage -Path C:\Mount -Save
```

([Microsoft Learn][23])

* Create WinPE working files and media (already shown above with CMD); the **same docs** show PowerShell-friendly contexts. ([Microsoft Learn][24])

---

If you want, I can tailor these scripts to **your exact disk sizes, firmware, and ISO**, and output a single, annotated `.cmd` you can run end-to-end.

[1]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11&utm_source=chatgpt.com "Create bootable Windows PE media"
[2]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11&utm_source=chatgpt.com "Repair a Windows Image"
[3]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro?view=windows-11&utm_source=chatgpt.com "Windows PE (WinPE)"
[4]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/take-inventory-of-an-image-or-component-using-dism?view=windows-11&utm_source=chatgpt.com "Take Inventory of an Image or Component Using DISM"
[5]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions?view=windows-11&utm_source=chatgpt.com "UEFI/GPT-based hard drive partitions"
[6]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/copype-command-line-options?view=windows-11&utm_source=chatgpt.com "Copype Command-Line Options"
[7]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-efi?utm_source=chatgpt.com "create partition efi"
[8]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/bcdboot?utm_source=chatgpt.com "bcdboot"
[9]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism---deployment-image-servicing-and-management-technical-reference-for-windows?view=windows-11&utm_source=chatgpt.com "DISM - Deployment Image Servicing and Management"
[10]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di?view=windows-11&utm_source=chatgpt.com "BCDBoot Command-Line Options"
[11]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/add-and-remove-drivers-to-an-offline-windows-image?view=windows-11&utm_source=chatgpt.com "Add and Remove Driver packages to an Offline Windows ..."
[12]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-net-framework-35-by-using-deployment-image-servicing-and-management--dism?view=windows-11&utm_source=chatgpt.com "Deploy .NET Framework 3.5 by using Deployment Image ..."
[13]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/reg-load?utm_source=chatgpt.com "reg load"
[14]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-re?view=windows-11&utm_source=chatgpt.com "Deploy Windows RE"
[15]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sfc?utm_source=chatgpt.com "sfc"
[16]: https://learn.microsoft.com/en-us/powershell/module/dism/dismount-windowsimage?view=windowsserver2025-ps&utm_source=chatgpt.com "Dismount-WindowsImage (Dism)"
[17]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options?view=windows-11&utm_source=chatgpt.com "REAgentC command-line options"
[18]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--generalize--a-windows-installation?view=windows-11&utm_source=chatgpt.com "Sysprep (Generalize) a Windows installation"
[19]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/capture-and-apply-windows-system-and-recovery-partitions?view=windows-11&utm_source=chatgpt.com "Capture and Apply Windows, System, and Recovery ..."
[20]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-using-full-flash-update--ffu?view=windows-11&utm_source=chatgpt.com "Capture and apply Windows Full Flash Update (FFU) images"
[21]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-image-management-command-line-options-s14?view=windows-11&utm_source=chatgpt.com "DISM Image Management Command-Line Options"
[22]: https://www.windowscentral.com/microsoft/windows-help/how-to-use-dism-command-tool-to-repair-windows-10-image?utm_source=chatgpt.com "How to use DISM command tool to repair Windows 10 image"
[23]: https://learn.microsoft.com/en-us/powershell/module/dism/mount-windowsimage?view=windowsserver2025-ps&utm_source=chatgpt.com "Mount-WindowsImage (Dism)"
[24]: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/makewinpemedia-command-line-options?view=windows-11&utm_source=chatgpt.com "Makewinpemedia Command-Line Options"
