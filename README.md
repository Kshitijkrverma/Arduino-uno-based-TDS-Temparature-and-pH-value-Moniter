# Arduino-uno-based-TDS-Temparature-and-pH-value-Moniter
This is a section of code for a project which is based on the hydroponic way of growing system 

#include <OneWire.h>
#include <LiquidCrystal.h>

LiquidCrystal lcd(2, 3, 4, 5, 6, 7);

#define SensorPin A0
#define TDS_Pin A1 // Pin used for TDS sensor

#define ONE_WIRE_BUS 8
OneWire oneWire(ONE_WIRE_BUS);
float temperature = 0;

float calibration_value = 21.34 + 0.7;
unsigned long int avgValue;
int buf[10], temp;
bool showTemperature = true;
bool showTDS = false;
bool showPh = false;

const int backlightPin = 10; // Backlight control pin

void setup() {
  pinMode(SensorPin, INPUT);
  pinMode(TDS_Pin, INPUT);

  lcd.begin(16, 2);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Welcome To");
  lcd.setCursor(0, 1);
  lcd.print("pH, Temp & TDS");
  delay(2000);
  lcd.clear();

  pinMode(backlightPin, OUTPUT);
  digitalWrite(backlightPin, HIGH); // Turn on the backlight
}

void loop() {
  if (showTemperature) {
    displayTemperature();
  } else if (showTDS) {
    displayTDS();
  } else if (showPh) {
    displayPh();
  }

  delay(2000); // Show each reading for 2 seconds
}

void displayTemperature() {
  // Reading temperature from DS18B20 sensor
  byte data[12];
  oneWire.reset();
  oneWire.skip();
  oneWire.write(0x44); // start conversion, with parasite power on at the end
  delay(2000);         // maybe 750ms is enough, maybe not
  oneWire.reset();
  oneWire.skip();
  oneWire.write(0xBE); // Read Scratchpad

  for (byte i = 0; i < 9; i++) { // we need 9 bytes
    data[i] = oneWire.read();
  }

  int16_t raw = (data[1] << 8) | data[0];
  byte cfg = (data[4] & 0x60);
  if (cfg == 0x00) raw = raw & ~7; // 9 bit resolution, 93.75 ms
  else if (cfg == 0x20) raw = raw & ~3; // 10 bit res, 187.5 ms
  else if (cfg == 0x40) raw = raw & ~1; // 11 bit res, 375 ms

  temperature = (float)raw / 16.0;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temperature);
  lcd.print(" C");

  showTemperature = false;
  showTDS = true;
}

void displayTDS() {
  int sensorValue = analogRead(TDS_Pin);
  float tdsValue = map(sensorValue, 0, 1023, 0, 5000); // Assuming the sensor reads up to 5000 ppm

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("TDS: ");
  lcd.print(tdsValue);
  lcd.print(" ppm");

  showTDS = false;
  showPh = true;
}

void displayPh() {
  // Reading and smoothing sensor values for pH
  for (int i = 0; i < 10; i++) {
    buf[i] = analogRead(SensorPin);
    delay(10);
  }
  for (int i = 0; i < 9; i++) {
    for (int j = i + 1; j < 10; j++) {
      if (buf[i] > buf[j]) {
        temp = buf[i];
        buf[i] = buf[j];
        buf[j] = temp;
      }
    }
  }
  avgValue = 0;
  for (int i = 2; i < 8; i++) avgValue += buf[i];

  float phValue = (float)avgValue * 5.0 / 1024 / 6;
  phValue = -5.70 * phValue + calibration_value;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("pH: ");
  lcd.print(phValue);

  lcd.setCursor(0, 1);
  lcd.print(" ");

  if (phValue < 4) {
    lcd.print("Very Acidic");
  } else if (phValue >= 4 && phValue < 5) {
    lcd.print("Acidic");
  } else if (phValue >= 5 && phValue < 7) {
    lcd.print("Mildly Acidic");
  } else if (phValue >= 7 && phValue < 8) {
    lcd.print("Neutral");
  } else if (phValue >= 8 && phValue < 10) {
    lcd.print("Mildly Alkaline");
  } else if (phValue >= 10 && phValue < 11) {
    lcd.print("Alkaline");
  } else if (phValue >= 11) {
    lcd.print("Very Alkaline");
  }

  showPh = false;
  showTemperature = true;
}
