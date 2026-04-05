# ☁️ AWS EC2 Storage Management 

> **Real-world cloud storage operations on AWS EC2 — covering EBS volume provisioning, live disk resizing, partition extension, and OS-level file system management on both Linux and Windows Server.**

---

## 📌 Overview

This repository documents my hands-on practice with **Amazon Web Services EC2 storage management**, where I worked through real-world scenarios involving EBS (Elastic Block Store) volumes, disk modifications, and OS-level storage configuration. All tasks were performed on live EC2 instances running **Amazon Linux 2023** and **Windows Server 2022**.

---

## 🧰 Tech Stack & Tools

| Category | Tools Used |
|---|---|
| Cloud Platform | AWS EC2, AWS Console |
| Linux OS | Amazon Linux 2023 |
| Windows OS | Windows Server 2022 |
| Storage Service | Amazon EBS (Elastic Block Store) |
| Linux Tools | `lsblk`, `fdisk`, `growpart`, `resize2fs`, `xfs_growfs`, `df`, `mount` |
| Windows Tools | Disk Management GUI, `diskpart`, PowerShell |
| Access Method | EC2 Instance Connect (browser-based SSH), RDP |

---

## ✅ What I Implemented

### 1. 🚀 Launched EC2 Instances

- Launched **Amazon Linux 2023** EC2 instance (`t2.micro`) in `ap-south-1` (Mumbai)
- Launched **Windows Server 2022** EC2 instance (`t3.micro`) in `ap-south-1`
- Configured Security Groups, Key Pairs, and IAM roles appropriately
- Connected to Linux instance via **EC2 Instance Connect** (browser SSH)
- Connected to Windows instance via **RDP** using decrypted password

---

### 2. 💾 Added New EBS Volumes (Drives)

#### Linux – Attach & Mount a New EBS Volume

```bash
# Step 1: List existing block devices
lsblk

# Step 2: Check the new volume (e.g., /dev/xvdf or /dev/nvme1n1)
sudo lsblk -f

# Step 3: Create a filesystem on the new volume
sudo mkfs -t xfs /dev/nvme1n1

# Step 4: Create a mount point
sudo mkdir /data

# Step 5: Mount the volume
sudo mount /dev/nvme1n1 /data

# Step 6: Verify
df -hT
```

#### Persist Mount Across Reboots (fstab)

```bash
# Get UUID of the new volume
sudo blkid /dev/nvme1n1

# Add to /etc/fstab
echo "UUID=<your-uuid>  /data  xfs  defaults,nofail  0  2" | sudo tee -a /etc/fstab

# Test the fstab entry
sudo mount -a
```

#### Windows – Attach & Initialize a New EBS Volume

1. Attached new EBS volume (`5 GB`) from AWS Console to the Windows instance
2. Opened **Disk Management** → `diskmgmt.msc`
3. Initialized the disk as **GPT**
4. Created a **New Simple Volume** and formatted as **NTFS**
5. Assigned drive letter **D:** → Volume labeled `New Volume`

> Result: `New Volume (D:)` — 4.96 GB free of 4.98 GB (visible in `This PC`)

---

### 3. 📏 Modified Existing Drive Sizes (EBS Volume Resize)

> Resized EBS root volume from **8 GB → 16 GB** via AWS Console without stopping the instance.

#### Steps in AWS Console

1. Go to **EC2 → Volumes**
2. Select the root volume → **Actions → Modify Volume**
3. Changed size from `8 GB` to `16 GB`
4. Clicked **Modify** → Confirmed

> ⚠️ Resizing the EBS volume in AWS does NOT automatically extend the OS file system — that requires OS-level steps below.

---

### 4. 🔄 Extended Partitions & File Systems (Linux)

After resizing the EBS volume in AWS, extended the partition and file system inside the OS:

```bash
# Step 1: Confirm the new disk size is recognized
lsblk
# nvme0n1   259:0    0  16G  0 disk
# └─nvme0n1p1 259:1  0   8G  0 part /   <-- still shows old size

# Step 2: Grow the partition (partition 1 on nvme0n1)
sudo growpart /dev/nvme0n1 1

# Step 3: Verify partition has expanded
lsblk
# └─nvme0n1p1 259:1  0  16G  0 part /   <-- partition now shows full size

# Step 4a: Extend XFS filesystem (Amazon Linux 2023 default)
sudo xfs_growfs /

# Step 4b: OR extend ext4 filesystem
sudo resize2fs /dev/nvme0n1p1

# Step 5: Verify the filesystem sees the new space
df -hT /
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1 xfs    16G  2.1G   14G  14% /
```

---

### 5. 🪟 Extended Partitions & File Systems (Windows)

After resizing the EBS volume in AWS, extended the volume in Windows:

**Using Disk Management GUI:**
1. Opened `diskmgmt.msc`
2. Right-clicked `C:` drive → **Extend Volume**
3. Extended to use all unallocated space
4. Verified in **This PC** → Windows (C:) now shows expanded capacity

**Using diskpart (CLI method):**
```cmd
diskpart
DISKPART> list disk
DISKPART> select disk 0
DISKPART> list partition
DISKPART> select partition 2
DISKPART> extend
DISKPART> exit
```

---

## 📊 Instance Details

### Linux Instance

| Property | Value |
|---|---|
| OS | Amazon Linux 2023 |
| Instance ID | `i-0cb1bda002efc72ac` |
| Private IP | `172.31.35.13` |
| Public IP | `3.111.144.234` |
| Region | ap-south-1 (Mumbai) |
| Root Disk | `/dev/nvme0n1` — 8 GB |
| Partition Layout | nvme0n1p1 (8G, `/`), nvme0n1p127 (1M), nvme0n1p128 (10M, `/boot/efi`) |

