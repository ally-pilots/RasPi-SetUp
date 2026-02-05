# Raspberry Pi IoT Gateway Setup Guide  
*CAMPUS 02 Token – MQTT – Node-RED – Grafana – Azure Cloud*

The Raspberry Pi provides:

- MQTT communication (Mosquitto broker)
- Data processing (Node-RED)
- Real-time visualization (Grafana)
- Optional cloud extension (Microsoft Azure IoT Hub + Cosmos DB)

---

## System Overview

The CAMPUS 02 Token includes the following sensors:

- Barometer (pressure)
- Temperature sensor
- Gyroscope
- Accelerometer

Sensor measurements are transmitted via MQTT to the Raspberry Pi, where they are processed and visualized locally.  
Optionally, data can be forwarded to Microsoft Azure for cloud storage and analysis.

Architecture:

CAMPUS 02 Token → MQTT → Raspberry Pi Gateway → Node-RED → Grafana dashboards  
                                                     ↘ Azure IoT Hub → Cosmos DB

---

## Step 1: Flash Raspberry Pi OS onto the SD Card

Download Raspberry Pi Imager:  
https://www.raspberrypi.com/software/

Flash Raspberry Pi OS onto the microSD card.

---

## Step 2: Initial Configuration

After first boot:

1. Set hostname (device name)
2. Select timezone and language
3. Create username and password
4. Enable SSH
5. Configure Wi-Fi or Ethernet

---

### Headless Connection via SSH

Find the Raspberry Pi IP address from Windows:

```powershell
arp -a
```

Connect via SSH:

```powershell
ssh username@ip_address
```

---

## Step 3: Update the System

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Step 4: Install Required Tools

```bash
sudo apt install -y curl wget git
```

---

## Step 5: Install Mosquitto (MQTT Broker)

Install Mosquitto and MQTT client tools:

```bash
sudo apt install -y mosquitto mosquitto-clients
```

Enable the service:

```bash
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

Check status:

```bash
sudo systemctl status mosquitto
```

---

### MQTT Topic Structure

Sensor data can be published under topics such as:

- `sensor/temperature`
- `sensor/pressure`
- `sensor/gyroscope`
- `sensor/accelerometer`

---

### MQTT Test

Terminal 1 (subscribe):

```bash
mosquitto_sub -t test/topic
```

Terminal 2 (publish):

```bash
mosquitto_pub -t test/topic -m "hello raspberry pi"
```

---

## Step 6: Install Node-RED

Install Node-RED:

```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```

Enable service:

```bash
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
```

Access Node-RED:

```
http://<pi-ip-address>:1880
```

Node-RED is used to subscribe to MQTT sensor topics, parse values, and forward them to visualization or cloud services.

---

## Step 7: Install Grafana

Install Grafana:

```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
```

Enable Grafana:

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access Grafana:

```
http://<pi-ip-address>:3000
```

Default login:

- admin / admin

Grafana dashboards visualize:

- Temperature trends
- Pressure values
- Motion sensor data (gyro + accelerometer)

---

## Step 8: Service Ports Summary

| Service     | Port  | Purpose |
|------------|------|---------|
| Mosquitto   | 1883 | MQTT broker |
| Node-RED    | 1880 | IoT flow processing |
| Grafana     | 3000 | Dashboards |

---

## Step 9: Cloud Extension with Microsoft Azure

### Azure Student Account

Create a free Azure student account:

https://azure.microsoft.com/free/students/

---

### Azure IoT Hub Setup

1. Open Azure Portal  
2. Create an **IoT Hub**
3. Register a device (e.g., `raspi-gateway`)
4. Copy the device connection string

---

### Sending Telemetry from Raspberry Pi

Python packages must be installed in a virtual environment:

```bash
sudo apt install -y python3-venv
python3 -m venv azure-env
source azure-env/bin/activate
pip install azure-iot-device
```

Example telemetry script (`send_to_azure.py`) forwards sensor values as JSON messages:

```python
from azure.iot.device import IoTHubDeviceClient, Message
import json, time

CONNECTION_STRING = "YOUR_DEVICE_CONNECTION_STRING"
client = IoTHubDeviceClient.create_from_connection_string(CONNECTION_STRING)

while True:
    data = {"temperature": 22.5, "pressure": 1013.2}
    client.send_message(Message(json.dumps(data)))
    time.sleep(5)
