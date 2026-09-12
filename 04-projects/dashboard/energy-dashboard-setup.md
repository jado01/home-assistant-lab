# Energy Dashboard Setup

## Purpose

Using smart plugs to monitor:
- real-time power usage
- long-term energy consumption

Examples:
- air purifier
- vacuum charger

---

# Important Difference

## Power
- measured in W
- current live consumption

## Energy
- measured in kWh
- long-term consumption over time

Energy Dashboard mainly uses kWh entities.

---

# Smart Plug Entities

Typical entities:
- Power
- Energy
- Voltage
- Current
- Switch state

---

# Energy Dashboard Setup

Path:
Settings → Dashboards → Energy

Added:
- Device energy consumption → kWh entity
- Device power consumption → W entity

---

# Notes

- Energy Dashboard needs time to collect statistics.
- Graphs become useful after several hours/days.
- Power = live usage
- Energy = accumulated consumption
