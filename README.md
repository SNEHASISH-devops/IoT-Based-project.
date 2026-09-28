#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// =====================================================
// SMART FIRE & GAS SAFETY SYSTEM
// Wokwi simulation version
// =====================================================

#define GAS_PIN       34
#define SMOKE_PIN     33
#define FLAME_PIN     32

#define BUZZER_PIN    25

// Single RGB LED (common cathode)
#define LED_RED       14
#define LED_GREEN     27
#define LED_BLUE      26

// Relay controls the simulated exhaust/fan indicator
#define RELAY_PIN     4

// Silence button
#define RESET_BUTTON  13

// OLED I2C
#define OLED_SDA      21
#define OLED_SCL      22
#define OLED_ADDRESS  0x3C

#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT 64

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

enum SystemState {
  SAFE,
  WARNING,
  DANGER,
  EMERGENCY
};

SystemState currentState = SAFE;

int gasRaw = 0;
int smokeRaw = 0;
int gasLevel = 0;
int smokeLevel = 0;
bool flameDetected = false;

const int WARNING_LEVEL   = 25;
const int DANGER_LEVEL    = 55;
const int EMERGENCY_LEVEL = 80;

bool buzzerSilenced = false;
bool buzzerOn = false;

unsigned long buzzerTimer = 0;
unsigned long startupTime = 0;
unsigned long lastSensorRead = 0;
unsigned long lastDisplay = 0;
unsigned long lastSerial = 0;

bool systemStarted = false;
bool lastButtonState = HIGH;
unsigned long lastButtonChange = 0;
const unsigned long DEBOUNCE_TIME = 50;

// =====================================================
// STATUS LED
// =====================================================
void setStatusLED(bool red, bool green, bool blue) {
  digitalWrite(LED_RED, red ? HIGH : LOW);
  digitalWrite(LED_GREEN, green ? HIGH : LOW);
  digitalWrite(LED_BLUE, blue ? HIGH : LOW);
}

// =====================================================
// SETUP
// =====================================================
void setup() {
  Serial.begin(115200);

  pinMode(GAS_PIN, INPUT);
  pinMode(SMOKE_PIN, INPUT);
  pinMode(FLAME_PIN, INPUT_PULLUP);
  pinMode(RESET_BUTTON, INPUT_PULLUP);

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_BLUE, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  setStatusLED(false, false, false);
  digitalWrite(RELAY_PIN, LOW);
  noTone(BUZZER_PIN);

  Wire.begin(OLED_SDA, OLED_SCL);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("OLED ERROR");
    while (true) {
      delay(100);
    }
  }

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  display.setCursor(10, 5);
  display.println("SMART FIRE & GAS");
  display.setCursor(24, 19);
  display.println("SAFETY SYSTEM");
  display.setCursor(30, 40);
  display.println("INITIALIZING...");
  display.display();

  startupTime = millis();
}

// =====================================================
// LOOP
// =====================================================
void loop() {
  unsigned long now = millis();

  if (!systemStarted) {
    if (now - startupTime < 3000) {
      return;
    }

    systemStarted = true;
    Serial.println("SMART SAFETY SYSTEM READY");
  }

  if (now - lastSensorRead >= 100) {
    lastSensorRead = now;
    readSensors();
    determineState();
    updateOutputs();
  }

  if (now - lastDisplay >= 250) {
    lastDisplay = now;
    updateOLED();
  }

  if (now - lastSerial >= 500) {
    lastSerial = now;
    printSerial();
  }

  updateBuzzer(now);
  handleResetButton();
}

// =====================================================
// READ SENSORS
// =====================================================
void readSensors() {
  gasRaw = analogRead(GAS_PIN);
  smokeRaw = analogRead(SMOKE_PIN);
  flameDetected = (digitalRead(FLAME_PIN) == LOW);

  // Relative simulation percentages only.
  // These are not calibrated real-world PPM values.
  gasLevel = constrain(map(gasRaw, 0, 4095, 0, 100), 0, 100);
  smokeLevel = constrain(map(smokeRaw, 0, 4095, 0, 100), 0, 100);
}

// =====================================================
// DETERMINE STATE
// =====================================================
void determineState() {
  SystemState newState;

  if (flameDetected) {
    newState = EMERGENCY;
  } else if (gasLevel >= EMERGENCY_LEVEL || smokeLevel >= EMERGENCY_LEVEL) {
    newState = EMERGENCY;
  } else if (gasLevel >= DANGER_LEVEL || smokeLevel >= DANGER_LEVEL) {
    newState = DANGER;
  } else if (gasLevel >= WARNING_LEVEL || smokeLevel >= WARNING_LEVEL) {
    newState = WARNING;
  } else {
    newState = SAFE;
  }

  if (newState != currentState) {
    currentState = newState;
    buzzerSilenced = false;
    buzzerOn = false;
    buzzerTimer = millis();
  }
}

