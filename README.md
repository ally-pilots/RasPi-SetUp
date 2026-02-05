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

## Step 11: Troubleshooting Notes

- Mosquitto commands must be run on the Raspberry Pi terminal or via SSH.
- If Grafana installation fails, verify repository setup.
- Azure student subscriptions restrict deployment regions.
- Python Azure SDK must be installed in a virtual environment.

---


