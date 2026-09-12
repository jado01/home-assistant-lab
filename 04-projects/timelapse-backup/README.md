# Timelapse Backup

Home Assistant project for reliable camera timelapse storage with a local buffer and periodic synchronization to an RPi3 server.

## Goal

The current setup stores a camera snapshot every 15 minutes directly to network storage on an RPi3 server.

The new design should be more robust and energy-efficient:

```text
Camera
  ↓
Home Assistant
  ↓
Local buffer on HA SSD
  ↓
Periodic synchronization
  ↓
RPi3 SSD
  ↓
Long-term archive
```

The main idea is that Home Assistant should always save snapshots locally first.

The RPi3 server does not need to run 24/7 and can stay powered off most of the time.

---

## Snapshot frequency

The camera creates one image every 15 minutes.

```text
4 images / hour
96 images / day
672 images / week
```

With image sizes around 100 kB, one week of snapshots is only around:

```text
~67 MB
```

This makes a weekly local buffer completely reasonable.

---

## Planned behavior

The system should support two main scenarios.

### 1. Automatic weekly synchronization

Example flow:

```text
Scheduled automation starts
        ↓
Home Assistant powers ON smart plug
        ↓
RPi3 starts booting
        ↓
HA waits until server is actually online
        ↓
Synchronization starts
        ↓
Files are verified
        ↓
RPi3 performs safe Linux shutdown
        ↓
HA waits about 3 minutes
        ↓
Smart plug is powered OFF
```

The automation should not rely only on a fixed delay.

Instead of:

```text
Power ON
Wait 60 seconds
Start sync
```

the preferred behavior is:

```text
Power ON
Wait until server is really available
Start sync
```

---

### 2. Manual server startup

If the server is powered on manually, Home Assistant should still synchronize new snapshots.

Example:

```text
User powers ON RPi3
        ↓
RPi3 boots
        ↓
HA detects server online
        ↓
New snapshots are synchronized
        ↓
Server remains ON
```

The server must not shut down automatically after synchronization because the user may be actively working on it.

---

## AUTO and MANUAL sessions

The system needs to distinguish why the server was powered on.

### AUTO

```text
Server was started by the scheduled backup automation.
```

After a successful synchronization:

```text
safe shutdown
wait
smart plug OFF
```

### MANUAL

```text
Server was started by the user.
```

After synchronization:

```text
server remains ON
```

A possible Home Assistant helper:

```text
input_boolean.server_automatic_session
```

Example meaning:

```text
ON  = automatic session
OFF = manual session
```

---

## Synchronization lock

The system needs a state that indicates when synchronization is running.

Possible HA helper:

```text
input_boolean.timelapse_sync_running
```

States:

```text
OFF = no synchronization running
ON  = synchronization is running
```

This acts as a simple lock.

Later it may be replaced or supplemented by a real Linux lock file.

---

## Safe shutdown behavior

If the user requests server shutdown while synchronization is still running, the server must not shut down immediately.

Desired logic:

```text
Shutdown requested
      ↓
Is sync running?
   ↙        ↘
 NO        YES
 ↓          ↓
Shutdown   Wait for sync
              ↓
        Sync completed
              ↓
           Shutdown
```

This should protect the transfer from interruption.

---

## Synchronization tool

For Timelapse v2, `rsync` is preferred over simple `cp` or `mv`.

Reason:

- designed for synchronization
- can transfer only new files
- safer for repeated syncs
- can preserve metadata
- can support verification
- source files can remain until transfer is confirmed

Basic principle:

```text
Do not delete source snapshots until transfer success is confirmed.
```

---

## Notifications

The system should send useful mobile notifications.

### Automatic backup success

```text
📷 Timelapse backup completed

672 images transferred.
RPi3 safely shut down.
Smart plug powered off.
```

### Manual session

```text
📷 Timelapse synchronized

192 images transferred.
Server remains ON because it was started manually.
```

### Failure

```text
⚠️ Timelapse backup failed

Synchronization was not successful.
Source images were kept.
Server remains ON for troubleshooting.
```

---

## Logging

The project should maintain a simple readable history.

Example:

```text
2026-09-12 03:00 | AUTO   | Server power ON
2026-09-12 03:01 | AUTO   | Server online
2026-09-12 03:01 | SYNC   | 672 files found
2026-09-12 03:03 | SYNC   | 672 files transferred successfully
2026-09-12 03:03 | AUTO   | Shutdown successful
2026-09-12 03:06 | AUTO   | Power OFF

2026-09-14 15:30 | MANUAL | Server power ON
2026-09-14 15:31 | SYNC   | 192 files transferred successfully
2026-09-14 15:32 | MANUAL | Server remains ON
```

The log should make it easy to determine:

- when the server was powered on
- whether the session was AUTO or MANUAL
- how many files were found
- how many files were transferred
- whether synchronization succeeded
- whether shutdown succeeded
- whether the smart plug was powered off
- whether any errors occurred

---

## Implementation plan

The project should be built step by step.

```text
1. Create local snapshot buffer
        ↓
2. Verify local snapshot automation
        ↓
3. Create manual synchronization
        ↓
4. Introduce rsync
        ↓
5. Automatic sync when RPi3 becomes available
        ↓
6. Weekly automatic server startup
        ↓
7. AUTO / MANUAL session detection
        ↓
8. Safe shutdown logic
        ↓
9. Synchronization lock
        ↓
10. Mobile notifications
        ↓
11. Logging
```

Each step should be tested independently before moving to the next one.

---

## Project goals

The purpose of this project is not only to create a working automation.

It should also be a practical learning project covering:

- Home Assistant
- Linux
- SSH
- filesystem and mount points
- Samba
- network storage
- rsync
- automation logic
- state management
- locking
- safe shutdown
- notifications
- logging
- troubleshooting

The final system should be understandable and maintainable.

I should know:

```text
what the system is doing
why it is doing it
where the data is stored
how synchronization works
how the server state is detected
how safe shutdown works
how data is protected
how to troubleshoot failures
```

---

## Related documentation

See:

```text
troubleshooting.md
```

for the real incident that led to the redesign of this project.
