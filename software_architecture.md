# Software Architecture and Communication Protocols

## Introduction
This document outlines the software architecture and communication protocols for the Ambulance GPS + Traffic Alert System. The software components are designed to be modular, scalable, and secure, ensuring reliable and efficient operation of the entire system.

## 1. Ambulance-Side IoT Device Firmware

### Platform
*   **Framework:** Arduino Core for ESP32. This provides a familiar and well-supported environment for programming the ESP32 microcontroller.
*   **Libraries:**
    *   **TinyGPS++:** For parsing NMEA sentences from the U-blox GPS module.
    *   **PubSubClient:** For MQTT communication with the central cloud platform.
    *   **ArduinoJson:** For creating and parsing JSON payloads.
    *   **SIM7600_LTE_Shield:** For interfacing with the SIM7600 cellular module.

### Core Logic
1.  **Initialization:**
    *   Initialize serial communication with the GPS and cellular modules.
    *   Establish a connection to the cellular network (GPRS/LTE).
    *   Connect to the MQTT broker on the central cloud platform.
2.  **Main Loop:**
    *   Continuously read GPS data (latitude, longitude, speed, heading).
    *   Check for a valid GPS fix.
    *   If a valid fix is obtained, format the data into a JSON payload.
    *   Publish the JSON payload to a specific MQTT topic (e.g., `ambulance/telemetry`).
    *   Implement a delay (e.g., every 1-2 seconds) to control the data transmission rate.
3.  **Emergency Status:**
    *   Implement a mechanism (e.g., a physical button or a software flag) to indicate the ambulance's emergency status.
    *   Include the emergency status in the JSON payload.
4.  **Over-the-Air (OTA) Updates:**
    *   Integrate OTA update capabilities to allow for remote firmware updates without physical access to the device.

### JSON Payload Format (Ambulance to Cloud)
```json
{
  "deviceId": "AMB-001",
  "timestamp": 1678886400,
  "latitude": 34.0522,
  "longitude": -118.2437,
  "speed": 60,
  "heading": 90,
  "emergency": true
}
```

## 2. Central Cloud Platform

### Architecture
*   **Cloud Provider:** Amazon Web Services (AWS) or a similar cloud platform (e.g., Google Cloud, Microsoft Azure).
*   **Microservices Architecture:** The platform will be built using a microservices architecture to ensure scalability, resilience, and maintainability.

### Key Services

*   **AWS IoT Core:**
    *   **MQTT Broker:** To securely ingest data from ambulance-side IoT devices.
    *   **Rules Engine:** To process incoming MQTT messages and route them to other AWS services.
*   **AWS Lambda:**
    *   **Data Processing:** A Lambda function will be triggered by the IoT Core Rules Engine to process incoming JSON payloads. This function will:
        *   Validate the data.
        *   Determine the ambulance's route and predict affected intersections.
        *   Query a database of street-side IoT box locations.
        *   Send preemption commands to the relevant street-side IoT boxes.
*   **Amazon DynamoDB:**
    *   **Database:** A NoSQL database to store:
        *   Ambulance telemetry data (for historical analysis).
        *   Street-side IoT box locations and statuses.
        *   System configuration data.
*   **Amazon API Gateway:**
    *   **API Endpoints:** To provide secure RESTful APIs for:
        *   Street-side IoT boxes to receive preemption commands.
        *   A web-based dashboard for monitoring and management.
*   **Web-based Dashboard (e.g., using React or Angular):**
    *   **Features:**
        *   Real-time map visualization of ambulance locations.
        *   Status of street-side IoT boxes.
        *   System configuration and management.
        *   Historical data analysis and reporting.

### Preemption Command Format (Cloud to Street-Side IoT Box)
```json
{
  "boxId": "SSB-123",
  "command": "PREEMPT",
  "ambulanceId": "AMB-001",
  "direction": "NORTH",
  "duration": 30 // in seconds
}
```

## 3. Street-Side IoT Box Firmware/Software

### Platform
*   **ESP32-based:** Arduino Core for ESP32.
*   **Raspberry Pi-based:** Python with libraries like `paho-mqtt` for MQTT communication and `RPi.GPIO` for controlling peripherals.

### Core Logic
1.  **Initialization:**
    *   Connect to the network (Wi-Fi, Ethernet, or Cellular).
    *   Establish a connection to the central cloud platform via API or MQTT.
2.  **Listen for Commands:**
    *   Continuously listen for preemption commands from the cloud.
3.  **Execute Commands:**
    *   Upon receiving a `PREEMPT` command:
        *   Activate the high-intensity LED panel with the appropriate message.
        *   Trigger the siren/speaker system.
        *   Send a preemption request to the traffic signal controller via the interface module.
        *   Start broadcasting alerts via FM and Bluetooth beacons.
4.  **Broadcast Alerts:**
    *   **Bluetooth Beacons:** Broadcast BLE advertising packets with a specific UUID and major/minor values to identify the alert.
    *   **FM Transmission:** Transmit a pre-recorded audio message or a specific tone on a designated FM frequency.
5.  **Reset:**
    *   After the specified duration, deactivate all alerts and return the traffic signal to normal operation.

## 4. Vehicle-Side Prototype Firmware/Application

### Platform
*   **ESP32-based Device:** Arduino Core for ESP32.
*   **Android Application:** Java/Kotlin with the Android SDK.

### Core Logic (ESP32 Device)
1.  **Initialization:**
    *   Initialize Bluetooth and FM receiver modules.
2.  **Scan for Alerts:**
    *   Continuously scan for BLE beacon advertisements with the predefined UUID.
    *   Monitor the designated FM frequency for alert transmissions.
3.  **Trigger Alerts:**
    *   If a valid alert is received:
        *   Activate the vibration motor.
        *   Flash the LED indicator.
        *   Play an audible beep through the small speaker.

### Core Logic (Android Application)
1.  **Background Service:**
    *   Run a background service to continuously scan for BLE beacons.
2.  **Beacon Detection:**
    *   Use the Android Bluetooth LE APIs to detect beacons with the specific UUID.
3.  **User Notification:**
    *   When a beacon is detected, create a high-priority notification to alert the user.
    *   The notification can include sound, vibration, and a visual message on the screen.

## Communication Protocols Summary

| From                    | To                      | Protocol        | Payload Format | Notes                                         |
| :---------------------- | :---------------------- | :-------------- | :------------- | :-------------------------------------------- |
| Ambulance-Side IoT      | Central Cloud Platform  | MQTT over TLS   | JSON           | Secure and efficient telemetry data transmission. |
| Central Cloud Platform  | Street-Side IoT Box     | HTTPS or MQTT   | JSON           | Secure command and control communication.       |
| Street-Side IoT Box     | Traffic Signal Controller | Proprietary/NEMA | -              | Depends on existing traffic signal infrastructure. |
| Street-Side IoT Box     | Vehicle-Side Prototype  | BLE Beacons, FM | -              | Wireless broadcast of alerts to nearby vehicles. |

This comprehensive software architecture and communication protocol design ensures a robust, scalable, and secure foundation for the Ambulance GPS + Traffic Alert System.

