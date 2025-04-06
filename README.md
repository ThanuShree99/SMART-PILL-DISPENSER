#SMART_PILL_DISPENSER
// ==========================
//    Include Libraries
// ==========================
#include <Wire.h>
#include <RTClib.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

// ==========================
//    Initialize Objects
// ==========================
RTC_DS3231 rtc;
Servo servo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ==========================
//    Define Pins
// ==========================
const int servoPin = 9;
const int buzzerPin = 13;

const int hourUpPin = A0;
const int hourDownPin = A1;
const int minuteUpPin = A2;
const int minuteDownPin = A3;
const int setPin = 6;
const int modePin = 7;

// ==========================
//    Define Variables
// ==========================
const int maxModes = 3;
int currentMode = 0;
bool setMode = false;

// Pill log tracking
bool pillTakenToday[maxModes] = {false, false, false};

// ==========================
//    Define Structures
// ==========================
struct Mode {
  int hour;
  int minute;
  int servoAngle;
};

// ==========================
//    Initialize Modes
// ==========================
Mode modes[maxModes] = {
  {8, 0, 45},
  {12, 0, 90},
  {18, 0, 135}
};

// ==========================
//        Setup Function
// ==========================
void setup() {
  servo.attach(servoPin);
  pinMode(buzzerPin, OUTPUT);
  pinMode(hourUpPin, INPUT_PULLUP);
  pinMode(hourDownPin, INPUT_PULLUP);
  pinMode(minuteUpPin, INPUT_PULLUP);
  pinMode(minuteDownPin, INPUT_PULLUP);
  pinMode(setPin, INPUT_PULLUP);
  pinMode(modePin, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Pill Dispenser");

  if (!rtc.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("RTC not found!");
    while (1);
  }

  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }

  delay(2000);
  lcd.clear();
}

// ==========================
//         Main Loop
// ==========================
void loop() {
  handleSetMode();

  if (!setMode) {  // Normal Operation
    DateTime now = rtc.now();

    lcd.setCursor(0, 0);
    lcd.print("Time: ");
    lcd.print(now.hour());
    lcd.print(":");
    if (now.minute() < 10) lcd.print("0");
    lcd.print(now.minute());
    lcd.print(":");
    if (now.second() < 10) lcd.print("0");
    lcd.print(now.second());

    lcd.setCursor(0, 1);
    lcd.print("Mode ");
    lcd.print(currentMode + 1);
    lcd.print(": ");
    lcd.print(modes[currentMode].hour);
    lcd.print(":");
    if (modes[currentMode].minute < 10) lcd.print("0");
    lcd.print(modes[currentMode].minute);

    checkMissedDosage(now);

    if (now.hour() == modes[currentMode].hour &&
        now.minute() == modes[currentMode].minute &&
        now.second() == 0) {
      dispensePill(modes[currentMode].servoAngle, currentMode);
      delay(60000);  // Wait 1 minute to avoid multiple triggers
    }
  }
}

// ==========================
//     Handle Set Mode
// ==========================
void handleSetMode() {
  if (digitalRead(setPin) == LOW) {
    delay(200);  // Debounce
    setMode = !setMode; // Toggle set mode
    lcd.clear();
  }

  if (setMode) {
    if (digitalRead(modePin) == LOW) {
      currentMode = (currentMode + 1) % maxModes;
      delay(200);
    }

    if (digitalRead(hourUpPin) == LOW) {
      modes[currentMode].hour = (modes[currentMode].hour + 1) % 24;
      delay(200);
    }
    if (digitalRead(hourDownPin) == LOW) {
      modes[currentMode].hour = (modes[currentMode].hour - 1 + 24) % 24;
      delay(200);
    }
    if (digitalRead(minuteUpPin) == LOW) {
      modes[currentMode].minute = (modes[currentMode].minute + 1) % 60;
      delay(200);
    }
    if (digitalRead(minuteDownPin) == LOW) {
      modes[currentMode].minute = (modes[currentMode].minute - 1 + 60) % 60;
      delay(200);
    }

    lcd.setCursor(0, 0);
    lcd.print("Mode ");
    lcd.print(currentMode + 1);
    lcd.print(": Set Time");
    lcd.setCursor(0, 1);
    lcd.print(modes[currentMode].hour);
    lcd.print(":");
    if (modes[currentMode].minute < 10) lcd.print("0");
    lcd.print(modes[currentMode].minute);
  }
}

// ==========================
//     Dispense Pill
// ==========================
void dispensePill(int angle, int modeIndex) {
  servo.write(angle);
  delay(1000);
  servo.write(0);
  tone(buzzerPin, 1000, 1000);
  pillTakenToday[modeIndex] = true;  // Mark pill as taken
}

// ==========================
//     Check Missed Dosage
// ==========================
void checkMissedDosage(DateTime now) {
  static int lastCheckedDay = -1;

  // Reset log at midnight
  if (now.day() != lastCheckedDay) {
    for (int i = 0; i < maxModes; i++) {
      pillTakenToday[i] = false;
    }
    lastCheckedDay = now.day();
  }

  // Check if pill was missed (1 hour after scheduled time)
  for (int i = 0; i < maxModes; i++) {
    int scheduledTimeInMinutes = modes[i].hour * 60 + modes[i].minute;
    int currentTimeInMinutes = now.hour() * 60 + now.minute();

    if (!pillTakenToday[i] && (currentTimeInMinutes > scheduledTimeInMinutes + 60)) {
      // Pill missed
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Missed Dose!");
      lcd.setCursor(0, 1);
      lcd.print("Mode ");
      lcd.print(i + 1);

      tone(buzzerPin, 2000, 1000);  // Higher tone for missed dose
      delay(3000);
      lcd.clear()
    }
  }
}
