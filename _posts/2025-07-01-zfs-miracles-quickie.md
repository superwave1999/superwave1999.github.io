---
title: "ZFS Miracles & HDD Migration"
date: 2025-07-01
tags:
  - zfs
  - data-recovery
  - smartctl
  - backups
  - tar
  - linux
  - system-recovery
categories: [documentation]
---

# The state of my data

With an up-and-running system, I want to begin bringing my data into the equation. I have 2 options:

1. Re-create my ZFS pools and restore from my S3 backup
2. Keep everything as-is since I have two additional files on the HDDs that weren't backed up

Since I wanted to test the magic of ZFS, I inserted the HDDs, powered-on the system and ran some commands.

## Importing my pool

To see the available pools that can be imported:

```bash
zfs import
```

I saw that my pool was detected so I imported it:

```bash
zfs import datapool
```

Not only was it effortless, but the pool was automatically imported to the same folder (`/storage`) keeping all the settings, and even remembered that it had a `zpool scrub` midway on my original machine. For me, the job was done.

## Additional help for you

In your case, dear reader, YMMV. If you have doubts about your drive's integrity, or the origin system wasn't powered off properly, or whatever your situation, I'll supply some additional zfs import commands that may help you. These following commands are considered "risky" and may not guarantee a successful import or may cause data loss.

The below command force imports a pool. I'd use this if you get warnings that the pool can't be imported, but know the drives are physically OK.

```bash
zfs import -f mypool
```

The below command imports a pool as readonly to inspect the state of your data and avoiding any writes to it. It's a good way to see how it behaves before committing to the pool migration.

```bash
zpool import -f -o readonly=on mypool
```

If you want to simulate an import, but you don't want to actually mount the pool for whatever reason, use the -n option:

```bash
zpool import -n -o readonly=on -f mypool
```

For additional force when importing you can run the following:

```bash
zpool import -f -m mypool
```

For extreme forcefulness, run the following. At this point, don't have high hopes about your data being intact:

```bash
zpool import -f -X mypool
```

## Post-import status check

After the import, I ran a status check:

```bash
zpool status datapool
```

The output was very interesting:

```text
  pool: datapool
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-9P
  scan: resilvered 23.9G in 00:05:23 with 0 errors on Mon Jun 30 22:25:29 2025
config:

        NAME                                   STATE     READ WRITE CKSUM
        datapool                               ONLINE       0     0     0
          raidz1-0                             ONLINE       0     0     0
            ata-WDC_WD80EFBX-68AZZN0_VRG1N21K  ONLINE       0     0     0
            ata-WDC_WD80EFBX-68AZZN0_VRG13ETK  ONLINE       0     0     0
            ata-WDC_WD80EFBX-68AZZN0_VRG0P6GK  ONLINE       0     0     1
            ata-WDC_WD80EFBX-68AZZN0_VRG10ZLK  ONLINE       0     0     0
```

We can see that the third drive was offlined' in my original system and that it took 05:23 to resilver (restore parity) the pool. In the CKSUM column, it marks which drives have had data parity errors. We will now follow the recommended action by ZFS and check if the drive is OK before clearing the error.

We will use the `smartmontools` package (apt install if unavailable) to test the drives. In my case, I ran a long test (overnight) with the following command. For a shorter test, feel free replace "long" with "short".

```bash
smartctl -t long /dev/disk/by-id/ata-WDC_WD80EFBX-68AZZN0_VRG0P6GK
```

After some time, check the status of the test:

```bash
smartctl -a /dev/disk/by-id/ata-WDC_WD80EFBX-68AZZN0_VRG0P6GK
```

To know if a test is running and it's progress, look for something akin to the following:

```text
Self-test execution status:      ( 249) Self-test routine in progress...
                                        90% of test remaining.
```

Once complete, check some of the key metrics:

```text
Reallocated_Sector_Ct 0
Current_Pending_Sector 0
Offline_Uncorrectable 0
UDMA_CRC_Error_Count 0
SMART Status   PASSED
Self-Test   Completed without error
```

Finally, clear the error and run a scrub again to be 100% safe.

```bash
zpool clear datapool
zpool scrub datapool
```

# What about the rest?

To transfer the data that was on the OS SSD (databases, thumbnails, scripts, composes...) I used trusty tar.

We create a compressed archive of the directories to transfer from inside the source machine:

```bash
sudo tar -czpf files.tar.gz /home/imanol/bin /storage-core

exit
```

In the intermediate Windows machine, we will now pull the archive.

```bash
scp imanol@myserver.lan:/home/imanol/files.tar.gz .
```

With the old system fully decommissioned and the new one up and running, we push, extract and delete the archive.

```bash
scp files.tar.gz imanol@myserver.lan:/home/imanol/
ssh imanol@myserver.lan 'sudo tar -xzpf /home/imanol/files.tar.gz -C /'
ssh imanol@myserver.lan 'rm /home/imanol/files.tar.gz'
```

# Next steps

Since our data is in place, we can now start our docker containers and have everything up-and-running as it was originally. This I'll do in a future post.
