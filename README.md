/************************************************************
   SMART IoT ENVIRONMENTAL MONITORING & SAFETY SYSTEM

   Controller : ESP32
   Sensors    : DHT22 + LDR + MQ-2 + PIR
   Output     : OLED + LEDs + Buzzer + Relay
   Cloud      : Blynk IoT
************************************************************/

#define BLYNK_TEMPLATE_ID "TMPL35C5xbkzp"
#define BLYNK_TEMPLATE_NAME "Smart Environmental Monitor"
#define BLYNK_AUTH_TOKEN "PXr9cPKcg9bM_kwMNBwBJp-xyU23Bfqg"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>

// ================= WIFI =================

char ssid[] = "Wokwi-GUEST";
char pass[] = "";

// ================= PINS =================

#define DHT_PIN       15
#define DHT_TYPE      DHT22

#define LDR_PIN       34

#define MQ2_AO        35
#define MQ2_DO        26

#define PIR_PIN       13

#define RELAY_PIN     23

#define GREEN_LED     5
#define YELLOW_LED    18
#define RED_LED       2

#define BUZZER_PIN    4

#define FAULT_BUTTON  27
#define RESET_BUTTON  14

// ================= OLED =================

#define OLED_SDA      21
#define OLED_SCL      22

#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT 64
#define OLED_ADDRESS  0x3C

Adafruit_SSD1306 display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &Wire,
  -1
);

DHT dht(DHT_PIN, DHT_TYPE);

// ================= BLYNK =================

#define VPIN_TEMP      V0
#define VPIN_HUM       V1
#define VPIN_LIGHT     V2
#define VPIN_STATUS    V3
#define VPIN_ALERT     V4
#define VPIN_GAS       V5
#define VPIN_MOTION    V6
#define VPIN_RELAY     V7

// ================= VARIABLES =================

float temperature = 0.0;
float humidity = 0.0;

int ldrRaw = 0;
int gasRaw = 0;

float lightPercent = 0.0;

bool gasDetected = false;
bool motionDetected = false;
bool manualFault = false;
bool relayState = false;

String systemStatus = "STARTING";
String alertMessage = "System Starting";

// ================= THRESHOLDS =================

const float TEMP_WARNING_HIGH  = 35.0;
const float TEMP_CRITICAL_HIGH = 45.0;

const float HUM_WARNING_HIGH   = 80.0;
const float HUM_CRITICAL_HIGH  = 90.0;

const float LIGHT_WARNING_LOW  = 10.0;

const int GAS_ANALOG_THRESHOLD = 4095;

// ================= TEST MODE =================

// true  = test values
// false = real DHT22 values

#define TEST_MODE true

const float TEST_TEMPERATURE = 25.0;
const float TEST_HUMIDITY = 40.0;

// ================= STATES =================

enum SystemState
{
  NORMAL,
  WARNING,
  CRITICAL,
  SENSOR_ERROR
};

SystemState currentState = NORMAL;

// ================= TIMERS =================

unsigned long lastSensorRead = 0;
unsigned long lastOLEDUpdate = 0;
unsigned long lastBlynkUpdate = 0;
unsigned long lastButtonCheck = 0;

const unsigned long SENSOR_INTERVAL  = 2000;
const unsigned long OLED_INTERVAL    = 1000;
const unsigned long BLYNK_INTERVAL   = 5000;
const unsigned long BUTTON_INTERVAL  = 100;

// ==========================================================
// READ SENSORS
// ==========================================================

void readSensors()
{
#if TEST_MODE

  temperature = TEST_TEMPERATURE;
  humidity = TEST_HUMIDITY;

#else

  temperature = dht.readTemperature();
  humidity = dht.readHumidity();

#endif

  // LDR
  ldrRaw = analogRead(LDR_PIN);

  lightPercent =
    (ldrRaw / 4095.0) * 100.0;

  lightPercent =
    constrain(lightPercent, 0.0, 100.0);

  // MQ-2
  gasRaw = analogRead(MQ2_AO);

// Temporary NORMAL testing
gasDetected = false;

  // PIR
  motionDetected =
    digitalRead(PIR_PIN) == HIGH;
}

// ==========================================================
// DECISION ENGINE
// ==========================================================

