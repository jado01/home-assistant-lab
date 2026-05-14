# SSD Migration

## Hardware used

- Raspberry Pi 5
- Raspberry Pi SSD Kit
  - Official Raspberry Pi M.2 HAT+
  - Raspberry Pi 2230 SSD 512 GB NVMe SSD
- SanDisk Extreme 64 GB microSD card

## Raspberry Pi SSD Kit Installation

1. Package
![SSD installation](./images/ssd_hat_install_01.jpg)
![SSD installation](./images/ssd_hat_install_02.jpg)
![SSD installation](./images/ssd_hat_install_03.jpg)
2. Preparing PCIe flex kabel
![SSD installation](./images/ssd_hat_install_04.jpg)
![SSD installation](./images/ssd_hat_install_05.jpg)
3. Preparing PCIe connector
![SSD installation](./images/ssd_hat_install_06.jpg)
![SSD installation](./images/ssd_hat_install_07.jpg)
4. Installed the GPIO stacking header between the Raspberry Pi 5 and SSD HAT.
![SSD installation](./images/ssd_hat_install_08.jpg)
![SSD installation](./images/ssd_hat_install_09.jpg)
5. Installed the metal standoffs between the Raspberry Pi 5 and SSD HAT.
![SSD installation](./images/ssd_hat_install_10.jpg)
![SSD installation](./images/ssd_hat_install_11.jpg)
![SSD installation](./images/ssd_hat_install_12.jpg)
6. Connected the PCIe ribbon cable between Raspberry Pi 5 and SSD HAT.
![SSD installation](./images/ssd_hat_install_13.jpg)
![SSD installation](./images/ssd_hat_install_14.jpg)
7. Mounted the SSD HAT onto the GPIO stacking header.
![SSD installation](./images/ssd_hat_install_15.jpg)
8. Mounted and secured the SSD HAT using screws and standoffs.
![SSD installation](./images/ssd_hat_install_16.jpg)
9. Done!
![SSD installation](./images/ssd_hat_install_17.jpg)

## Installation process

1. Shutdown Home Assistant
2. Installed Raspberry Pi SSD HAT
3. Connected ribbon cable
4. Mounted NVMe SSD
5. Booted Home Assistant

## Home Assistant migration

1. Opened:
   Settings → System → Storage

2. Selected:
   Move data disk

3. Selected NVMe SSD

4. Started migration process
   - approximately 10 minutes

5. Waited for reboot and migration completion

## Result

- Home Assistant successfully migrated data to NVMe SSD
- System is running correctly
- Dashboard, Zigbee2MQTT and automations are working

## Current status

- Raspberry Pi still requires SD card for boot
- Home Assistant data is running from NVMe SSD
- Full NVMe boot without SD card will be solved later

## Future plan

- Create full Home Assistant OS installation directly on NVMe SSD
- Restore backup from current system
- Remove SD card completely
