# Timelapse Backup - Troubleshooting

## Incident: Home Assistant network storage failure

**Date:** 2026-09-12

Home Assistant creates a balcony camera snapshot every 15 minutes.

Originally, the snapshots were written directly to an SSD connected to an RPi3 server through Samba network storage.

Home Assistant network storage configuration:

```text
Name: linux_server_ssd
Server: 192.168.1.149
Share: storage
```

RPi3 storage:

```text
/dev/sda1
```

mounted at:

```text
/media/storage
```

The camera images are stored under:

```text
timelapse/balcony/
```

The RPi3 server can be accessed using:

```bash
ssh server-home
```

---

## Problem

The RPi3 had previously been safely shut down from Home Assistant.

After shutdown, the smart plug powering the server was switched off.

The server was later powered on again, but the Home Assistant network storage mount was not checked afterward.

Later, Home Assistant reported:

```text
Network storage device failed
```

and:

```text
Could not set up linux_server_ssd
```

At the same time, new camera snapshots were no longer appearing on the RPi3 SSD.

---

## Troubleshooting approach

The system was checked layer by layer:

```text
SSD
 ↓
Linux mount
 ↓
Samba service
 ↓
Samba share
 ↓
network
 ↓
Home Assistant network mount
 ↓
automation
 ↓
camera
```

The goal was to determine exactly where the failure occurred instead of randomly restarting devices and services.

---

## 1. SSH connection to RPi3

Command:

```bash
ssh server-home
```

SSH connection worked correctly.

---

## 2. Check storage devices

Command:

```bash
lsblk
```

Relevant result:

```text
sda
└─sda1  931.5G  /media/storage
```

Conclusion:

- RPi3 sees the SSD
- partition exists
- filesystem is mounted
- mount point is `/media/storage`

The SSD was not the cause of the problem.

---

## 3. Check Samba service

Command:

```bash
systemctl status smbd
```

Result:

```text
active (running)
```

Samba was running correctly.

Important note:

A running Samba service does not automatically mean that a specific share is configured correctly.

---

## 4. Check Samba configuration

Command:

```bash
testparm -s
```

Relevant configuration:

```text
[storage]
    path = /media/storage
    read only = No
    valid users = jan
```

The Samba share:

```text
storage
```

correctly points to:

```text
/media/storage
```

---

## 5. Test Samba from Windows

The share was opened from Windows Explorer:

```text
\\192.168.1.149\storage
```

Existing files were visible and accessible.

At this point the following components were confirmed working:

```text
RPi3              OK
SSD               OK
Linux mount       OK
Samba             OK
Samba share       OK
LAN               OK
```

This narrowed the problem down to Home Assistant.

---

## 6. Home Assistant network storage reload

In Home Assistant:

```text
Settings → System → Storage
```

the storage entry existed:

```text
linux_server_ssd
192.168.1.149:storage
```

After using Reload, Home Assistant reported:

```text
Cannot mount linux_server_ssd because there is existing data at
/data/media/linux_server_ssd.
Move it away first, then retry.
```

This revealed the actual problem.

---

## Root cause

While the RPi3 was powered off, the Samba network share was unavailable.

The camera automation continued creating snapshots every 15 minutes.

Instead of failing completely, Home Assistant started writing the images locally into the mount point.

Likely sequence:

```text
RPi3 OFF
   ↓
network storage unavailable
   ↓
camera automation continues
   ↓
snapshots are written locally
   ↓
local files appear in mount point
   ↓
RPi3 is powered on later
   ↓
HA tries to mount Samba share again
   ↓
mount point already contains local data
   ↓
HA refuses to mount over existing files
```

Home Assistant was protecting the local files from being hidden by the network mount.

---

## 7. Home Assistant Terminal and container paths

Home Assistant Terminal & SSH add-on prompt:

```text
[core-ssh ~]$
```

Attempting:

```bash
ls -lah /data/media/linux_server_ssd
```

returned:

```text
No such file or directory
```

This did not mean the Supervisor path did not exist.

The Terminal add-on runs in its own container/environment and does not see all host filesystem paths in the same way.

The path visible from the Terminal add-on was:

```bash
ls -lah /media
```

where:

```text
linux_server_ssd
```

was visible.

Important lesson:

```text
Different Home Assistant OS components can see the filesystem differently.
```

---

## 8. Recover blocking local data

Home Assistant Repair offered:

```text
Move blocking local data away
```

Home Assistant confirmed that no files would be deleted.

The local data was moved to:

```text
linux_server_ssd_local_recovery
```

After recovery, the following path existed:

```text
/media/linux_server_ssd_local_recovery/
└── timelapse/
    └── balcony/
        └── snapshots
```