void checkConditions()
{
  systemStatus = "NORMAL";
  alertMessage = "No Alert";

  // DHT22 sensor failure
  if (isnan(temperature) || isnan(humidity))
  {
    currentState = SENSOR_ERROR;

    systemStatus = "SENSOR ERROR";
    alertMessage = "DHT22 Error";

    return;
  }

  // Manual fault
  if (manualFault)
  {
    currentState = CRITICAL;

    systemStatus = "CRITICAL";
    alertMessage = "Manual Fault";

    return;
  }

  // Critical temperature
  if (temperature >= TEMP_CRITICAL_HIGH)
  {
    currentState = CRITICAL;

    systemStatus = "CRITICAL";
    alertMessage = "Critical Temperature";

    return;
  }

  // Critical humidity
  if (humidity >= HUM_CRITICAL_HIGH)
  {
    currentState = CRITICAL;

    systemStatus = "CRITICAL";
    alertMessage = "Critical Humidity";

    return;
  }

  // Gas detection
  if (gasDetected)
  {
    currentState = CRITICAL;

    systemStatus = "CRITICAL";
    alertMessage = "Gas / Smoke Detected";

    return;
  }

  // Temperature warning
  if (temperature >= TEMP_WARNING_HIGH)
  {
    currentState = WARNING;

    systemStatus = "WARNING";
    alertMessage = "High Temperature";

    return;
  }

  // Humidity warning
  if (humidity >= HUM_WARNING_HIGH)
  {
    currentState = WARNING;

    systemStatus = "WARNING";
    alertMessage = "High Humidity";

    return;
  }

  // Low light
  if (lightPercent <= LIGHT_WARNING_LOW)
  {
    currentState = WARNING;

    systemStatus = "WARNING";
    alertMessage = "Low Light";

    return;
  }

  // Normal
  currentState = NORMAL;

  systemStatus = "NORMAL";
  alertMessage = "No Alert";
}

// ==========================================================
// ACTUATORS
// ==========================================================

void updateOutputs()
{
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(RED_LED, LOW);

  noTone(BUZZER_PIN);

  // NORMAL
  if (currentState == NORMAL)
  {
    digitalWrite(GREEN_LED, HIGH);

    digitalWrite(RELAY_PIN, LOW);
    relayState = false;
  }

  // WARNING
  else if (currentState == WARNING)
  {
    digitalWrite(YELLOW_LED, HIGH);

    if (temperature >= TEMP_WARNING_HIGH)
    {
      digitalWrite(RELAY_PIN, HIGH);
      relayState = true;
    }
    else
    {
      digitalWrite(RELAY_PIN, LOW);
      relayState = false;
    }
  }

  // CRITICAL
  else if (currentState == CRITICAL)
  {
    digitalWrite(RED_LED, HIGH);

    tone(BUZZER_PIN, 1500);

    digitalWrite(RELAY_PIN, HIGH);
    relayState = true;
  }

  // SENSOR ERROR
  else if (currentState == SENSOR_ERROR)
  {
    digitalWrite(RED_LED, HIGH);

    tone(BUZZER_PIN, 2000);

    digitalWrite(RELAY_PIN, HIGH);
    relayState = true;
  }
}

// ==========================================================
// OLED
// ==========================================================

void updateOLED()
{
  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  display.setCursor(0, 0);
  display.println("SMART ENV MONITOR");

  display.drawLine(
    0, 10, 127, 10,
    SSD1306_WHITE
  );

  display.setCursor(0, 15);

  display.print("T:");
  display.print(temperature, 1);
  display.print("C ");

  display.print("H:");
  display.print(humidity, 1);
  display.println("%");

  display.setCursor(0, 27);

  display.print("Light:");
  display.print(lightPercent, 0);
  display.println("%");

  display.setCursor(0, 39);

  display.print("Gas:");
  display.print(gasRaw);

  display.print(" PIR:");

  if (motionDetected)
    display.println("YES");
  else
    display.println("NO");

  display.setCursor(0, 52);

  display.print(systemStatus);

  display.display();
}

// ==========================================================
// BLYNK
// ==========================================================

void sendToBlynk()
{
  if (!Blynk.connected())
  {
    return;
  }

  Blynk.virtualWrite(
    VPIN_TEMP,
    temperature
  );

  Blynk.virtualWrite(
    VPIN_HUM,
    humidity
  );

  Blynk.virtualWrite(
    VPIN_LIGHT,
    lightPercent
  );

  Blynk.virtualWrite(
    VPIN_STATUS,
    systemStatus
  );

  Blynk.virtualWrite(
    VPIN_ALERT,
    alertMessage
  );

  Blynk.virtualWrite(
    VPIN_GAS,
    gasRaw
  );

  Blynk.virtualWrite(
    VPIN_MOTION,
    motionDetected
  );

  Blynk.virtualWrite(
    VPIN_RELAY,
    relayState
  );
}

// ==========================================================
// SERIAL MONITOR
// ==========================================================

