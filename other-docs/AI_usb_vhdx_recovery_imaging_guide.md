# USB recovery drive imaging guide: Windows VHDX backup, then restore on Windows, Linux, or macOS

**NOTE: This guide was created using AI (LLM / ChatGPT) assistance**

This guide covers a whole-device USB backup workflow:

1. Create a `.vhdx` image of a recovery USB in Windows using Rufus.
2. Inspect whether the image can fit on the target USB.
3. Restore the image to a USB on Windows, Linux, or macOS.
4. Handle the case where the target USB is smaller than the original USB.

The core rule is simple: the target USB must be large enough for the restored **disk layout**, not merely large enough for the compressed or sparse `.vhdx` file.

---

## Terms

**Whole-device image**  
An image of the entire USB device, including the partition table, boot code, all partitions, and any unallocated space. This is what you want for OEM recovery media.

**Partition image**  
An image of only one partition, such as only the visible Windows drive letter. This is usually not enough for multi-partition recovery USBs.

**VHDX virtual size**  
The disk size the image claims to be when restored or mounted. A `.vhdx` file can be smaller on your storage drive because it is dynamic or sparse, but its virtual size can still be 64 GB.

**Raw image / DD image**  
A byte-for-byte disk image. Linux and macOS can write this directly to a USB with `dd`. A `.vhdx` should usually be converted to raw before writing with `dd`.

**Tail unallocated space**  
Unused space after the final partition. If the original USB had unused space at the end, the image may be reducible without touching any filesystems.

---

## Safety rules

1. Double-check the target disk before writing. These commands overwrite entire USB devices.
2. Write to the whole disk, not a partition.
   - Linux: use `/dev/sdX`, not `/dev/sdX1`.
   - macOS: use `/dev/rdiskN`, not `/dev/rdiskNs1`.
3. Keep the original `.vhdx` unchanged. Work on a copy when shrinking.
4. Do not rely on an ISO as the primary backup for OEM recovery USBs. An ISO may not preserve a multi-partition USB layout.
5. If the image uses GPT, blind truncation can break the backup GPT header. Repair or relocate GPT metadata after shrinking/truncating.

---

## Part 1: Image the USB in Windows with Rufus as VHDX

1. Insert only:
   - the source recovery USB, and
   - the drive where you will save the image.
2. Open Rufus as Administrator.
3. Under **Device**, select the recovery USB.
4. Expand **Show advanced drive properties**.
5. Click the **Save** icon next to the selected drive.
6. Save the backup as a `.vhdx` file.
7. Store the image somewhere safe.

Optional hash after imaging:

```powershell
Get-FileHash -Algorithm SHA256 .\MSI-Recovery.vhdx |
  Format-List > .\MSI-Recovery.vhdx.sha256.txt
```

Keep that hash file with the image.

---

## Part 2: Decide whether the target USB is large enough

A target USB marked “32 GB” or “64 GB” may be smaller in exact bytes than another USB with the same advertised capacity. Always compare byte sizes.

### Windows: check USB byte size

Open PowerShell as Administrator:

```powershell
Get-Disk | Format-Table Number,FriendlyName,BusType,Size,PartitionStyle
```

Find the target USB by size/model. Note its `Number` and exact `Size`.

### Linux: check USB byte size

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
sudo blockdev --getsize64 /dev/sdX
```

Replace `/dev/sdX` with the whole USB device.

### macOS: check USB byte size

```bash
diskutil list external
diskutil info /dev/diskN | grep "Disk Size"
```

Replace `diskN` with the external USB disk identifier.

---

## Part 3: Inspect the VHDX image

### Linux or macOS

Install QEMU tools, then run:

```bash
qemu-img info MSI-Recovery.vhdx
```

Look for `virtual size`. If the target USB is equal to or larger than that virtual size, restoration is straightforward.

To inspect partitions more easily, convert to raw:

```bash
qemu-img convert -p -f vhdx -O raw MSI-Recovery.vhdx MSI-Recovery.raw
parted MSI-Recovery.raw unit B print
```

Look at the final partition’s `End` value. If that final `End` value is below the target USB byte size, the image may fit after truncating unused tail space.

### Windows

If the Hyper-V PowerShell module is available, inspect the image read-only:

```powershell
Mount-VHD -Path .\MSI-Recovery.vhdx -ReadOnly -PassThru |
  Get-Disk |
  Get-Partition |
  Sort-Object Offset |
  Format-Table DiskNumber,PartitionNumber,Offset,Size,Type,GptType
