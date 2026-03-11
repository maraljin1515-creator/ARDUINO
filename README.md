// =====================================================
// SIMPLE WHITE LINE FOLLOWER + SEARCH LINE
// Based on your motorA() / motorB() code
// white = 1, black = 0
// Sensor order: LEFT -> RIGHT = A4 A3 A2 A1 A0
// =====================================================

// ---------------- L298N pins ----------------
const int ENA = 5;   // PWM
const int IN1 = 8;
const int IN2 = 9;

const int ENB = 6;   // PWM
const int IN3 = 10;
const int IN4 = 11;

// ---------------- Sensor pins ----------------
// LEFT -> RIGHT
const byte sensorPins[5] = {A4, A3, A2, A1, A0};

// ---------------- Settings ----------------
const int THRESHOLD = 500;   // black ~10, white ~980+

// Straight speed
int speedA_straight = 140;   // motorA
int speedB_straight = 150;   // motorB

// Soft turn speed
int speedA_turn = 110;
int speedB_turn = 120;

// Search speed
int searchA = 110;
int searchB = 120;

// ---------------- Variables ----------------
int sensorRaw[5];
int white[5];   // white=1, black=0

int lastDirection = 0;
// -1 = last line was on left
//  1 = last line was on right
//  0 = center / unknown

// =====================================================

void setup() {
  Serial.begin(9600);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  pinMode(ENB, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  for (int i = 0; i < 5; i++) {
    pinMode(sensorPins[i], INPUT);
  }

  stopMotors();
}

// =====================================================

void loop() {
  readSensors();
  printPattern();

  // 00100 -> straight
  if (isPattern(0, 0, 1, 0, 0)) {
    lastDirection = 0;
    forwardStraight();
  }

  // Slight left: 01000 or 01100
  else if (isPattern(0, 1, 0, 0, 0) || isPattern(0, 1, 1, 0, 0)) {
    lastDirection = -1;
    turnLeftSoft();
  }

  // Hard left: 10000 or 11000
  else if (isPattern(1, 0, 0, 0, 0) || isPattern(1, 1, 0, 0, 0)) {
    lastDirection = -1;
    turnLeftHard();
  }

  // Slight right: 00010 or 00110
  else if (isPattern(0, 0, 0, 1, 0) || isPattern(0, 0, 1, 1, 0)) {
    lastDirection = 1;
    turnRightSoft();
  }

  // Hard right: 00001 or 00011
  else if (isPattern(0, 0, 0, 0, 1) || isPattern(0, 0, 0, 1, 1)) {
    lastDirection = 1;
    turnRightHard();
  }

  // Intersection for now
  else if (isPattern(1, 1, 1, 1, 1)) {
    forwardStraight();
  }

  // 00000 -> do not stop, search line
  else if (isPattern(0, 0, 0, 0, 0)) {
    searchLine();
  }

  // Other uncommon patterns
  else {
    searchLine();
  }

  delay(20);
}

// =====================================================
// SENSOR
// =====================================================

void readSensors() {
  for (int i = 0; i < 5; i++) {
    sensorRaw[i] = analogRead(sensorPins[i]);

    if (sensorRaw[i] > THRESHOLD) {
      white[i] = 1;   // white
    } else {
      white[i] = 0;   // black
    }
  }
}

bool isPattern(int s0, int s1, int s2, int s3, int s4) {
  return (white[0] == s0 &&
          white[1] == s1 &&
          white[2] == s2 &&
          white[3] == s3 &&
          white[4] == s4);
}

// =====================================================
// MOTOR
// =====================================================

void motorA(int speedVal, bool forward) {
  digitalWrite(IN1, forward ? HIGH : LOW);
  digitalWrite(IN2, forward ? LOW  : HIGH);
  analogWrite(ENA, speedVal);
}

void motorB(int speedVal, bool forward) {
  digitalWrite(IN3, forward ? HIGH : LOW);
  digitalWrite(IN4, forward ? LOW  : HIGH);
  analogWrite(ENB, speedVal);
}

void stopMotors() {
  motorA(0, true);
  motorB(0, true);
}

// =====================================================
// MOVEMENTS
// =====================================================

void forwardStraight() {
  motorA(speedA_straight, true);
  motorB(speedB_straight, true);
}

// line is on left -> turn left softly
void turnLeftSoft() {
  motorA(speedA_straight, true);
  motorB(speedB_turn, true);
}

// stronger left correction
void turnLeftHard() {
  motorA(speedA_straight, true);
  motorB(0, true);
}

// line is on right -> turn right softly
void turnRightSoft() {
  motorA(speedA_turn, true);
  motorB(speedB_straight, true);
}

// stronger right correction
void turnRightHard() {
  motorA(0, true);
  motorB(speedB_straight, true);
}

// when 00000: search where the line was last seen
void searchLine() {
  if (lastDirection == -1) {
    // search left
    motorA(searchA, true);
    motorB(0, true);
  }
  else if (lastDirection == 1) {
    // search right
    motorA(0, true);
    motorB(searchB, true);
  }
  else {
    // unknown -> slight right search
    motorA(0, true);
    motorB(searchB, true);
  }
}

// =====================================================
// DEBUG
// =====================================================

void printPattern() {
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint < 100) return;
  lastPrint = millis();

  Serial.print("Raw: ");
  for (int i = 0; i < 5; i++) {
    Serial.print(sensorRaw[i]);
    Serial.print(" ");
  }

  Serial.print("| Pattern: ");
  for (int i = 0; i < 5; i++) {
    Serial.print(white[i]);
  }

  Serial.print("| lastDir: ");
  Serial.println(lastDirection);
}