void printSerial()
{
  Serial.println();
  Serial.println("================================");
  Serial.println("SMART ENVIRONMENT MONITOR");
  Serial.println("================================");

  Serial.print("Temperature : ");
  Serial.print(temperature);
  Serial.println(" C");

  Serial.print("Humidity    : ");
  Serial.print(humidity);
  Serial.println(" %");

  Serial.print("Light       : ");
  Serial.print(lightPercent);
  Serial.println(" %");

  Serial.print("Gas Raw     : ");
  Serial.println(gasRaw);

  Serial.print("Gas Status  : ");

  if (gasDetected)
    Serial.println("DETECTED");
  else
    Serial.println("NORMAL");

  Serial.print("Motion      : ");

  if (motionDetected)
    Serial.println("DETECTED");
  else
    Serial.println("NONE");

  Serial.print("Status      : ");
  Serial.println(systemStatus);

  Serial.print("Alert       : ");
  Serial.println(alertMessage);

  Serial.print("Relay       : ");

  if (relayState)
    Serial.println("ON");
  else
    Serial.println("OFF");

  Serial.println("================================");
}

// ==========================================================
// BUTTONS
// ==========================================================

void checkButtons()
{
  // Fault button
  if (digitalRead(FAULT_BUTTON) == LOW)
  {
    manualFault = true;

    Serial.println(
      ">>> MANUAL FAULT ACTIVATED <<<"
    );

    delay(200);
  }

  // Reset button
  if (digitalRead(RESET_BUTTON) == LOW)
  {
    manualFault = false;

    Serial.println(
      ">>> SYSTEM RESET <<<"
    );

    delay(200);
  }
}

// ==========================================================
// SETUP
// ==========================================================

void setup()
{
  Serial.begin(115200);

  // GPIO
  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);

  pinMode(BUZZER_PIN, OUTPUT);

  pinMode(RELAY_PIN, OUTPUT);

  pinMode(MQ2_DO, INPUT);

  pinMode(PIR_PIN, INPUT);

  pinMode(
    FAULT_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    RESET_BUTTON,
    INPUT_PULLUP
  );

  // Initial outputs
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(RED_LED, LOW);

  digitalWrite(RELAY_PIN, LOW);

  noTone(BUZZER_PIN);

  // DHT
  dht.begin();

  // I2C
  Wire.begin(
    OLED_SDA,
    OLED_SCL
  );

  // OLED
  if (!display.begin(
        SSD1306_SWITCHCAPVCC,
        OLED_ADDRESS))
  {
    Serial.println(
      "OLED initialization failed!"
    );
  }
  else
  {
    display.clearDisplay();

    display.setTextColor(
      SSD1306_WHITE
    );

    display.setTextSize(1);

    display.setCursor(15, 20);
    display.println("SMART IoT");

    display.setCursor(8, 35);
    display.println("ENV MONITOR");

    display.display();

    delay(1500);
  }

  // WiFi
  Serial.println(
    "Connecting to WiFi..."
  );

  WiFi.begin(
    ssid,
    pass
  );

  unsigned long startTime =
    millis();

  while (
    WiFi.status() != WL_CONNECTED &&
    millis() - startTime < 15000
  )
  {
    delay(500);

    Serial.print(".");
  }

  Serial.println();

  // Blynk
  if (WiFi.status() == WL_CONNECTED)
  {
    Serial.println(
      "WiFi Connected!"
    );

    Blynk.config(
      BLYNK_AUTH_TOKEN
    );

    if (Blynk.connect(10000))
    {
      Serial.println(
        "Blynk Connected!"
      );
    }
    else
    {
      Serial.println(
        "Blynk Connection Failed"
      );
    }
  }

  Serial.println();
  Serial.println(
    "SMART ENVIRONMENT SYSTEM READY"
  );
}

// ==========================================================
// LOOP
// ==========================================================

void loop()
{
  if (Blynk.connected())
  {
    Blynk.run();
  }

  unsigned long now = millis();

  // Sensor
  if (
    now - lastSensorRead >=
    SENSOR_INTERVAL
  )
  {
    lastSensorRead = now;

    readSensors();

    checkConditions();

    updateOutputs();

    printSerial();
  }

  // OLED
  if (
    now - lastOLEDUpdate >=
    OLED_INTERVAL
  )
  {
    lastOLEDUpdate = now;

    updateOLED();
  }

  // Blynk
  if (
    now - lastBlynkUpdate >=
    BLYNK_INTERVAL
  )
  {
    lastBlynkUpdate = now;

    sendToBlynk();
  }

  // Buttons
  if (
    now - lastButtonCheck >=
    BUTTON_INTERVAL
  )
  {
    lastButtonCheck = now;

    checkButtons();
  }
}
