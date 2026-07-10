---
title: Expanding EC2 Root EBS Volume Without Downtime
slug: expanding-ec2-root-ebs-volume-without-downtime
date_published: 2026-05-24T05:38:48.000Z
date_updated: 2026-05-24T05:38:48.000Z
excerpt: Expanded an EC2 root EBS system volume on Ubuntu 24.04.4 LTS (ARM) without downtime using Terraform.
---

Ubuntu 24.04.4 LTS (ARM), root volume on EBS, managed via Terraform.

Disk usage was already near full:

    root@anantafatur:~/anantafatur-tfworkload# df -h
    Filesystem       Size  Used Avail Use% Mounted on
    /dev/root         23G   22G  1.2G  95% /
    

Current disk layout:

    root@anantafatur:~/anantafatur-tfworkload# lsblk
    NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    loop0          7:0    0 24.2M  1 loop /snap/amazon-ssm-agent/13008
    loop1          7:1    0   69M  1 loop /snap/core22/2412
    loop2          7:2    0 42.6M  1 loop /snap/snapd/26869
    nvme0n1      259:0    0   24G  0 disk
    ├─nvme0n1p1  259:1    0   23G  0 part /
    ├─nvme0n1p15 259:2    0   99M  0 part /boot/efi
    └─nvme0n1p16 259:3    0  923M  0 part /boot
    

Updated only the root volume size in Terraform:

    root_block_device {
      volume_size = 26
    }
    

Terraform showed in-place modification:

    root@anantafatur:~/anantafatur-tfworkload# terraform plan
    
    Terraform will perform the following actions:
    
      # aws_instance.main will be updated in-place
      ~ resource "aws_instance" "main" {
    
          ~ root_block_device {
              ~ volume_size = 24 -> 26
            }
        }
    
    Plan: 0 to add, 1 to change, 0 to destroy.
    

Applied:

    root@anantafatur:~/anantafatur-tfworkload# terraform apply
    

After apply, EBS volume size increased immediately, but partition size was still unchanged:

    root@anantafatur:~/anantafatur-tfworkload# lsblk
    NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    loop0          7:0    0 24.2M  1 loop /snap/amazon-ssm-agent/13008
    loop1          7:1    0   69M  1 loop /snap/core22/2412
    loop2          7:2    0 42.6M  1 loop /snap/snapd/26869
    nvme0n1      259:0    0   26G  0 disk
    ├─nvme0n1p1  259:1    0   23G  0 part /
    ├─nvme0n1p15 259:2    0   99M  0 part /boot/efi
    └─nvme0n1p16 259:3    0  923M  0 part /boot
    

Expanded partition online:

    root@anantafatur:~/anantafatur-tfworkload# sudo growpart /dev/nvme0n1 1
    
    CHANGED: partition=1 start=2099200 old: size=48232415 end=50331614 new: size=52426719 end=54525918
    

Then resized ext4 filesystem:

    root@anantafatur:~/anantafatur-tfworkload# sudo resize2fs /dev/nvme0n1p1
    
    resize2fs 1.47.0 (5-Feb-2023)
    Filesystem at /dev/nvme0n1p1 is mounted on /; on-line resizing required
    old_desc_blocks = 3, new_desc_blocks = 4
    The filesystem on /dev/nvme0n1p1 is now 6553339 (4k) blocks long.
    

Final result:

    root@anantafatur:~/anantafatur-tfworkload# df -h
    Filesystem       Size  Used Avail Use% Mounted on
    /dev/root         25G   22G  3.1G  88% /
    

    root@anantafatur:~/anantafatur-tfworkload# lsblk
    NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    loop0          7:0    0 24.2M  1 loop /snap/amazon-ssm-agent/13008
    loop1          7:1    0   69M  1 loop /snap/core22/2412
    loop2          7:2    0 42.6M  1 loop /snap/snapd/26869
    nvme0n1      259:0    0   26G  0 disk
    ├─nvme0n1p1  259:1    0   25G  0 part /
    ├─nvme0n1p15 259:2    0   99M  0 part /boot/efi
    └─nvme0n1p16 259:3    0  923M  0 part /boot
    

Notes:

- AWS volume state was still `optimizing` while resizing filesystem. No issue observed.
- Nitro instances expose EBS as NVMe (`/dev/nvme0n1`).
- `growpart` came from `cloud-guest-utils`.
- No reboot needed during the process.
