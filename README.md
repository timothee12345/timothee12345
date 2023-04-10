#include <HX711_ADC.h>
#include <SPI.h>
#include <SD.h>

#if defined(ESP8266) || defined(ESP32) || defined(AVR)
#include <EEPROM.h>
#endif

//pins:
const int HX711_dout = 4; //mcu > HX711 dout pin
const int HX711_sck = 5; //mcu > HX711 sck pin
const int chipSelect = 10; //SD card chip select pin
const int buttonPin = 2; // Définissez la broche du bouton

//HX711 constructor:
HX711_ADC LoadCell(HX711_dout, HX711_sck);

const int calVal_eepromAdress = 0;
unsigned long t = 0;
int folderCounter = 1;

File dataFile;

void setup() {
  Serial.begin(57600);
  delay(10);
  Serial.println();
  Serial.println("Starting...");

  // SD card initialization
  Serial.print("Initializing SD card...");
  if (!SD.begin(chipSelect)) {
    Serial.println("Card failed, or not present");
    while (1);
  }
  Serial.println("Card initialized.");

  pinMode(buttonPin, INPUT_PULLUP); // Configurez la broche du bouton en mode INPUT_PULLUP

  LoadCell.begin();
  float calibrationValue;
#if defined(ESP8266) || defined(ESP32)
  EEPROM.begin(512);
#endif
  EEPROM.get(calVal_eepromAdress, calibrationValue);

  unsigned long stabilizingtime = 2000;
  boolean _tare = true;
  LoadCell.start(stabilizingtime, _tare);
  if (LoadCell.getTareTimeoutFlag()) {
    Serial.println("Timeout, check MCU>HX711 wiring and pin designations");
    while (1);
  } else {
    LoadCell.setCalFactor(calibrationValue);
    Serial.println("Startup is complete");
  }
}

void loop() {
  static boolean newDataReady = 0;
  const int serialPrintInterval = 0;

  if (LoadCell.update()) newDataReady = true;

  if (newDataReady) {
    if (millis() > t + serialPrintInterval) {
      float i = LoadCell.getData();
      Serial.print("Load_cell output val: ");
      Serial.println(i);

      // Save data to SD card
      String folderName = "data" + String(folderCounter);
      SD.mkdir(folderName);
      dataFile = SD.open(folderName + "/data.txt", FILE_WRITE);
      if (dataFile) {
        dataFile.println(i);
        dataFile.close();
        Serial.println("Data saved to SD card.");
      } else {
        Serial.println("Error opening data.txt");
      }

      newDataReady = 0;
      t = millis();
    }
  }

  if (digitalRead(buttonPin) == LOW) { // Vérifiez si le bouton est enfoncé
    delay(50); // Anti-rebond
    if (digitalRead(buttonPin) == LOW) {
      folderCounter++; // Incrémentez le compteur de dossiers
      while (digitalRead(buttonPin) == LOW); // Attendez que le bouton soit relâché
    }
  }

  if (Serial.available() > 0) {
    char inByte = Serial.read();
    if (inByte == 't') LoadCell.tareNoDelay();
  }

  if (LoadCell.getTareStatus() == true) {
    Serial.println("Tare complete");
  }
}
