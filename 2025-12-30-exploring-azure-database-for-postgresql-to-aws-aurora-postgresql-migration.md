---
title: Exploring Azure Database for PostgreSQL to AWS Aurora PostgreSQL Migration
slug: exploring-azure-database-for-postgresql-to-aws-aurora-postgresql-migration
date_published: 2025-12-30T10:16:53.000Z
date_updated: 2025-12-30T10:16:53.000Z
excerpt: Exploring a secure, one-time migration of Azure Database for PostgreSQL to AWS Aurora PostgreSQL for private-network databases.
---

> This write-up was originally prepared as an internal report to my senior, but I’m publishing it here because database migration is a common engineering challenge, and sharing the technical approach may help others facing similar situations.

---

Our team planned a migration from **Azure Database for PostgreSQL** to **AWS Aurora PostgreSQL**. The motivation was **operational familiarity**:

- The team is more confident troubleshooting Aurora vs. Azure PostgreSQL.
- One-time migration, downtime is acceptable.
- Both databases are in **private subnets**; no public exposure.

**Approach:** Backup and restore using `pg_dump` → S3 → `pg_restore` via bastion hosts. Managed migration services (DMS) were avoided due to VPN/tunnel overhead and the one-time nature of this migration.

---

### **High-Level Architecture**

    [Azure PostgreSQL] (Private Subnet)
               |
               v
      [Azure Bastion VM] -- Temporary Disk Mounted
               |
               v
          [Amazon S3] -- Object Storage
               |
               v
      [AWS Bastion EC2] -- Temporary Disk Mounted
               |
               v
      [AWS Aurora PostgreSQL] (Private Subnet)

---

### **Scope**

- **Source:** Azure Database for PostgreSQL single instance
- **Target:** AWS Aurora PostgreSQL (Primary)
- **Data size:** ~45 GB **logical database** (from Azure monitoring metrics)
- **Method:**`pg_dump` → S3 → `pg_restore` via bastion
- **Access:** Bastion-only
- **Region:** Azure Jakarta → AWS Jakarta

---

### **Step 1 – Prepare Temporary Disk on Azure Bastion**

    lsblk
    sudo mkfs.ext4 /dev/sdc
    sudo mkdir /mnt/pgdump
    sudo mount /dev/sdc /mnt/pgdump
    

- The OS disk cannot be decreased in size, so we attach a temporary disk to safely store the database dump.
- Disk size ≥ 100 GB (safety margin for ~45 GB logical database).
- Using a separate disk avoids risk to the system and simplifies cleanup after migration.

---

### **Step 2 – Dump Database Using `pg_dump`**

    pg_dump \
      -h <azure-postgres-private-endpoint> \
      -U <db_user> \
      -d <db_name> \
      -F c \
      -Z 6 \
      --no-owner \
      --no-acl \
      -f /mnt/pgdump/appdb.dump
    

**Flags explained:**

- `-F c` → Custom format (supports parallel restore)
- `-Z 6` → Compression (balance CPU and dump size)
- `--no-owner --no-acl` → Avoid ownership/permission issues on Aurora
- Expected dump size: ~15–25 GB from a 45 GB source, as gzip compression typically reduces logical dumps; actual size may vary based on data composition.

> Tip: Temporarily revoke write access to ensure consistent snapshot.

---

### **Step 3 – Upload Dump to S3**

    aws s3 cp /mnt/pgdump/appdb.dump s3://<bucket-name>/migration/appdb.dump
    

- Use `screen` or `tmux` to prevent session termination.
- S3 acts as intermediate storage for cross-cloud transfer.

---

### **Step 4 – Prepare Temporary Disk on AWS Bastion**

    lsblk
    sudo mkfs.xfs /dev/nvme1n1
    sudo mkdir /mnt/pgdump
    sudo mount /dev/nvme1n1 /mnt/pgdump
    

- Attach ≥100 GB EBS volume.

---

### **Step 5 – Download Dump from S3**

    aws s3 cp s3://<bucket-name>/migration/appdb.dump /mnt/pgdump/
    

---

### **Step 6 – Restore to AWS Aurora PostgreSQL**

    pg_restore \
      -h <aurora-endpoint> \
      -U <db_user> \
      -d <db_name> \
      --no-owner \
      --no-acl \
      -j 8 \
      /mnt/pgdump/appdb.dump
    

**Aurora-specific notes:**

- Parallel restore (`-j 8`) speeds up ingestion.
- Ensure the target database is empty/new.
- Restore during low-traffic window.

---

### **Step 7 – Post-Restore Validation**

    ANALYZE;
    

**Checks:**

- Row counts match source
- Indexes exist
- Sequences aligned

---

### **Step 8 – Cleanup**

**Azure Bastion**

    sudo umount /mnt/pgdump
    # Delete temporary disk
    

**AWS Bastion**

    sudo umount /mnt/pgdump
    # Delete temporary EBS volume
    

---

### **Cost Considerations (Jakarta Region)**
ComponentApprox CostAzure temporary disk~$0.29/dayAWS temporary disk (EBS)~$0.32/dayS3 storage (~25–30 GB)~$0.05Azure egress (~25–30 GB)~$3.0–3.6**Total**~$4–5 USD
> Note: Cross-cloud transfer cost is minimal for a one-time migration.

---

### **Conclusion**

This PoC shows a bastion-based backup and restore approach for migrating Azure PostgreSQL (~45 GB) to AWS Aurora. Temporary disks store the dump since OS disks cannot shrink. `pg_dump` → S3 → `pg_restore` allows secure transfer over private subnets, with parallel restore (`-j`) accelerating ingestion. The method avoids VPN/tunnel setup, requires minimal downtime, and is cost-efficient for one-time migrations.