### Windows Instance

| Property | Value |
|---|---|
| OS | Windows Server 2022 |
| Hostname | `EC2AMAZ-DVCB0UD` |
| Instance ID | `i-0b59acb46d882c8dc` |
| Private IP | `172.31.44.107` |
| Public IP | `3.110.77.186` |
| Instance Type | `t3.micro` |
| Availability Zone | ap-south-1a |
| Architecture | AMD64 |
| C: Drive | 29.4 GB (9.28 GB free) |
| D: Drive | 4.98 GB (4.96 GB free) |

---

## 🔍 Key Commands Reference

### Linux Block Device Commands

```bash
# List all block devices with partition info
lsblk

# List with filesystem types and UUIDs
lsblk -f

# Check disk space usage
df -hT

# Show partition table
sudo fdisk -l /dev/nvme0n1

# Grow a partition (requires cloud-utils-growpart)
sudo growpart /dev/nvme0n1 1

# Extend XFS filesystem
sudo xfs_growfs /

# Extend ext4 filesystem
sudo resize2fs /dev/nvme0n1p1

# Create XFS filesystem on new volume
sudo mkfs.xfs /dev/nvme1n1

# Create ext4 filesystem on new volume
sudo mkfs.ext4 /dev/nvme1n1

# Mount a volume
sudo mount /dev/nvme1n1 /mnt/data

# Unmount a volume
sudo umount /mnt/data

# View mount table
cat /etc/fstab
```

### Windows PowerShell Commands

```powershell
# List disks
Get-Disk

# List partitions
Get-Partition

# List volumes
Get-Volume

# Resize partition to maximum size
$drive = Get-Partition -DriveLetter C
$maxSize = (Get-PartitionSupportedSize -DiskNumber $drive.DiskNumber -PartitionNumber $drive.PartitionNumber).SizeMax
Resize-Partition -DiskNumber $drive.DiskNumber -PartitionNumber $drive.PartitionNumber -Size $maxSize

# Initialize a new disk
Initialize-Disk -Number 1 -PartitionStyle GPT

# Create new partition and format
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel "DataDrive"
```

---

## 📁 Repository Structure

```
aws-ec2-storage-management/
│
├── README.md                        # This file
│
├── linux/
│   ├── 01_attach_ebs_volume.sh      # Script: Attach & mount a new EBS volume
│   ├── 02_extend_root_partition.sh  # Script: Grow partition + extend filesystem
│   ├── 03_persistent_mount.sh       # Script: fstab configuration
│   └── notes.md                     # Linux-specific observations
│
├── windows/
│   ├── 01_initialize_disk.ps1       # PowerShell: Initialize new EBS volume
│   ├── 02_extend_volume.ps1         # PowerShell: Extend C: drive after resize
│   └── notes.md                     # Windows-specific observations
│
└── screenshots/
    ├── linux_lsblk_output.jpeg
    ├── windows_this_pc_drives.jpeg
    └── windows_desktop_instance_info.jpeg
```

---

## 📸 Screenshots

### 🐧 Linux EC2 – `lsblk` Output (Amazon Linux 2023)
> Connected via EC2 Instance Connect | Instance: `i-0cb1bda002efc72ac` | IP: `172.31.35.13`

![Linux EC2 lsblk output](screenshots/linux_lsblk_output.jpeg)

---

### 🪟 Windows EC2 – This PC (C: & D: Drives)
> After attaching and initializing a new EBS volume as `D:` | Hostname: `EC2AMAZ-DVCB0UD`

![Windows EC2 This PC drives](screenshots/windows_this_pc_drives.jpeg)

---

### 🖥️ Windows EC2 – Desktop with Instance Metadata
> Instance ID: `i-0b59acb46d882c8dc` | Public IP: `3.110.77.186` | Type: `t3.micro` | Zone: `ap-south-1a`

![Windows EC2 desktop](screenshots/windows_desktop_instance_info.jpeg)

---

## 💡 Key Learnings

- **EBS volumes are network-attached** — they can be detached and moved between instances in the same AZ
- **Resizing EBS in AWS ≠ resizing the OS filesystem** — both steps are always required
- **`growpart` + `xfs_growfs`** is the standard combo for Amazon Linux 2023 root volume expansion
- **Windows Disk Management** provides a GUI-friendly way to extend volumes but PowerShell is more scriptable for automation
- **`/etc/fstab` with `nofail`** is critical for persistent mounts — missing this caused boot failures during testing
- **NVMe naming** (`nvme0n1`) is used on Nitro-based instances, while older instances use `xvda`/`xvdf` style naming
- **GPT vs MBR** — modern AWS instances use GPT partition tables; the `nvme0n1p128` partition is the EFI system partition

---

## 🔗 References

- [AWS Docs – EBS Volumes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volumes.html)
- [AWS Docs – Extend Linux File System](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)
- [AWS Docs – Extend Windows File System](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-windows.html)
- [Amazon Linux 2023 Documentation](https://aws.amazon.com/linux/amazon-linux-2023/)

---

## 🏷️ Tags

`AWS` `EC2` `EBS` `Cloud Storage` `Linux` `Windows Server` `DevOps` `Cloud Computing` `Amazon Linux` `Disk Management` `Storage Management` `Hands-On` `ap-south-1`

---

> 🌐 *All experiments were performed on live AWS EC2 instances in the Asia Pacific (Mumbai) region.*
