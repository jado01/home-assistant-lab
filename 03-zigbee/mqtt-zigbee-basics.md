# MQTT & Zigbee Basics

## Overview

My Home Assistant smart home currently uses:

- Raspberry Pi 5
- Home Assistant OS
- Zigbee devices
- Sonoff ZBT-1 coordinator
- Zigbee2MQTT
- Mosquitto Broker

This creates a complete local IoT infrastructure.

---

# Zigbee

Zigbee is a wireless communication protocol used mainly for smart home and IoT devices.

Typical Zigbee devices:
- PIR sensors
- smart plugs
- smart bulbs
- switches
- temperature sensors
- door/window sensors

Zigbee devices communicate using Zigbee protocol, NOT MQTT.

---

# Zigbee Characteristics

- works mostly on 2.4 GHz
- low power consumption
- mesh network support
- designed for small data packets
- supports battery powered devices

Zigbee uses channels similar to WiFi.

---

# Zigbee Mesh Network

Zigbee creates a mesh network.

Powered Zigbee devices usually work as routers:
- smart plugs
- bulbs
- powered switches

Battery devices are usually end devices:
- PIR sensors
- door/window sensors
- temperature sensors

Example:

Sensor → Smart Plug → Coordinator

If one route fails, Zigbee can automatically find another route through the mesh.

---

## Home Assistant ZBT-1 Coordinator

The Home Assistant ZBT-1 is a Zigbee coordinator.

It is basically:
- USB Zigbee radio
- transmitter/receiver
- antenna for Zigbee communication

Responsibilities:
- receives Zigbee packets
- sends Zigbee packets
- connects Zigbee network to Home Assistant

The coordinator itself is only hardware.

It does NOT manage automations or devices by itself.

## Zigbee2MQTT

Zigbee2MQTT is a software addon/service running inside Home Assistant.

It communicates with:
- Home Assistant ZBT-1 coordinator

Responsibilities:
- pairing Zigbee devices
- managing Zigbee network
- receiving Zigbee packets
- translating Zigbee → MQTT
- creating MQTT messages

Without Zigbee2MQTT, Home Assistant would not understand Zigbee communication directly.

## MQTT

MQTT is NOT Zigbee.

MQTT is a lightweight messaging protocol used for:
- IoT devices
- ESP32
- Home Assistant
- automation systems
- scripts
- servers

MQTT works using:
- broker
- topics
- publishers
- subscribers

## Mosquitto Broker

Mosquitto Broker is the MQTT server.

Responsibilities:
- receives MQTT messages
- distributes MQTT messages to subscribers

Mosquitto does NOT understand Zigbee.

It only works with MQTT messages.

# MQTT Concepts

## Publisher

Device/application that SENDS MQTT messages.

Examples:
- Zigbee2MQTT
- ESP32
- Python script
- Node-RED

## Subscriber

Device/application that LISTENS to MQTT topics.

Examples:
- Home Assistant
- Node-RED
- dashboards
- scripts

## Topic

Communication channel/address.

Example:

zigbee2mqtt/toilet_pir

Topics are NOT Home Assistant entities.

They are communication paths/channels.

## Payload

The actual message/data content.

Example:

{
  "occupancy": true
}

## Retain

Stores the last MQTT message on the broker.

Useful because:

newly connected subscribers immediately receive the latest known state.

## QoS (Quality of Service)

Controls MQTT message delivery reliability.

QoS 0
- send and forget
- fastest
QoS 1
- delivery confirmation
QoS 2
- guaranteed exactly once
- slowest but most reliable

Most smart home setups use:
- QoS 0
- QoS 1

## Discovery

Automatic device discovery in Home Assistant.

Zigbee2MQTT sends discovery messages.

Home Assistant automatically creates:
- devices
- entities
- sensors

# Communication Flow
## Complete communication flow in my setup

PIR Sensor
↓ Zigbee
Sonoff ZBT-1 Coordinator
↓ USB Serial
Zigbee2MQTT
↓ MQTT
Mosquitto Broker
↓ MQTT
Home Assistant
↓
Automation
↓
Light ON

# Installed Home Assistant Addons
## Mosquitto Broker
MQTT server/broker.
## Zigbee2MQTT
Zigbee network manager and Zigbee → MQTT translator.
## File Editor
Editing Home Assistant configuration files.
## Terminal & SSH
Linux terminal and remote shell access.
## Google Drive Backup
Automatic Home Assistant backups.
## Tailscale
Secure remote access to Home Assistant.

# Important Understanding
## Zigbee
Wireless communication protocol.
## Zigbee2MQTT
Translator and Zigbee network manager.
## MQTT
Messaging/communication protocol.
## Mosquitto
MQTT broker/server.

All of these are separate layers/components of the smart home architecture.
