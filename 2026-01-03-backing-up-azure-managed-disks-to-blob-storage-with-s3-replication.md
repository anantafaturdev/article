---
title: Backing Up Azure Managed Disks to Blob Storage with S3 Replication
slug: backing-up-azure-managed-disks-to-blob-storage-with-s3-replication
date_published: 2026-01-03T08:25:16.000Z
date_updated: 2026-01-03T08:25:16.000Z
excerpt: Automated disk-level backup of Azure Managed Disks to Blob Storage with snapshots, secure SAS, VHD export, and optional S3 replication via rclone.
---

A few days ago, my senior asked me to check how **Azure Managed Disks** could be backed up to **Azure Blob Storage** and to write the steps clearly.

There was no requirement for migration, no architectural redesign, and no Azure Backup policies involved. The scope was intentionally small: disk-level backup only.

Instead of stopping at documentation or copying commands from the Azure docs, I wanted to validate that the process actually worked end to end. I ran every step myself, verified the outputs, and automated the parts that made sense. This post documents that process from start to finish.

---

## The Problem, Reduced to Its Core

Once stripped of context and tooling preferences, the problem became very simple. A managed disk exists inside Azure, the backup should not be tied to the VM lifecycle, and the backup should live in object storage.

That naturally leads to a straightforward flow:

    Azure Managed Disk → Snapshot → Azure Blob Storage
    

Snapshots provide a crash-consistent copy of the disk, and Blob Storage provides durable and inexpensive object storage. Nothing else is required to satisfy the task.

---

## Why Disk-Level Backup Was the Right Choice

The task explicitly focused on managed disks rather than virtual machines or Azure Backup policies. Working at the disk level keeps the process simple and predictable.

Disk-level backups work for both OS disks and data disks, produce portable VHD files, and avoid introducing long-running backup services or policy management. For ad-hoc or one-time backups, this level of abstraction is easier to reason about and easier to document.

---

## Making the Environment Reproducible

Before touching the backup logic, I created a small lab environment using Terraform. The environment includes a lightweight Linux VM, an OS disk, an attached managed data disk, and a storage account with a private Blob container.

Nothing is oversized and everything is disposable. The goal was not to build realistic infrastructure, but to eliminate guesswork. With a consistent environment, the backup flow can be validated properly and repeated without manual setup.

All Terraform and automation scripts are available here:
[https://github.com/anantafaturdev/yt-anantafatur/tree/main/azure](https://github.com/anantafaturdev/yt-anantafatur/tree/main/azure)

---

## Discovering Managed Disks Automatically

Instead of hardcoding disk names, the backup script dynamically discovers all managed disks in the target resource group.

    az disk list \
      --resource-group rg-vm-backup-poc \
      --query "[].name" -o tsv
    

This mirrors how the script would behave in a real environment. Any disk that exists in the resource group is included automatically, whether it is an OS disk or a data disk. No manual tracking or updates are required as disks are added or removed.

---

## Creating Snapshots (The Actual Backup Step)

For each discovered disk, the script creates a snapshot.

    az snapshot create \
      --resource-group "$SOURCE_RG" \
      --name "$SNAPSHOT_NAME" \
      --source "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$SOURCE_RG/providers/Microsoft.Compute/disks/$DISK" \
      --sku Standard_LRS \
      --incremental false \
      --tags backup=true type=crash-consistent source_disk="$DISK"
    

This is the moment where the disk state is captured. Snapshots are fast to create, relatively inexpensive, and provide crash-consistent copies, which are sufficient for most disk-level backup use cases. At this stage, Azure already holds a safe copy of the disk, but the snapshot still lives inside Azure’s managed disk ecosystem.

---

## Granting Temporary Access to the Snapshot

Snapshots are private by default, so Azure requires a temporary SAS token to export them.

    az snapshot grant-access \
      --resource-group "$SOURCE_RG" \
      --name "$SNAPSHOT_NAME" \
      --duration-in-seconds "$SNAPSHOT_TTL" \
      --access-level Read \
      --query accessSAS -o tsv
    

This command grants read-only access for a limited duration. Once the SAS token expires, the snapshot becomes inaccessible again. This keeps the export process secure by ensuring that access exists only for as long as it is needed.

---

## Exporting the Snapshot to Azure Blob Storage

With the SAS URI available, the snapshot can be copied directly into Azure Blob Storage as a VHD file.

    az storage blob copy start \
      --account-name "$TARGET_STORAGE" \
      --account-key "$STORAGE_KEY" \
      --destination-container "$TARGET_CONTAINER" \
      --destination-blob "$BLOB_NAME" \
      --source-uri "$SNAPSHOT_URI"
    

After this step completes, the backup is no longer tied to the virtual machine or the snapshot lifecycle. It exists as a plain object in Blob Storage, which means it can be retained, replicated, or deleted independently of the original disk.

At this point, the original requirement is already fulfilled.

---

## Optional: Replicating the Backup to Amazon S3

The task mentioned comparing Blob Storage and S3 pricing, but did not require cross-cloud replication. Since the backup already exists as an object, adding replication is trivial.

I included an optional step using `rclone` to copy the VHD from Blob Storage to Amazon S3.

    rclone copy \
      blob:$TARGET_CONTAINER/$BLOB_NAME \
      "$S3_OBJECT" \
      --progress \
      --transfers 3 \
      --buffer-size 64M \
      --ignore-size
    

This is not a migration. It is closer to a simple disaster recovery idea. Once a backup exists in object storage, copying it elsewhere becomes a straightforward operation. This step can be removed entirely without affecting the core backup flow.

---

## Automation Strategy

All steps are automated, but responsibilities are kept separate. Terraform handles infrastructure provisioning, while a Bash script handles the backup logic, including disk discovery, snapshot creation, SAS handling, Blob export, and optional S3 replication.

---

## Cleanup

Because everything is provisioned using Terraform, cleanup is straightforward.

    terraform destroy
    

This removes all resources created for the lab environment and ensures there are no orphaned disks, storage accounts, or unexpected costs left behind.

---

## Closing Thoughts

This setup focuses on doing one thing well: backing up Azure Managed Disks into durable object storage in a way that is simple, verifiable, and easy to repeat.

There is no overengineering, no hidden services, and no unnecessary abstraction. It is just a clean disk-level backup flow that can be validated end to end and explained clearly.