```

Then detach it:

```powershell
Dismount-VHD -Path .\MSI-Recovery.vhdx
```

You can also use Disk Management: **Action → Attach VHD**, preferably read-only for inspection.

---

# Restore paths

## Path A: Restore on Windows

Use this when the target USB is the same size or larger than the VHDX virtual size.

1. Open Rufus as Administrator.
2. Select the target USB under **Device**.
3. Under boot/image selection, choose the `.vhdx` image.
4. Start the write process.
5. If Rufus asks for image mode, use the raw/DD-style mode.
6. Wait for completion, then safely eject the USB.

If Rufus refuses the `.vhdx`, convert it to raw with `qemu-img` for Windows, then write the raw `.img` with Rufus or another whole-device USB imaging tool:

```powershell
qemu-img.exe convert -p -f vhdx -O raw .\MSI-Recovery.vhdx .\MSI-Recovery.raw.img
```

Then select `MSI-Recovery.raw.img` in Rufus and write it to the target USB.

---

## Path B: Restore on Linux

Install tools:

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install qemu-utils parted gparted gdisk
```

Convert VHDX to raw:

```bash
qemu-img convert -p -f vhdx -O raw MSI-Recovery.vhdx MSI-Recovery.raw
```

Identify the target USB:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

Unmount any mounted target USB partitions:

```bash
sudo umount /dev/sdX?* 2>/dev/null || true
```

Write the image to the whole USB:

```bash
sudo dd if=MSI-Recovery.raw of=/dev/sdX bs=16M status=progress conv=fsync
sync
```

Replace `/dev/sdX` with the whole USB device.

---

## Path C: Restore on macOS

Install QEMU tools, for example with Homebrew:

```bash
brew install qemu
```

Convert VHDX to raw:

```bash
qemu-img convert -p -f vhdx -O raw MSI-Recovery.vhdx MSI-Recovery.raw
```

Identify the USB:

```bash
diskutil list external
```

Unmount the USB disk:

```bash
diskutil unmountDisk /dev/diskN
```

Write to the raw disk device for better speed:

```bash
sudo dd if=MSI-Recovery.raw of=/dev/rdiskN bs=16m
sync
diskutil eject /dev/diskN
```

Replace `diskN` and `rdiskN` with the target USB disk number. During `dd`, press `Ctrl+T` to show progress on macOS.

---

# Restoring to a smaller USB

A smaller target USB works only if the final usable disk layout can fit. There are three cases.

## Case 1: The image already has enough unallocated space at the end

This is the easiest smaller-USB case. You do not need to shrink filesystems; you only remove unused tail space from the raw image.

### Linux

Convert to raw if needed:

```bash
qemu-img convert -p -f vhdx -O raw MSI-Recovery.vhdx MSI-Recovery.raw
```

Inspect partition ends:

```bash
parted MSI-Recovery.raw unit B print
```

Choose a new image size that is:

- larger than the final partition `End` value,
- smaller than the target USB byte size,
- preferably rounded up to a MiB boundary.

Example:

```bash
cp --sparse=always MSI-Recovery.raw MSI-Recovery-small.raw
truncate -s 32000000000 MSI-Recovery-small.raw
```

If the image is GPT, repair/relocate the backup GPT after truncation:

```bash
sgdisk -e MSI-Recovery-small.raw
```

Then write the smaller raw image:

```bash
sudo dd if=MSI-Recovery-small.raw of=/dev/sdX bs=16M status=progress conv=fsync
sync
```

### macOS

Convert to raw, inspect the layout using a suitable partition tool, then truncate if the final partition ends before the target USB size:

```bash
cp MSI-Recovery.raw MSI-Recovery-small.raw
truncate -s 32000000000 MSI-Recovery-small.raw
```

If the image is GPT, install `gptfdisk` and repair/relocate the backup GPT:

```bash
brew install gptfdisk
sgdisk -e MSI-Recovery-small.raw
```

Then write `MSI-Recovery-small.raw` to the USB with `dd` as shown above.

## Case 2: The final partition is too large, but the filesystem has free space

You must shrink the final partition before writing to the smaller USB.

### Linux with GParted

Convert to raw:

```bash
qemu-img convert -p -f vhdx -O raw MSI-Recovery.vhdx MSI-Recovery.raw
```

Attach the raw image as a loop disk:

```bash
LOOP=$(sudo losetup --find --show -P MSI-Recovery.raw)
echo "$LOOP"
```

Open GParted on the loop disk:

```bash
sudo gparted "$LOOP"
```

In GParted:

1. Select the loop device, such as `/dev/loop0`.
2. Shrink the final partition so that enough unallocated space exists at the end.
3. Apply the operation.
4. Close GParted.

Detach the loop device:

```bash
sudo losetup -d "$LOOP"
```

Then inspect and truncate as in Case 1:

```bash
parted MSI-Recovery.raw unit B print
cp --sparse=always MSI-Recovery.raw MSI-Recovery-small.raw
truncate -s <NEW_SIZE_IN_BYTES> MSI-Recovery-small.raw
```

If GPT:

```bash
sgdisk -e MSI-Recovery-small.raw
```

Then flash the smaller raw image.

### Windows

Make a working copy first:

```powershell
Copy-Item .\MSI-Recovery.vhdx .\MSI-Recovery-small.vhdx
```

Attach the copy read-write:

```powershell
Mount-VHD -Path .\MSI-Recovery-small.vhdx -PassThru |
  Get-Disk |
  Get-Partition |
  Sort-Object Offset |
  Format-Table DiskNumber,PartitionNumber,DriveLetter,Offset,Size,Type
```

