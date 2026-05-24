##  Firmware & Control Architecture

The core software of Bills is developed in C++ using the Arduino framework, optimized for the **ESP32-S3** microcontroller. It implements a closed-loop control system combining real-time telemetry, Bluetooth communication, and precise locomotion.

###  Core Software Features
*  **Proportional Distance Control:** Smooth deceleration profile based on target distance to avoid overshooting cell boundaries.
*  **PID Alignment:** High-frequency ($1000\text{Hz}$ target loop) differential encoder balancing to keep the robot perfectly straight.
*  **Over-The-Air Tuning:** Real-time PID gain and target tuning via Bluetooth Low Energy (BLE).
*  **Live Telemetry:** Onboard I2C SSD1306 OLED display diagnostics monitoring motor speeds and encoder drift errors.

---

### Source Code (`main.cpp`)

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <ESP32Encoder.h>

ESP32Encoder encLeft;
ESP32Encoder encRight;

// ============================
// BLE CONFIGURATION
// ============================
BLECharacteristic *pCharacteristic;
#define SERVICE_UUID        "ea47ed89-d36f-4c52-9060-49eddaf4af45"
#define CHARACTERISTIC_UUID "d35385bb-709d-44c3-b677-f521ce350f9a"

// ============================
// DISPLAY CONFIGURATION
// ============================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
unsigned long lastDisplayUpdate = 0;

// ============================
// MOTOR PINOUT
// ============================
const int M2_A = 6;
const int M2_B = 7;
const int M1_A = 15;
const int M1_B = 16;

// ============================
// SENSOR PINOUT
// ============================
#define sld 2
#define sfd 18
#define sfe 3
#define sle 10
int minVal[4], maxVal[4];

// ============================
// PARAMETERS & PID TUNING
// ============================
const float PULSES_PER_REV = 14;
const float GEAR_RATIO = 20.0;
const float WHEEL_DIAMETER_CM = 2.2;
const float DIST_PER_PULSE_CM = (PI * WHEEL_DIAMETER_CM) / (PULSES_PER_REV * GEAR_RATIO);

float alvo = 18.0; // Target distance per cell (cm)
float distancia = 0;
int velocMax = 900;
int velocR = 0, velocL = 0;

// PID Gains for straight-line alignment (Encoder Balancing)
float Kp = 10.0, Ki = 3.0, Kd = 0.0; 
float error = 0, lastError = 0, integral = 0;
unsigned long lastTimePID = 0;

// Proportional Gain for target arrival execution (Distance Profile)
float Kp_dist = 65.0; // Conversion multiplier from error distance (cm) to PWM value

// ============================
// BLE CALLBACK HANDLER
// ============================
class MyCallbacks: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *pCharacteristic) {
    String value = pCharacteristic->getValue().c_str();
    if(value.startsWith("P:")) {
      value.remove(0, 2);
      int i1 = value.indexOf(',');
      int i2 = value.indexOf(',', i1 + 1);
      int i3 = value.indexOf(',', i2 + 1);
      Kp = value.substring(0, i1).toFloat();
      Ki = value.substring(i1+1, i2).toFloat();
      Kd = value.substring(i2+1, i3).toFloat();
      alvo = value.substring(i3+1).toFloat();
    }
  }
};

// ============================
// HELPER FUNCTIONS
// ============================
void atualizarDistancia() {
  // Calculates total traveled distance by averaging left and right wheel encoder counts
  distancia = ((abs(encLeft.getCount()) + abs(encRight.getCount())) / 2.0) * DIST_PER_PULSE_CM;
}

void pararMotores() {
  // Hard braking execution by pulling all motor control lines HIGH
  ledcWrite(M1_A, 1); ledcWrite(M1_B, 1);
  ledcWrite(M2_A, 1); ledcWrite(M2_B, 1);
}

// ============================
// ALIGNMENT PID CONTROL LOOP
// ============================
void pidAlinhamento(int vBase) {
  unsigned long now = micros();
  float dt = (now - lastTimePID) / 1000000.0;
  if (dt <= 0) dt = 0.001;
  lastTimePID = now;

  // Error is calculated as the difference between left and right encoders to maintain straight path
  error = encLeft.getCount() - encRight.getCount();
  integral += error * dt;
  float derivative = (error - lastError) / dt;
  lastError = error;

  float correction = (Kp * error) + (Ki * integral) + (Kd * derivative);

  // Computes closed-loop bounded individual motor velocities
  velocL = constrain(vBase - (int)correction, -4095, 4095);
  velocR = constrain(vBase + (int)correction, -4095, 4095);
}

void acionarMotores() {
  // Left Motor Execution (M2)
  if (velocL >= 0) { ledcWrite(M2_A, 0); ledcWrite(M2_B, velocL); }
  else { ledcWrite(M2_A, abs(velocL)); ledcWrite(M2_B, 0); }

  // Right Motor Execution (M1)
  if (velocR >= 0) { ledcWrite(M1_A, 0);  ledcWrite(M1_B, velocR); }
  else { ledcWrite(M1_A, abs(velocR)); ledcWrite(M1_B, 0); }
}

// ============================
// CELL NAVIGATION LOGIC
// ============================
void andarCelula() {
  atualizarDistancia();
  
  float erroDistancia = alvo - distancia;

  // Dead-band Check: If closer than 0.2cm to the target cell, stop execution.
  if (abs(erroDistancia) <= 0.2) {
    pararMotores();
    return;
  }

  // Base velocity scales proportionally down depending on distance left to target
  // Fast travel at long range, progressive deceleration when approaching target cell
  int vBase = constrain((int)(erroDistancia * Kp_dist), -velocMax, velocMax);

  // Compensates lateral motor drift using encoder tracking
  pidAlinhamento(vBase);
  
  // Applies computed outputs to the H-Bridges
  acionarMotores();
}

// ============================
// HARDWARE INITIALIZATION
// ============================
void setup() {
  Serial.begin(115200);
  Wire.begin();

  // Encoder Configuration
  ESP32Encoder::useInternalWeakPullResistors = puType::up;
  encLeft.attachHalfQuad(5, 4);
  encRight.attachHalfQuad(1, 11); // Pin assignment matching updated hardware definitions
  encLeft.clearCount();
  encRight.clearCount();

  // OLED Initialization
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.println("Starting...");
  display.display();

  // PWM Configuration (ESP32 ledc architecture - 12-bit resolution)
  ledcAttach(M1_A, 980, 12);
  ledcAttach(M1_B, 980, 12);
  ledcAttach(M2_A, 980, 12);
  ledcAttach(M2_B, 980, 12);

  // Bluetooth Low Energy (BLE) Stack Setup
  BLEDevice::init("Micromouse_V2");
  BLEServer *pServer = BLEDevice::createServer();
  BLEService *pService = pServer->createService(SERVICE_UUID);
  pCharacteristic = pService->createCharacteristic(CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_WRITE);
  pCharacteristic->setCallbacks(new MyCallbacks());
  pService->start();
  BLEDevice::getAdvertising()->start();
}

// ============================
// TELEMETRY DISPLAY HANDLING
// ============================
void atualizaDisplay() {
  if (millis() - lastDisplayUpdate < 1000) return;
  lastDisplayUpdate = millis();

  display.clearDisplay();
  display.setCursor(0, 0);
  display.print("Dist: "); display.println(distancia);
  display.print("vL: "); display.print(velocL);
  display.print(" vR: "); display.println(velocR);
  display.print("Align Err: "); display.println(error);
  display.display();
}

// ============================
// MAIN EXECUTION LOOP
// ============================
void loop() {
  andarCelula();
  atualizaDisplay();
}
