# Replacing an NVMe Drive in an mdadm RAID1 Array

Swapping a drive in a Linux software RAID1 (mdadm) is routine, but a few NVMe-specific details make it easy to pull the wrong disk. This note walks through identifying the array, removing the drive, adding the replacement and estimating rebuild time.

## 1. Identify the array and its members

```bash
cat /proc/mdstat
```

Example output:

```
md1 : active raid1 nvme1n1[1] nvme0n1[0]
      937560384 blocks super 1.2 [2/2] [UU]
      bitmap: 2/7 pages [8KB], 65536KB chunk
```

From this:

- The array is `/dev/md1`.
- The members are **whole disks** (`nvme0n1`, `nvme1n1`). If they were partitions, you would see names like `nvme1n1p1`.
- `[2/2] [UU]` means both disks are healthy. A failed or missing member shows as `[U_]` or `[_U]`, or is marked `(F)`.

For more detail, including each member's state:

```bash
sudo mdadm --detail /dev/md1
```

If a disk holds partitions belonging to several arrays (for example `/boot`, swap and `/`), check each partition with `sudo mdadm --examine /dev/nvme1n1p1` and repeat the removal steps for every array.

## 2. Confirm which physical drive to remove

NVMe numbering (`nvme0`, `nvme1`) does not necessarily follow slot order, so always match by serial number:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL,SERIAL
sudo nvme list                  # needs nvme-cli
sudo smartctl -a /dev/nvme1n1   # health and serial number
```

Compare the serial number with the label on the physical drive before pulling it. If both members show `[UU]`, use SMART data to decide which disk is actually failing.

## 3. Fail and remove the drive

```bash
sudo mdadm /dev/md1 --fail /dev/nvme1n1 --remove /dev/nvme1n1
```

Verify that the array is now degraded but running:

```bash
cat /proc/mdstat    # expect [2/1] [U_] or [_U]
```

Before shutting down or pulling the disk, make sure the remaining disk can boot the system, for example by confirming the bootloader is installed on it.

If the drive is hot-swappable and you are not powering the server off, detach it from the kernel before pulling it. This is the NVMe equivalent of `echo 1 > /sys/block/sdX/device/delete` for SATA/SAS disks. NVMe has no `delete` attribute on the block device, so remove the underlying PCI device instead:

```bash
echo 1 | sudo tee /sys/block/nvme1n1/device/device/remove
```

Skip this step if you shut the server down to replace the drive.

## 4. Add the replacement drive

After installing the new drive, check its device name with `lsblk`. If you hot-swapped the drive and removed the PCI device in the previous step, rescan the PCI bus first so the kernel detects the new one:

```bash
echo 1 | sudo tee /sys/bus/pci/rescan
```

Then add it:

```bash
sudo mdadm /dev/md1 --add /dev/nvme1n1
```

If the array members are partitions, partition the new disk to match the surviving one first.

## 5. Rebuild time

mdadm works below the filesystem and does not know which blocks hold data. **A new disk always gets a full copy**, so a half-empty 1 TB array takes as long to rebuild as a full one.

The usual bottleneck is the kernel's resync throttle, not the NVMe drive. The default `speed_limit_max` is about 200 MB/s:

| Setting | Approx. time for ~937 GB |
|---|---|
| Default limit (~200 MB/s) | ~1–1.5 hours |
| Raised limit, fast NVMe (1–3 GB/s) | ~5–15 minutes |
| Raised limit, QLC drive after cache fills | ~20–30 minutes |
| Busy server (falls toward `speed_limit_min`) | Much longer |

To speed up the rebuild:

```bash
sudo sysctl -w dev.raid.speed_limit_max=2000000
sudo sysctl -w dev.raid.speed_limit_min=500000
```

Higher limits put more I/O load on the server, so raise `speed_limit_min` cautiously on production systems. The values reset at reboot.

Watch progress with:

```bash
watch -n 5 cat /proc/mdstat
```

## Re-adding the same disk

If a disk was removed only temporarily and is put back unchanged, the write-intent bitmap lets md resync just the regions that changed:

```bash
sudo mdadm /dev/md1 --re-add /dev/nvme1n1
```

This can finish in seconds or minutes. It does not apply to a new replacement disk.