```

---

## Step 10: Azure Cloud Database (Cosmos DB)

Azure Cosmos DB provides cloud storage for telemetry data.

1. Create Cosmos DB Account  
2. Select **Azure Cosmos DB for NoSQL**
3. Choose an allowed region (Azure Student subscriptions restrict locations)
4. Create database: `iotDatabase`
5. Create container: `sensorData`

IoT Hub telemetry can be routed into Cosmos DB using Message Routing.

---
## Reference Sensor Data Format

The MQTT payload format depends, so here I use one just as an example.

---

### MQTT Topic

`sensor/campus02-token`

---

### Example Payload

```json
{
  "deviceId": "campus02-token",
  "timestamp": "2026-02-05T12:00:00Z",
  "temperature": 22.5,
  "pressure": 1013.2,
  "gyroscope": {
    "x": 0.01,
    "y": 0.02,
    "z":  0.00
  },
  "accelerometer": {
    "x": 0.10,
    "y": 0.00,
    "z": 9.81
  }
}
```

This format is compatible with Node-RED processing, Grafana dashboards, and Azure IoT Hub telemetry storage.

---

## MQTT Sensor Data Simulator (Testing Without Hardware)

To test the full MQTT → Node-RED → Grafana/Azure pipeline without requiring the physical token,
a simple Python script can be used to simulate sensor measurements.

### Install required package

```bash
pip install paho-mqtt
```

---

### Example Simulator Script (`mqtt_simulator.py`)

```python
import paho.mqtt.client as mqtt
import json, time, random
from datetime import datetime

# Connect to local Mosquitto broker
client = mqtt.Client()
client.connect("localhost", 1883)

print("Publishing simulated sensor data...")

while True:
    payload = {
        "deviceId": "campus02-token",
        "timestamp": datetime.utcnow().isoformat(),
        "temperature": round(20 + random.random() * 5, 2),
        "pressure": round(1010 + random.random() * 5, 2),
        "gyroscope": {"x": 0.01, "y": 0.02, "z": 0.00},
        "accelerometer": {"x": 0.10, "y": 0.00, "z": 9.81}
    }

    # Publish JSON payload to MQTT topic
    client.publish("sensor/campus02-token", json.dumps(payload))

    print("Published:", payload)

    time.sleep(5)
```

---

### Run the simulator

```bash
python3 mqtt_simulator.py
```

This will publish a new JSON telemetry message every 5 seconds, which can then be consumed by Node-RED, Grafana, or forwarded to Azure IoT Hub.

---

## Forwarding MQTT Sensor Data to Azure IoT Hub

The simulator publishes JSON telemetry messages to the local MQTT broker under:

`sensor/campus02-token`

To send these MQTT messages to Azure IoT Hub, a small gateway script can be used.

This enables the full pipeline:

MQTT → Raspberry Pi → Azure IoT Hub → Cosmos DB

---

### Step 1: Install required packages

Inside the Azure virtual environment:

```bash
source azure-env/bin/activate
pip install azure-iot-device paho-mqtt
```

---

### Step 2: Create the MQTT → Azure Forwarder Script

Create a new file on the Raspberry Pi:

```bash
nano mqtt_to_azure.py
```

Paste the following code:

```python
from azure.iot.device import IoTHubDeviceClient, Message
import paho.mqtt.client as mqtt

# Azure IoT Hub device connection string
CONNECTION_STRING = "YOUR_DEVICE_CONNECTION_STRING"

# Create Azure IoT client
azure_client = IoTHubDeviceClient.create_from_connection_string(CONNECTION_STRING)

# Callback function: executed when an MQTT message is received
def on_message(client, userdata, msg):
    payload = msg.payload.decode()

    print("Received from MQTT:", payload)

    # Forward payload directly to Azure IoT Hub
    message = Message(payload)
    azure_client.send_message(message)

    print("Forwarded to Azure IoT Hub!")

# MQTT client setup
mqtt_client = mqtt.Client()
mqtt_client.on_message = on_message

# Connect to local Mosquitto broker
mqtt_client.connect("localhost", 1883)

# Subscribe to simulator topic
mqtt_client.subscribe("sensor/campus02-token")

print("Listening for MQTT messages and forwarding to Azure...")

# Run forever
mqtt_client.loop_forever()
```

---

### Step 3: Run the Forwarder

Start the script:

```bash
source azure-env/bin/activate
python3 mqtt_to_azure.py
```

---

### Step 4: Run the Simulator

In another terminal, start the simulator:

```bash
python3 mqtt_simulator.py
```

The forwarder will now receive MQTT messages and upload them to Azure IoT Hub in real time.

---

### Step 5: Verify Telemetry in Azure

Telemetry can be monitored in the Azure Portal under:

IoT Hub → Devices → Device Telemetry

Once IoT Hub receives the messages, they can be routed into Cosmos DB using Message Routing.


## Step 11: Troubleshooting Notes

- Mosquitto commands must be run on the Raspberry Pi terminal or via SSH.
- If Grafana installation fails, verify repository setup.
- Azure student subscriptions restrict deployment regions.
- Python Azure SDK must be installed in a virtual environment.

---


