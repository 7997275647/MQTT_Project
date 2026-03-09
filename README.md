# NodeStream: Real-Time STM32 MQTT Telemetry

## Overview
NodeStream is a local, edge-based Internet of Things (IoT) telemetry pipeline. It demonstrates the implementation of a lightweight publish/subscribe (Pub/Sub) messaging architecture using an STM32F7 microcontroller. The system captures localized data and publishes it over an Ethernet network via the LwIP (Lightweight IP) stack to a locally hosted MQTT broker, allowing real-time monitoring from a mobile client.

This project highlights low-level embedded network configuration, asynchronous C programming, and decentralized IoT system design.

## System Architecture

![System Architecture Diagram](docs/architecture_diagram.png)

The system consists of three main nodes:
1. **The Publisher (Edge Node):** An STM32F7 development board connected to the local network via Ethernet. It utilizes the LwIP network stack and a C-based MQTT client to publish simulated telemetry data (e.g., sensor readings) to a specific topic.
2. **The Broker (Middleware):** A local machine (laptop) running Eclipse Mosquitto. It handles the routing of messages without requiring an external internet connection, ensuring low latency and high security.
3. **The Subscriber (Client):** A mobile smartphone running an MQTT dashboard application, subscribed to the STM32's topic to display real-time data updates.

## Hardware Requirements
* **Microcontroller:** STM32F7xx series development board (e.g., Nucleo-F767ZI) with an integrated Ethernet MAC/PHY.
* **Networking:** Standard Ethernet cable and a local Wi-Fi router (with DHCP enabled).
* **Broker Host:** Windows, macOS, or Linux machine.
* **Client:** Android or iOS smartphone.

## Software & Tools
* **Development Environment:** STM32CubeIDE
* **Network Stack:** LwIP (Lightweight IP) configured via STM32CubeMX
* **MQTT Broker:** Eclipse Mosquitto (v2.0+)
* **Mobile Client:** MQTT Dash (Android) / EasyMQTT (iOS)
* **Languages:** C

## Setup Instructions

### 1. Broker Setup (Laptop)
1. Install [Eclipse Mosquitto](https://mosquitto.org/download/).
2. Locate the `mosquitto.conf` file in the installation directory.
3. Allow external network connections by adding the following lines to the top of the configuration file:
   ```text
   listener 1883
   allow_anonymous true