Use Disk Management or DiskPart to shrink the final volume. DiskPart example:

```text
diskpart
list volume
select volume <VOLUME_NUMBER>
shrink desired=<MB_TO_REMOVE>
exit
```

Detach the VHDX:

```powershell
Dismount-VHD -Path .\MSI-Recovery-small.vhdx
```

Shrink the VHDX container itself:

```powershell
Resize-VHD -Path .\MSI-Recovery-small.vhdx -ToMinimumSize
```

If needed, inspect the resulting virtual size:

```powershell
Get-VHD .\MSI-Recovery-small.vhdx | Format-List Path,Size,MinimumSize,FileSize
```

Then restore `MSI-Recovery-small.vhdx` with Rufus. If Rufus still reports that the image is too large, the partition layout probably still does not fit the target USB.

### macOS

macOS is fine for converting and writing the image, but it is not the best place to shrink Windows/OEM recovery partitions inside a raw USB image. Use the Linux/GParted path or the Windows path to shrink the image, then return to macOS for the final `dd` write if desired.

## Case 3: The data layout cannot fit

If the final partition cannot be shrunk enough, do not force the image onto the smaller USB. Use a target USB with equal or greater exact byte capacity.

---

## Quick decision table

| Situation | Recommended path |
|---|---|
| Target USB is same size or larger | Restore VHDX with Rufus on Windows, or convert to raw and use `dd` on Linux/macOS. |
| Target USB is smaller, but final partition ends before the target size | Convert to raw, truncate unused tail, repair GPT if applicable, then write. |
| Target USB is smaller and final partition is too large | Shrink the final partition with GParted or Windows Disk Management/DiskPart, then shrink/truncate the image. |
| Target USB is smaller and partitions cannot be shrunk enough | Use a larger USB. |
| You have both `.iso` and `.vhdx` backups | Prefer `.vhdx` for OEM recovery media because it preserves the disk layout. |

---

## Post-restore checks

1. Boot-test the restored USB on the target machine.
2. Confirm that the recovery environment starts.
3. Keep the original `.vhdx` and checksum until you have tested the replacement USB.
4. If the restored USB is larger than the image, leaving extra unallocated space at the end is usually harmless.

---

## Troubleshooting

### Rufus or `dd` says the target is too small

Compare exact byte sizes. The target must be larger than the image’s virtual size or the truncated raw image size.

### Windows shows only one partition from the recovery USB

That does not prove the USB has only one partition. Windows may hide EFI, recovery, or OEM partitions. Use whole-device imaging, not drive-letter imaging.

### The VHDX file is much smaller than the original USB

That can be normal for a dynamic or sparse VHDX. Use virtual size, not file size, when deciding whether it fits a target USB.

### GParted cannot resize a partition

The filesystem may not be supported for shrinking, may be damaged, or may not have enough free space. Do not force a shrink operation on recovery media unless you have a backup.

### GPT warning after writing or truncating

If the image used GPT and the disk size changed, the backup GPT may be in the wrong place. On Linux, use:

```bash
sudo sgdisk -e /dev/sdX
```

Use this only on the correct target USB.

---

## Sources

- [Rufus FAQ: saving an existing drive to VHD](https://github.com/pbatard/rufus/wiki/FAQ#:~:text=Saving%20an%20existing%20drive%20to%20VHD)
- [Rufus project features](https://github.com/pbatard/rufus#:~:text=Create%20bootable%20drives%20from%20bootable%20disk%20images)
- [QEMU `qemu-img` documentation](https://qemu-project.gitlab.io/qemu/tools/qemu-img.html#:~:text=qemu-img%20allows%20you%20to%20create%2C%20convert%20and%20modify%20images%20offline)
- [GParted project description](https://gparted.org/#:~:text=With%20GParted%20you%20can%20resize%2C%20copy%2C%20and%20move%20partitions)
- [Microsoft `Mount-DiskImage`](https://learn.microsoft.com/en-us/powershell/module/storage/mount-diskimage?view=windowsserver2025-ps#:~:text=mounts%20a%20previously%20created%20disk%20image)
- [Microsoft `Mount-VHD`](https://learn.microsoft.com/en-us/powershell/module/hyper-v/mount-vhd?view=windowsserver2025-ps#:~:text=mounts%20a%20virtual%20hard%20disk%20in%20read-only%20mode)
- [Microsoft DiskPart `shrink`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shrink#:~:text=reduces%20the%20size%20of%20the%20selected%20volume)
- [Microsoft `Resize-VHD`](https://learn.microsoft.com/en-us/powershell/module/hyper-v/resize-vhd?view=windowsserver2025-ps#:~:text=can%20shrink%20only%20VHDX)
- [Apple Disk Utility: unmount a disk](https://support.apple.com/guide/disk-utility/unmount-a-disk-set-or-disk-member-dskud709f49b/mac#:~:text=Unmount%20a%20disk%20set%20or%20disk%20member)