// =====================================================
// OUTPUTS
// =====================================================
void updateOutputs() {
  switch (currentState) {
    case SAFE:
      setStatusLED(false, true, false);       // GREEN
      digitalWrite(RELAY_PIN, LOW);
      break;

    case WARNING:
      setStatusLED(true, true, false);        // YELLOW
      digitalWrite(RELAY_PIN, LOW);
      break;

    case DANGER:
      setStatusLED(true, false, false);       // RED
      digitalWrite(RELAY_PIN, HIGH);
      break;

    case EMERGENCY:
      setStatusLED(true, false, false);       // RED
      digitalWrite(RELAY_PIN, HIGH);
      break;
  }
}

// =====================================================
// BUZZER
// =====================================================
void updateBuzzer(unsigned long now) {
  if (currentState == SAFE || currentState == WARNING || buzzerSilenced) {
    noTone(BUZZER_PIN);
    buzzerOn = false;
    return;
  }

  if (currentState == EMERGENCY) {
    if (!buzzerOn) {
      tone(BUZZER_PIN, 2200);
      buzzerOn = true;
    }
    return;
  }

  // DANGER: 750 ms ON, 250 ms OFF
  if (buzzerOn) {
    if (now - buzzerTimer >= 250) {
      noTone(BUZZER_PIN);
      buzzerOn = false;
      buzzerTimer = now;
    }
  } else {
    if (now - buzzerTimer >= 750) {
      tone(BUZZER_PIN, 1700);
      buzzerOn = true;
      buzzerTimer = now;
    }
  }
}

// =====================================================
// SILENCE BUTTON
// =====================================================
void handleResetButton() {
  bool reading = digitalRead(RESET_BUTTON);

  if (reading != lastButtonState) {
    lastButtonChange = millis();
    lastButtonState = reading;
  }

  if (millis() - lastButtonChange > DEBOUNCE_TIME && reading == LOW) {
    if (!buzzerSilenced) {
      buzzerSilenced = true;
      noTone(BUZZER_PIN);
      buzzerOn = false;
      Serial.println("ALARM SILENCED");
    }
  }
}

// =====================================================
// OLED
// =====================================================
void updateOLED() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  display.setCursor(0, 0);
  display.println("SMART SAFETY SYSTEM");

  display.setCursor(0, 12);
  display.print("GAS   : ");
  display.print(gasLevel);
  display.println("%");

  display.setCursor(0, 23);
  display.print("SMOKE : ");
  display.print(smokeLevel);
  display.println("%");

  display.setCursor(0, 34);
  display.print("FLAME : ");
  display.println(flameDetected ? "DETECTED" : "SAFE");

  display.setCursor(0, 45);
  display.print("STATUS: ");

  switch (currentState) {
    case SAFE:       display.println("SAFE"); break;
    case WARNING:    display.println("WARNING"); break;
    case DANGER:     display.println("DANGER"); break;
    case EMERGENCY:  display.println("FIRE!"); break;
  }

  display.setCursor(0, 56);
  display.print("EXHAUST: ");
  display.println(
    (currentState == DANGER || currentState == EMERGENCY) ? "ON" : "OFF"
  );

  display.display();
}

// =====================================================
// SERIAL MONITOR
// =====================================================
void printSerial() {
  Serial.print("GasRaw=");
  Serial.print(gasRaw);
  Serial.print(" | Gas=");
  Serial.print(gasLevel);
  Serial.print("% | SmokeRaw=");
  Serial.print(smokeRaw);
  Serial.print(" | Smoke=");
  Serial.print(smokeLevel);
  Serial.print("% | Flame=");
  Serial.print(flameDetected ? "YES" : "NO");
  Serial.print(" | State=");

  switch (currentState) {
    case SAFE:       Serial.print("SAFE"); break;
    case WARNING:    Serial.print("WARNING"); break;
    case DANGER:     Serial.print("DANGER"); break;
    case EMERGENCY:  Serial.print("EMERGENCY"); break;
  }

  Serial.print(" | Exhaust=");
  Serial.println(
    (currentState == DANGER || currentState == EMERGENCY) ? "ON" : "OFF"
  );
}
