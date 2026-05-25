# STM32F7 Nucleo — MQTT Setup Guide

> **Publisher · Mosquitto Broker · Mobile Subscriber**

| Parameter | Value |
|---|---|
| **Target Board** | STM32F7 Nucleo (e.g. NUCLEO-F746ZG / F767ZI) |
| **IDE** | STM32CubeIDE |
| **Network Stack** | lwIP (lightweight TCP/IP) |
| **RTOS** | FreeRTOS (CMSIS-V2) |
| **MQTT Library** | lwIP built-in `mqtt.h` |
| **Broker** | Eclipse Mosquitto (Windows) |
| **Subscriber** | Any MQTT client app (e.g. MQTT Explorer on mobile) |

*Version 1.0 · 2025*

---

## Table of Contents

1. [Phase 1 — Install & Configure Mosquitto Broker on Windows](#phase-1)
2. [Phase 2 — Determine Network Topology and IP Address](#phase-2)
3. [Phase 3 — STM32CubeMX Configuration](#phase-3)
4. [Phase 4 — STM32 Firmware Implementation](#phase-4)
5. [Phase 5 — Testing and Verification](#phase-5)
6. [Phase 6 — Troubleshooting](#phase-6)

---

<a name="phase-1"></a>
## Phase 1 — Install & Configure Mosquitto Broker on Windows

---

### Step 1.1 — Download and Install Mosquitto

1. Open your browser and navigate to: <https://mosquitto.org/download/>
2. Under the **Windows** section, download the latest `.exe` installer (e.g. `mosquitto-2.x.x-install-win64.exe`).
3. Run the installer **as Administrator**. Accept all default options and complete the installation.
4. By default, Mosquitto installs to: `C:\Program Files\mosquitto\`

> 💡 **TIP:** During installation, tick **Service** if you want Mosquitto to auto-start with Windows. For development testing, running it manually in verbose mode is recommended.

---

### Step 1.2 — Create the Mosquitto Configuration File

The default `mosquitto.conf` is minimal. You must configure it to allow external (non-localhost) connections.

1. Navigate to the installation folder: `C:\Program Files\mosquitto\`
2. Open `mosquitto.conf` in Notepad (**run Notepad as Administrator** if needed).
3. Replace or append the following configuration:

```conf
# mosquitto.conf — STM32F7 development configuration

# Allow connections from all network interfaces
listener 1883

# Disable authentication for local development
# (use password_file in production!)
allow_anonymous true

# Log all events to console (verbose mode covers this)
log_type all
```

> ⚠️ **WARNING:** `allow_anonymous true` is safe for closed home networks only. In production, always configure a `password_file` and use TLS on port `8883`.

---

### Step 1.3 — Configure Windows Defender Firewall

Windows blocks incoming connections by default. You must open port **1883** to allow the STM32 to connect.

1. Press `Win + S`, type **Windows Defender Firewall with Advanced Security**, and press Enter.
2. Click **Inbound Rules** in the left panel.
3. Click **New Rule...** in the right-hand Actions panel.
4. Select **Port** → click Next.
5. Select **TCP**. In the *Specific local ports* field, type: `1883` → click Next.
6. Select **Allow the connection** → click Next.
7. Keep all three profiles checked (**Domain**, **Private**, **Public**) → click Next.
8. Name the rule: `Mosquitto MQTT` → click **Finish**.

> 💡 **TIP:** To verify the rule was created, scroll the Inbound Rules list and find **Mosquitto MQTT**. It should show a green tick (Enabled).

---

### Step 1.4 — Start the Broker

Open Command Prompt (`cmd.exe`) and run the broker in verbose mode so you can monitor connections in real time:

```cmd
cd "C:\Program Files\mosquitto"
mosquitto -c mosquitto.conf -v
```

You should see output similar to:

```
1700000000: mosquitto version 2.x.x starting
1700000000: Config loaded from mosquitto.conf.
1700000000: Opening ipv4 listen socket on port 1883.
1700000000: Opening ipv6 listen socket on port 1883.
1700000000: mosquitto version 2.x.x running
```

Leave this Command Prompt window **open**. Every device connection and MQTT message will appear here.

---

<a name="phase-2"></a>
## Phase 2 — Determine Network Topology and IP Address

The STM32 and your PC must be on the **same network**. There are two supported topologies:

---

### Step 2.1 — Find Your PC's IP Address

Open a new Command Prompt and run:

```cmd
ipconfig
```

Look for your active adapter. The **IPv4 Address** is what you will enter in the STM32 firmware later.

---

### Scenario A — Home Router (DHCP) — Recommended

Both the PC and the Nucleo board are connected to the same home router (either by Ethernet cable or Wi-Fi for the PC).

| Parameter | Value |
|---|---|
| **Broker IP** | Your PC's Wi-Fi or Ethernet IPv4 (e.g. `192.168.2.171`) |
| **STM32 IP Strategy** | Enable DHCP — the router assigns an IP automatically |
| **Physical connection** | Nucleo's RJ45 port → router via Ethernet cable |

---

### Scenario B — Direct Cable Connection (Static IP)

The Nucleo board is plugged directly into the PC's Ethernet port with an Ethernet cable — no router involved.

| Parameter | Value |
|---|---|
| **Broker IP** | Your PC's physical Ethernet adapter IPv4 (e.g. `192.168.56.1`) |
| **STM32 IP Strategy** | Disable DHCP. Use Static IP (e.g. `192.168.56.2`) |
| **STM32 Netmask** | `255.255.255.0` |
| **STM32 Gateway** | `192.168.56.1` (your PC) |
| **Physical connection** | Nucleo RJ45 → PC Ethernet port directly |

> ⚠️ **WARNING:** For Scenario B, your PC's Ethernet adapter must also be configured with a static IP. Go to **Control Panel → Network Adapters → Ethernet → IPv4 Properties** and set it to `192.168.56.1` / `255.255.255.0`.

---

<a name="phase-3"></a>
## Phase 3 — STM32CubeMX Configuration

Create a new project for your specific Nucleo board in STM32CubeIDE. The following settings are required in the CubeMX perspective before generating code.

---

### Step 3.1 — Enable Ethernet (ETH) Peripheral

1. In the **Pinout & Configuration** view, go to **Connectivity → ETH**.
2. Set **Mode** to `RMII` (this is the hardware default for all STM32F7 Nucleo boards).
3. Leave all other ETH parameters at their defaults.

---

### Step 3.2 — Enable FreeRTOS

1. Go to **Middleware → FREERTOS**.
2. Set the Interface to **CMSIS_V2** (recommended — provides `osDelay()` and task APIs used in the firmware).
3. Under **Config parameters**, increase `configTOTAL_HEAP_SIZE` to at least **32768** (32 KB). lwIP and MQTT require significant heap for buffers.

> 💡 **TIP:** If you see a hard fault or the board resets during connection, increase `configTOTAL_HEAP_SIZE` to `49152` (48 KB) first — memory exhaustion is the most common cause.

---

### Step 3.3 — Enable lwIP

1. Go to **Middleware → LWIP** and enable it.
2. Under **General Settings**, configure IP addressing based on your Phase 2 scenario:

| Scenario | lwIP Setting |
|---|---|
| Scenario A (Router / DHCP) | Enable DHCP checkbox = **checked** |
| Scenario B (Direct cable) | Enable DHCP = **unchecked**. Set IP Address, Netmask, Gateway manually. |

3. Under **Key Options / Advanced Parameters**, ensure **MQTT** is enabled so `mqtt.h` is included in the generated code. Tick `LWIP_ALTCP` if using secure MQTT.

---

### Step 3.4 — Generate Code

1. Click **Project → Generate Code** (or press `Alt + K`).
2. Open the generated project in STM32CubeIDE.

---

<a name="phase-4"></a>
## Phase 4 — STM32 Firmware Implementation

All code additions go inside the `USER CODE BEGIN` / `USER CODE END` comment guards so they survive CubeMX regeneration. Open `main.c` (or a dedicated network task `.c` file) in STM32CubeIDE.

---

### Step 4.1 — Add Includes and Global Variables

At the top of `main.c`, inside the `USER CODE BEGIN Includes` block:

```c
/* USER CODE BEGIN Includes */
#include "lwip/apps/mqtt.h"
#include "lwip/ip_addr.h"
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN PV */
mqtt_client_t *mqtt_client;
ip_addr_t broker_ip;
/* USER CODE END PV */
```

---

### Step 4.2 — Add the Connection Callback and Connect Function

Inside the `USER CODE BEGIN 4` section (below the task definitions):

```c
/* USER CODE BEGIN 4 */

/**
 * Called by lwIP when the MQTT connection attempt completes.
 * status == MQTT_CONNECT_ACCEPTED means the broker accepted us.
 */
static void mqtt_connection_cb(mqtt_client_t *client,
                               void *arg,
                               mqtt_connection_status_t status)
{
    if (status == MQTT_CONNECT_ACCEPTED) {
        printf("Connected to Mosquitto broker!\n");

        /* ── Publish a test message ── */
        const char *payload = "Hello from Nucleo F7!";
        err_t err = mqtt_publish(client,
                                 "stm32/status",   /* topic */
                                 payload,
                                 strlen(payload),
                                 0,                /* QoS 0 */
                                 0,                /* retain = 0 */
                                 NULL, NULL);
        if (err != ERR_OK) {
            printf("Publish failed: %d\n", err);
        }
    } else {
        printf("MQTT disconnected, reason: %d\n", status);
    }
}

/**
 * Creates an MQTT client and initiates a connection to the broker.
 * Call this ONCE after lwIP is fully initialised.
 */
void connect_to_mqtt(void)
{
    mqtt_client = mqtt_client_new();

    /* REPLACE with your PC's IP address (from Phase 2) */
    IP4_ADDR(&broker_ip, 192, 168, 2, 171);

    struct mqtt_connect_client_info_t ci;
    memset(&ci, 0, sizeof(ci));
    ci.client_id  = "Nucleo_F7_Client";
    ci.keep_alive = 60;   /* seconds */

    err_t err = mqtt_client_connect(mqtt_client,
                                    &broker_ip,
                                    1883,              /* port */
                                    mqtt_connection_cb,
                                    0,                 /* callback arg */
                                    &ci);
    if (err != ERR_OK) {
        printf("MQTT connect init failed: %d\n", err);
    }
}

/* USER CODE END 4 */
```

> ⚠️ **WARNING:** Replace `192, 168, 2, 171` in `IP4_ADDR()` with your PC's actual IP address found in Phase 2. A wrong IP is the **#1 cause of connection failure**.

---

### Step 4.3 — Call the Connect Function from the Default Task

Inside `StartDefaultTask()`, within `USER CODE BEGIN 5`:

```c
/* USER CODE BEGIN 5 */
void StartDefaultTask(void *argument)
{
    /* Wait for lwIP stack to initialise and (if DHCP) obtain an IP.
       5 seconds is a safe margin. Reduce to 2 s if using static IP. */
    osDelay(5000);

    connect_to_mqtt();

    /* Infinite loop — add your sensor-read-and-publish logic here */
    for (;;)
    {
        osDelay(1000);
    }
}
/* USER CODE END 5 */
```

> 💡 **TIP:** For periodic temperature publishing, add your sensor read call and `mqtt_publish()` inside the `for(;;)` loop. Use `osDelay(5000)` to publish every 5 seconds.

---

<a name="phase-5"></a>
## Phase 5 — Testing and Verification

---

### Step 5.1 — Ensure the Broker is Running

The Mosquitto Command Prompt from Phase 1.4 must still be running. If you closed it, restart it:

```cmd
cd "C:\Program Files\mosquitto"
mosquitto -c mosquitto.conf -v
```

---

### Step 5.2 — Open a Subscriber on Your PC

Open a **second** Command Prompt and subscribe to the test topic:

```cmd
mosquitto_sub -h localhost -t "stm32/status" -d
```

Leave this window open. Any message published by the STM32 on the topic `stm32/status` will appear here.

---

### Step 5.3 — Subscribe on Your Mobile Phone

Install an MQTT client app on your phone. Recommended options:

| App | Platform |
|---|---|
| MQTT Explorer | Android / iOS |
| IoT MQTT Panel | Android |
| MQTTool | iOS |
| MyMQTT | Android |

1. Open the app and create a **new connection**.
2. Set **Host / Broker** to your PC's IP address (e.g. `192.168.2.171`).
3. Set **Port** to `1883`.
4. Set **Client ID** to anything (e.g. `phone_subscriber`).
5. Tap **Connect**.
6. Subscribe to the topic: `stm32/status`

---

### Step 5.4 — Flash and Reset the STM32

1. In STM32CubeIDE, click **Run → Debug** (or press `F11`) to compile and flash the firmware.
2. Once flashing is complete, press the **black Reset button** on the Nucleo board.
3. Wait approximately **5–6 seconds** for lwIP to initialise (the `osDelay(5000)` in `StartDefaultTask`).

---

### Step 5.5 — Verify All Three Endpoints

Check each of the following in order:

| What to check | Expected output |
|---|---|
| **Mosquitto broker window** | `New client connected from <Nucleo_IP> as Nucleo_F7_Client` |
| **PC subscriber window** | `Hello from Nucleo F7!` |
| **Mobile phone app** | Message appears on topic `stm32/status`: `Hello from Nucleo F7!` |
| **STM32 serial output** | `Connected to Mosquitto broker!` (if SWO/UART `printf` is configured) |

---

<a name="phase-6"></a>
## Phase 6 — Troubleshooting

| Problem | Likely Cause & Fix |
|---|---|
| **STM32 resets / hard fault** | `configTOTAL_HEAP_SIZE` too small. Increase to 48 KB in `FreeRTOSConfig.h`. |
| **Broker shows no connection** | Wrong IP in `IP4_ADDR()`. Verify with `ipconfig` and update the firmware. |
| **Broker connects but no message** | `mqtt_publish()` called before connection is established. Publish inside the callback instead. |
| **Firewall blocking connection** | Verify the Inbound Rule for port `1883` exists and is enabled. Temporarily disable Windows Defender to test. |
| **Mobile cannot connect** | Phone and PC must be on the same Wi-Fi network. Verify the IP is reachable (some apps support `ping`). |
| **DHCP never gets IP** | Increase `osDelay()` to `8000` ms. Some routers are slow to assign addresses. |
| **`mqtt_client_new()` returns NULL** | Out of heap memory. Increase `configTOTAL_HEAP_SIZE`. |

---

*STM32F7 MQTT Setup Guide · Version 1.0 · 2025*
