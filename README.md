# 🌱 Smart IoT Environmental Monitoring & Safety System

An **ESP32-based IoT project** that monitors environmental conditions using multiple sensors and displays the system status locally and on the **Blynk IoT Cloud**.

## 🔧 Components

* ESP32
* DHT22 – Temperature & Humidity
* LDR – Light monitoring
* MQ-2 – Gas/Smoke sensing
* PIR – Motion detection
* 0.96" OLED Display
* Green, Yellow & Red LEDs
* Buzzer
* Relay
* Fault & Reset buttons

## ⚙️ Features

* Real-time temperature and humidity monitoring
* Light-level monitoring
* Gas/smoke sensor monitoring
* Motion detection
* OLED status display
* Warning and critical safety states
* Buzzer and LED alerts
* Relay control during warning/critical conditions
* Remote monitoring through Blynk IoT
* Manual fault and reset functionality

## 🧠 System Logic

The system operates in four states:

* **NORMAL** – Green LED, system operating normally
* **WARNING** – Yellow LED when temperature, humidity or light crosses warning limits
* **CRITICAL** – Red LED, buzzer and relay activation
* **SENSOR ERROR** – Safety response when the DHT22 reading is invalid

## 🛠️ Technologies Used

**ESP32 | Embedded C/C++ | Arduino | Blynk IoT | Wokwi | I2C OLED**

## 🧪 Simulation

The project is developed and tested in **Wokwi**, allowing the sensor readings, alerts, OLED display and actuator responses to be tested without physical hardware.

## 📌 Project Status

**Completed simulation prototype**

The current version is focused on demonstrating the monitoring, alert and IoT-cloud functionality in simulation.

## 👩‍💻 Author

**Sapna Thakur**
B.Tech ECE | Embedded Systems & IoT Enthusiast