At the same time:

```text
/media/linux_server_ssd
```

became the real Samba network mount again.

---

## 9. Verify timeline continuity

Last snapshots on RPi3 before the failure:

```text
2026-07-29 16:15
2026-07-29 16:30
2026-07-29 16:45
```

First recovery snapshot:

```text
2026-07-29 17:00
```

Last recovery snapshot:

```text
2026-09-12 12:45
```

First new snapshot written to RPi3 after repair:

```text
2026-09-12 13:00
```

Timeline:

```text
RPi3                         RECOVERY                         RPi3

16:15 → 16:30 → 16:45 → 17:00 → ... → 12:30 → 12:45 → 13:00
```

The 15-minute interval was continuous.

Conclusion:

The camera and automation had been working the whole time.

Only the destination storage changed.

---

## 10. Count files

General command:

```bash
find /path/to/folder -type f | wc -l
```

Recovery:

```bash
find /media/linux_server_ssd_local_recovery/timelapse/balcony -type f | wc -l
```

Result:

```text
4303
```

Before recovery copy, the RPi3 contained approximately:

```text
1029
```

files.

---

## 11. Show newest files

Command:

```bash
ls -lt /path | head -5
```

Meaning:

```text
-l = detailed listing
-t = sort by modification time
head -5 = show first 5 lines
```

---

## 12. Show oldest files

Command:

```bash
ls -ltr /path | head -5
```

`-r` reverses the sorting order.

---

## 13. Check directory size

Command:

```bash
du -sh /path
```

Recovery size:

```text
360M
```

Full timelapse directory on RPi3 after recovery:

```text
462M
```

---

## 14. Copy recovery files to RPi3

The recovery files were copied instead of moved.

Reason:

The recovery directory should remain intact until the transfer is verified.

Command:

```bash
cp -av /media/linux_server_ssd_local_recovery/timelapse/balcony/. /media/linux_server_ssd/timelapse/balcony/
```

Meaning:

```text
cp = copy
-a  = archive mode, preserve metadata
-v  = verbose output
```

The `/.` at the end means:

```text
copy the contents of this directory
```

---

## 15. Shell line continuation

A backslash at the end of a shell line:

```bash
command \
next-part
```

means:

```text
the command continues on the next line
```

It is not part of the path.

---

## 16. File counts after copy

After copying:

```text
Recovery: 4303
RPi3:     5343
```

The RPi3 count was larger because it contained:

- old snapshots
- 4303 recovered snapshots
- new snapshots created after repair

---

## 17. BusyBox find limitation

The Home Assistant Terminal uses BusyBox utilities.

The following option was not supported:

```text
find -printf
```

Error:

```text
find: unrecognized: -printf
```

Important lesson:

Not all Linux command implementations support the same options.

---

## 18. Compare recovery files with server files

Create a list of recovery filenames:

```bash
find /media/linux_server_ssd_local_recovery/timelapse/balcony -type f | sed 's#.*/##' | sort > /tmp/recovery.txt
```

Create a list of server filenames:

```bash
find /media/linux_server_ssd/timelapse/balcony -type f | sed 's#.*/##' | sort > /tmp/server.txt
```

Compare them:

```bash
comm -23 /tmp/recovery.txt /tmp/server.txt
```

The command produced no output.

Meaning:

```text
Every recovery filename also exists on the RPi3 server.
```

This confirmed that all 4303 recovery files had been transferred successfully.

---

## 19. Remove recovery directory

Before deletion, verify the path:

```bash
ls -ld /media/linux_server_ssd_local_recovery
```

Then remove it:

```bash
rm -rf /media/linux_server_ssd_local_recovery
```

Meaning:

```text
rm = remove
-r = recursive
-f = force
```

Important:

```text
Always verify the full path before using rm -rf.
```

This command does not move files to a recycle bin and normally does not ask for confirmation.

---

## 20. Verify cleanup

Command:

```bash
ls -lah /media
```

The recovery directory was gone and only the active network storage remained:

```text
linux_server_ssd
```

---

## Final state

After repair:

```text
RPi3                 OK
SSD                  OK
Samba                OK
Network storage HA   OK
Automation           OK
New snapshots        OK
4303 recovery files  transferred
Recovery             removed
```

---

## Main lesson

The most useful troubleshooting approach was to check the system layer by layer:

```text
hardware
 ↓
filesystem
 ↓
service
 ↓
network
 ↓
mount
 ↓
application
 ↓
automation
```

Instead of restarting everything, verify each layer and identify the first one that fails.

This incident directly led to the redesign described in:

```text
README.md
```

where Timelapse v2 uses a local Home Assistant buffer and periodic synchronization instead of continuous direct writing to network storage.
