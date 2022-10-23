# Smart_Protection_System_for_Home_Appliances
Under and over voltage smart protection system for home appliances.

&nbsp;

![image_1]()

&nbsp;

![image_2]()

&nbsp;

![image_3]()

&nbsp;

![image_4]()

&nbsp;

![image_5]()

&nbsp;

firmware.ino

<!-- &nbsp; -->

```c
//for Voltage Sensor
#include "EmonLib.h"

// for Current Sensor
#include "ACS712.h"

// for Timer & Buzzer
#include <NoDelay.h>

// for Display
#include <LiquidCrystal.h>

// for Voltage Sensor
#define voltageSenPin A1

// Current Sensor
#define currentSenPin A2

// for Test Mode
#define testButton 2
#define testIndi 3
#define testPot A5

// for Relay Switch
#define relayPin 12

// for Buzzer
#define buzzerPin 13
#define buzzDisButtPin A4

// for Indicator
#define redLED 5
#define greLED 1
#define orgLED 4

// for Voltage Sensor
EnergyMonitor emon1;

// for Current Sensor
ACS712  ACS(currentSenPin, 5.0, 1023, 66);

// for Timer
noDelay timeInt(1000);

// for Buzzer
noDelay normalTime(1000);
noDelay UorOTime(50);

// for Display
LiquidCrystal lcd(11, 10, 9, 8, 7, 6);

// for System Data
String sysStatus, sysComment;
float sysVoltage, sysCurrent;

// for Test Mode
bool testMode, pTestMode, testModeSen;
unsigned int testVoltage;

// for Timer
unsigned int timerTime;
bool timerOn;

// for Buzzer
bool buzzerEnable = 1;
bool outForNormal;
unsigned int count = 1;
bool buzzerDisable, pBuzzerDisable, buzzerDisableSen;

void setup() {
  // for Voltage Sensor
  emon1.voltage(voltageSenPin, 255, 1.7);
  
  // for Test Mode
  pinMode(testButton, INPUT);
  pinMode(testIndi, OUTPUT);

  // for Relay Switch
  pinMode(relayPin, OUTPUT);
  setOut(LOW);

  // for Buzzer
  pinMode(buzzerPin, OUTPUT);
  pinMode(buzzDisButtPin, INPUT);

  // for Indicator
  pinMode(orgLED, OUTPUT);
  pinMode(greLED, OUTPUT);
  pinMode(redLED, OUTPUT);

  // for Display
  lcd.begin(20, 4);
  
}

void loop() {
  checkTestMode();
  runTimer();
  checkBuzzDis();
  
  if(testMode == HIGH) {
    sysVoltage = testVoltage;
  } else if(testMode == LOW) {
    sysVoltage = getVoltage();
    sysCurrent = getCurrent();
  }
  
  if(sysVoltage >= 216 && sysVoltage <= 244 && timerTime == 0) {
    sysStatus = "Normal";
    sysComment = "Working properly...";
    setOut(HIGH);
    accessBazzer("NORMAL");
    buzzerDisable = 0;
    accessIndi("NORMAL");
    
  } else if(sysVoltage < 216) {
    setOut(LOW);
    if(timerOn == false) {
      OnTimer(9);
    }
    sysStatus = "Under";
    sysComment = "Retry: " + String(timerTime);
    accessBazzer("UorO");
    accessIndi("UNDER");
    
  } else if(sysVoltage > 244) {
    setOut(LOW);
    if(timerOn == false) {
      OnTimer(9);
    }
    sysStatus = "Over";
    sysComment = "Retry: " + String(timerTime);
    accessBazzer("UorO");
    accessIndi("OVER");
    
  } else {
    sysComment = "Retry: " + String(timerTime);
    accessBazzer("UorO");
  }

  if(testMode == HIGH) {
    displayTheData("System stat: " + sysStatus, "Voltage: " + String(sysVoltage) + "V",
        "Current: ", sysComment);
  } else {
    displayTheData("System stat: " + sysStatus, "Voltage: " + String(sysVoltage) + "V",
      "Current: " + String(sysCurrent) + "A", sysComment);
  }
  
  delay(250);
}

float getVoltage() {
  emon1.calcVI(20, 2000);
  float supplyVoltage   = emon1.Vrms;
  return supplyVoltage;
  
}

float getCurrent() {
  float current = ACS.mA_AC();
  current = current / 1000.0;
  return current;
  
}

void runTimer() {
  if(timeInt.update() && timerOn == true && timerTime != 0) {
    timerTime = timerTime - 1;
  }
  if(timerTime == 0) {
    timerOn = false;
  }
}

void OnTimer(unsigned int tim) {
  timerTime = tim;
  timerOn = true;
}

void checkTestMode() {
  if(digitalRead(testButton) == HIGH && testModeSen == 0) {
    testMode = ! pTestMode;
    digitalWrite(testIndi, testMode);
    pTestMode = testMode;
    testModeSen = 1;
    
  } else if(digitalRead(testButton) == LOW && testModeSen == 1) {
    testModeSen = 0;
    
  }
  testVoltage = analogRead(testPot);
  testVoltage = map(testVoltage, 0, 1023, 202, 258);
  delay(1);
}

void setOut(bool out) {
  out = ! out;
  digitalWrite(relayPin, out);
}

void accessBazzer(String string) {
  if(normalTime.update() && string == "NORMAL" && buzzerEnable == 1) {
    outForNormal = !outForNormal;
    digitalWrite(buzzerPin, outForNormal);
    buzzerEnable = outForNormal;
    
  } else if(UorOTime.update() && string == "UorO") {
    if(count == 1 && buzzerDisable == 0) {
      digitalWrite(buzzerPin, HIGH);
      
    } else {
      digitalWrite(buzzerPin, LOW);
      
    }
    count++;
    if(count == 7) count = 1;
    buzzerEnable = 1;
    
  }
}

void checkBuzzDis() {
  if(digitalRead(buzzDisButtPin) == LOW && buzzerDisableSen == 0) {
    buzzerDisable = !pBuzzerDisable;
    pBuzzerDisable = buzzerDisable;
    buzzerDisableSen = 1;
    
  } else if(digitalRead(buzzDisButtPin) == HIGH && buzzerDisableSen == 1) {
    buzzerDisableSen = 0;
    
  }
}

void accessIndi(String string) {
  if(string == "UNDER") {
    digitalWrite(orgLED, HIGH);
    digitalWrite(greLED, LOW);
    digitalWrite(redLED, LOW);
    
  } else if(string == "NORMAL") {
    digitalWrite(orgLED, LOW);
    digitalWrite(greLED, HIGH);
    digitalWrite(redLED, LOW);
    
  } else if(string == "OVER") {
    digitalWrite(orgLED, LOW);
    digitalWrite(greLED, LOW);
    digitalWrite(redLED, HIGH);
    
  }
}

void displayTheData(String firstLine, String secondLine, String thiedLine, String fouthLine) {
  // lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(fillString(firstLine));
  lcd.setCursor(0, 1);
  lcd.print(fillString(secondLine));
  lcd.setCursor(0, 2);
  lcd.print(fillString(thiedLine));
  lcd.setCursor(0, 3);
  lcd.print(fillString(fouthLine));
}

String fillString(String string) {
  unsigned int lengt;
  lengt = string.length();
  lengt = 20 - lengt;
  for(int i = 1; i < lengt; i++) {
    string = string + " ";
  }
  return string;
  
}
```

&nbsp;
