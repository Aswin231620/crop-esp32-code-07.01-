# crop-esp32-code-11.02

#include <Servo.h>

// ---------- PIN DEFINITIONS ----------
#define TRIG_PIN    7
#define ECHO_PIN    6
#define BUZZER_PIN  5
#define LED_PIN     8
#define SERVO_PIN   9

// ---------- SETTINGS ----------
#define DISTANCE_THRESHOLD 15   // cm
#define ULTRA_ACTIVE_TIME  5000 // 5 seconds
#define CAMERA_TIMEOUT     3000 // ms

Servo deterrentServo;

// ---------- STATE ----------
bool cameraDetected = false;
unsigned long lastCameraTime = 0;

bool ultraActive = false;
unsigned long ultraStartTime = 0;

void setup() {
  Serial.begin(9600);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  deterrentServo.attach(SERVO_PIN);
  deterrentServo.write(0);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);

  Serial.println("CropGuardian Ready");
}

// ---------- ULTRASONIC FUNCTION ----------
long getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return 999;

  return duration * 0.034 / 2;
}

void loop() {

  // ---------- CAMERA INPUT ----------
  if (Serial.available()) {
    String cmd = Serial.readStringUntil('\n');
    cmd.trim();

    if (cmd == "ALERT") {
      cameraDetected = true;
      lastCameraTime = millis();
    } 
    else if (cmd == "CLEAR") {
      cameraDetected = false;
    }
  }

  // Auto-clear camera flag
  if (cameraDetected && millis() - lastCameraTime > CAMERA_TIMEOUT) {
    cameraDetected = false;
  }

  // ---------- ULTRASONIC ----------
  long distance = getDistance();

  // Trigger ultrasonic ONCE
  if (!ultraActive && distance <= DISTANCE_THRESHOLD) {
    ultraActive = true;
    ultraStartTime = millis();
  }

  // Auto stop after 5 seconds
  if (ultraActive && millis() - ultraStartTime >= ULTRA_ACTIVE_TIME) {
    ultraActive = false;
  }

  // ---------- OR LOGIC ----------
  bool activate = cameraDetected || ultraActive;

  if (activate) {
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(LED_PIN, HIGH);
    deterrentServo.write(90);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(LED_PIN, LOW);
    deterrentServo.write(0);
  }

  // Send distance to Python / Firebase
  Serial.println(distance);

  delay(200);
}
