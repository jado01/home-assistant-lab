## WC Adaptive Motion Lighting

Created a smart WC lighting automation using two MOES Zigbee GU10 bulbs and a PIR motion sensor.

### Current setup

- 2x MOES Zigbee GU10 smart bulbs
- 1x PIR motion sensor
- Motion-based light activation
- Automatic light turn-off after no motion
- Brightness and color temperature change based on time of day

### Lighting modes

| Mode | Time | Brightness | Color temperature |
|---|---|---:|---|
| Night | 22:00 - 06:00 | 5% | Warm white |
| Evening | 18:00 - 22:00 | 25% | Warm white |
| Day | 06:00 - 18:00 | 40% | Neutral white |

### Automation logic

- When motion is detected, lights turn on with settings based on the current time.
- When no motion is detected for 30 seconds, lights turn off.
- The automation uses a Choose block with Night, Evening and Day/default branches.

### Notes

- Brightness levels are initial values and may be adjusted after real-world testing.
- PIR sensor is a temporary solution.
- Future improvement: replace PIR sensor with a presence sensor for more reliable detection.
