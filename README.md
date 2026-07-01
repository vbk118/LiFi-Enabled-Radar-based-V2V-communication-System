# LiFi-Enabled-Radar-based-V2V-communication-System
Designed a prototype for a LiFi enabled v2v communication system - two motored cars move back to back or one behind the other when there is any obstacle infront  of the front car  it will alert the back car using the LiFi system . Incase the driver doesn't slow down there is a radar and an US sensor using which  automated braking system is enabled

---------------------------------------------------------------------------------------------------

Code for Lifi transmitter , radar , USS , motor
lifi , radar , relay , lcd display 

// ===========================================================
//                    LIBRARIES
// ===========================================================
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Create LCD object (change 0x27 → 0x3F if needed)
LiquidCrystal_I2C lcd(0x27, 16, 2);


// ===========================================================
//                    DEFINES + GLOBALS
// ===========================================================

// ---- LiFi ----
#define LED_PIN 3
#define BIT_DELAY 300
char message[] = "HELLO";

int msgIndex = 0;           
int bitIndex = 10;          
unsigned long lastBitTime = 0;
bool transmitting = false;

// ---- Ultrasonic ----
#define TRIG_PIN 6
#define ECHO_PIN 7

// ---- Relay ----
#define RELAY_PIN 8

long duration;
int distance;


// ===========================================================
//                           SETUP
// ===========================================================
void setup() {
  Serial.begin(9600);

  // LCD INIT
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("LiFi V2V System");

  // LiFi LED
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  // Ultrasonic
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Relay
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);

  Serial.println("System Ready (NON-BLOCKING)");
}


// ===========================================================
//                           LOOP
// ===========================================================
void loop() {

  // Read ultrasonic continuously
  distance = getDistance();

  Serial.print("Distance: ");
  Serial.println(distance);

  // ---------------- LCD MESSAGES -------------------
  lcd.setCursor(0, 1);

  if (distance <= 20) {
    lcd.print("STOP!          ");
  }
  else if (distance <= 30) {
    lcd.print("Object Ahead!  ");
  }
  else {
    lcd.print("Clear Ahead    ");
  }

  // ---------------- Relay logic ---------------------
  digitalWrite(RELAY_PIN, distance >= 20);

  // ---------------- LiFi Start condition -----------
  if (distance <= 30) {
    if (!transmitting) {
      transmitting = true;
      msgIndex = 0;
      bitIndex = 10;
      lastBitTime = millis();
    }
  } else {
    transmitting = false;
    digitalWrite(LED_PIN, LOW);
  }

  // Run LiFi transmitter WITHOUT stopping ultrasonic
  if (transmitting) {
    runLiFiNonBlocking();
  }
}



// ===========================================================
//                   ULTRASONIC FUNCTION
// ===========================================================
int getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  duration = pulseIn(ECHO_PIN, HIGH, 25000);

  if (duration == 0) return 999;

  return duration * 0.0343 / 2;
}



// ===========================================================
//               NON-BLOCKING LIFI TRANSMISSION
// ===========================================================
void runLiFiNonBlocking() {

  if (millis() - lastBitTime < BIT_DELAY) return;
  lastBitTime = millis();

  if (msgIndex >= strlen(message)) { 
    transmitting = false;
    digitalWrite(LED_PIN, LOW);
    return;
  }

  byte data = message[msgIndex];
  int bitToSend;

  if (bitIndex == 10) bitToSend = 1;              
  else if (bitIndex == 1) bitToSend = 0;          
  else bitToSend = (data >> (bitIndex - 2)) & 1;  

  digitalWrite(LED_PIN, bitToSend);

  bitIndex--;

  if (bitIndex < 1) {  
    bitIndex = 10;
    msgIndex++;  
  }
}

-------------------------------------------------------------------------------------------------

code for lifi receiver , radar , USS , motor

lifi reciever , relay , ultrasonic , motor 

// ================================================================
//                 NON-BLOCKING LIFI + ULTRASONIC + LCD + RELAY
// ================================================================

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27, 16, 2);   // Change to 0x3F if needed

// ----------- LiFi (LDR Receiver) -----------
#define LDR_PIN A0
#define BIT_DELAY 300
#define SAMPLES 6
int THRESHOLD = 530;

// ----------- Ultrasonic Sensor -----------
#define TRIG_PIN 6
#define ECHO_PIN 7

// ----------- Relay -----------
#define RELAY_PIN 8    

// ----------- LiFi State Machine -----------
enum LiFiState { WAIT_START, READ_BITS };
LiFiState lifiState = WAIT_START;

unsigned long lastBitTime = 0;
byte receivedByte = 0;
int bitIndex = 7;

bool sosDetected = false;   // TRUE when LiFi receives 'S'

// ================================================================
//                           SETUP
// ================================================================
void setup() {
  Serial.begin(9600);

  pinMode(LDR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, HIGH);

  // LCD INIT
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("LiFi Receiver");
  lcd.setCursor(0, 1);
  lcd.print("System Ready");
  delay(1000);
  lcd.clear();

  Serial.println("SYSTEM RUNNING (NON-BLOCKING)");
}

// ================================================================
//                   LDR BIT READING (LI-FI RX)
// ================================================================
int readBit() {
  int sum = 0;
  for (int i = 0; i < SAMPLES; i++) {
    sum += analogRead(LDR_PIN);
    delayMicroseconds(800);
  }
  int avg = sum / SAMPLES;
  return (avg < THRESHOLD) ? 1 : 0;
}

// ================================================================
//                       ULTRASONIC DISTANCE
// ================================================================
int getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(3);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 25000);
  if (duration == 0) return 999;

  return duration * 0.034 / 2;
}

// ================================================================
//               NON-BLOCKING LIFI RECEIVE FSM
// ================================================================
void updateLiFi() {
  int bit = readBit();

  switch (lifiState) {

    case WAIT_START:
      if (bit == 1) {  // START pulse
        lifiState = READ_BITS;
        lastBitTime = millis();
        receivedByte = 0;
        bitIndex = 7;
      }
      break;

    case READ_BITS:
      if (millis() - lastBitTime >= BIT_DELAY) {

        receivedByte |= (bit << bitIndex);
        bitIndex--;
        lastBitTime = millis();

        if (bitIndex < 0) {
          char c = (char)receivedByte;
          Serial.print("LiFi Received: ");
          Serial.println(c);

          if (c == 'S') {
            sosDetected = true;   // SOS TRIGGER
          }

          lifiState = WAIT_START;
        }
      }
      break;
  }
}

// ================================================================
//                          MAIN LOOP
// ================================================================
void loop() {

  int distance = getDistance();
  Serial.print("Distance: ");
  Serial.println(distance);

  // Run LiFi receiver when object is nearby (<30 cm)
  if (distance < 30) updateLiFi();

  // -------- LCD Priority Logic --------
  lcd.setCursor(0, 0);
  lcd.print("Dist:");
  lcd.print(distance);
  lcd.print("cm     ");

  lcd.setCursor(0, 1);

  if (distance < 20) {
    lcd.print("STOP!          ");
    digitalWrite(RELAY_PIN, LOW);
    sosDetected = false;  // vehicle already in danger zone
  }
  else if (sosDetected) {
    lcd.print("OBJECT AHEAD!  ");
    digitalWrite(RELAY_PIN, HIGH);
  }
  else {
    lcd.print("CLEAR AHEAD    ");
    digitalWrite(RELAY_PIN, HIGH);
  }

  delay(40);
}
