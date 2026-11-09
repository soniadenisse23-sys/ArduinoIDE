#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <max6675.h>
#include <math.h>


const byte LCD_ADDRESS = 0x27;
const byte LCD_COLUMNS = 16;
const byte LCD_ROWS = 2;

const unsigned long SERIAL_BAUD = 9600;

const bool SIMULATE_TEMP = true;

const bool ENABLE_OUTPUTS = true;

// Potenciometros
const byte POT_FAN = A0;
const byte POT_HEATER = A1;

// Boton de niebla
const byte BUTTON_MIST = 22;

// MAX6675
const byte PIN_SCK = 52;
const byte PIN_CS = 53;
const byte PIN_SO = 50;

// Salidas
const byte OUT_HEATER = 6;
const byte OUT_FAN = 7;
const byte OUT_MIST = 8;


LiquidCrystal_I2C lcd(
  LCD_ADDRESS,
  LCD_COLUMNS,
  LCD_ROWS
);

MAX6675 thermocouple(
  PIN_SCK,
  PIN_CS,
  PIN_SO
);

float temperature = NAN;

int fanPercent = 0;
int heaterPercent = 0;

bool mistState = false;

// Control manual desde computadora
bool serialFanControl = true;
bool serialHeaterControl = true;

byte screenPage = 0;

unsigned long lastLCD = 0;
unsigned long lastSerial = 0;

const unsigned long LCD_INTERVAL = 500;
const unsigned long SERIAL_INTERVAL = 1000;

// Boton
bool lastButtonReading = HIGH;
bool stableButtonState = HIGH;

unsigned long lastDebounceTime = 0;
const unsigned long DEBOUNCE_TIME = 50;

void setup() {

  Serial.begin(SERIAL_BAUD);

  lcd.init();
  lcd.backlight();

  pinMode(POT_FAN, INPUT);
  pinMode(POT_HEATER, INPUT);

  pinMode(BUTTON_MIST, INPUT_PULLUP);

  pinMode(OUT_HEATER, OUTPUT);
  pinMode(OUT_FAN, OUTPUT);
  pinMode(OUT_MIST, OUTPUT);

  digitalWrite(OUT_HEATER, LOW);
  digitalWrite(OUT_FAN, LOW);
  digitalWrite(OUT_MIST, LOW);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("AIR LAB SYSTEM");

  lcd.setCursor(0, 1);
  lcd.print("Iniciando...");

  delay(1500);

  lcd.clear();

  Serial.println();
  Serial.println("================================");
  Serial.println("       SISTEMA AIR LAB");
  Serial.println("================================");

  Serial.println("Arduino Mega 2560 conectado.");
  Serial.println("Sistema iniciado correctamente.");

  Serial.println(
    ENABLE_OUTPUTS ?
    "Salidas habilitadas." :
    "MODO SEGURO: salidas deshabilitadas."
  );

  Serial.println();
  printHelp();

}


void readTemperature() {

  if (SIMULATE_TEMP) {

    temperature = 25.0 +
      10.0 * sin(millis() / 5000.0);

  } else {

    temperature = thermocouple.readCelsius();

    if (isnan(temperature) ||
        temperature < 0 ||
        temperature > 1024) {

      temperature = NAN;

    }

  }

}


void readPotentiometers() {

  if (!serialFanControl) {

    int fanValue = analogRead(POT_FAN);

    fanPercent = map(
      fanValue, 0, 1023, 0, 100
    );

  }

  if (!serialHeaterControl) {

    int heaterValue = analogRead(POT_HEATER);

    heaterPercent = map(
      heaterValue, 0, 1023, 0, 100
    );

  }

}

void readMistButton() {

  bool reading = digitalRead(BUTTON_MIST);

  if (reading != lastButtonReading) {

    lastDebounceTime = millis();

  }

  if (millis() - lastDebounceTime > DEBOUNCE_TIME) {

    if (reading != stableButtonState) {

      stableButtonState = reading;

      if (stableButtonState == LOW) {

        mistState = !mistState;

        Serial.println(
          mistState ?
          "Niebla activada por boton." :
          "Niebla desactivada por boton."
        );

      }

    }

  }

  lastButtonReading = reading;

}



void controlOutputs() {

  if (!ENABLE_OUTPUTS) {

    digitalWrite(OUT_HEATER, LOW);
    digitalWrite(OUT_FAN, LOW);
    digitalWrite(OUT_MIST, LOW);

    return;

  }

  int fanPWM = map(
    fanPercent, 0, 100, 0, 255
  );

  int heaterPWM = map(
    heaterPercent, 0, 100, 0, 255
  );

  analogWrite(OUT_FAN, fanPWM);
  analogWrite(OUT_HEATER, heaterPWM);

  digitalWrite(
    OUT_MIST,
    mistState ? HIGH : LOW
  );

}


void printStatus() {

  Serial.println();
  Serial.println("========== AIR LAB ==========");

  Serial.print("Temperatura: ");

  if (isnan(temperature)) {

    Serial.println("ERROR DE SENSOR");

  } else {

    Serial.print(temperature, 2);
    Serial.println(" C");

  }

  Serial.print("Ventilador: ");
  Serial.print(fanPercent);
  Serial.println("%");

  Serial.print("Calentador: ");
  Serial.print(heaterPercent);
  Serial.println("%");

  Serial.print("Generador de niebla: ");
  Serial.println(mistState ? "ON" : "OFF");

  Serial.print("Salidas fisicas: ");
  Serial.println(
    ENABLE_OUTPUTS ? "HABILITADAS" : "DESHABILITADAS"
  );

  Serial.println("=============================");
  Serial.println();

}


void printHelp() {

  Serial.println("COMANDOS DISPONIBLES:");
  Serial.println();

  Serial.println("TEMP       - Consultar temperatura");
  Serial.println("STATUS     - Estado del sistema");

  Serial.println("FAN 0-100  - Controlar ventilador");
  Serial.println("HEATER 0-100 - Controlar calentador");

  Serial.println("MIST ON    - Activar niebla");
  Serial.println("MIST OFF   - Desactivar niebla");

  Serial.println("POT FAN    - Control ventilador por potenciometro");
  Serial.println("POT HEATER - Control calentador por potenciometro");

  Serial.println("HELP       - Mostrar comandos");

  Serial.println();
  Serial.println("Escribe un comando y presiona ENTER.");
  Serial.println();

}

void processCommand(String command) {

  command.trim();
  command.toUpperCase();

  if (command.length() == 0) {
    return;
  }

  if (command == "HELP") {

    printHelp();
    return;

  }

  if (command == "TEMP") {

    readTemperature();

    if (isnan(temperature)) {

      Serial.println("ERROR: revisar MAX6675.");

    } else {

      Serial.print("Temperatura: ");
      Serial.print(temperature, 2);
      Serial.println(" C");

    }

    return;

  }

  if (command == "STATUS") {

    printStatus();
    return;

  }

  if (command.startsWith("FAN ")) {

    int value = command.substring(4).toInt();

    if (value < 0 || value > 100) {

      Serial.println("Error: utiliza un valor de 0 a 100.");
      return;

    }

    fanPercent = value;
    serialFanControl = true;

    Serial.print("Ventilador configurado a: ");
    Serial.print(fanPercent);
    Serial.println("%");

    return;

  }

  if (command.startsWith("HEATER ")) {

    int value = command.substring(7).toInt();

    if (value < 0 || value > 100) {

      Serial.println("Error: utiliza un valor de 0 a 100.");
      return;

    }

    heaterPercent = value;
    serialHeaterControl = true;

    Serial.print("Calentador configurado a: ");
    Serial.print(heaterPercent);
    Serial.println("%");

    return;

  }

  if (command == "MIST ON") {

    mistState = true;

    Serial.println("Generador de niebla: ON");
    return;

  }

  if (command == "MIST OFF") {

    mistState = false;

    Serial.println("Generador de niebla: OFF");
    return;

  }

  if (command == "POT FAN") {

    serialFanControl = false;

    Serial.println("Ventilador controlado por potenciometro.");

    return;

  }

  if (command == "POT HEATER") {

    serialHeaterControl = false;

    Serial.println("Calentador controlado por potenciometro.");

    return;

  }

  Serial.println("Comando no reconocido. Escribe HELP.");

}


void readSerialCommands() {

  if (Serial.available() > 0) {

    String command = Serial.readStringUntil('\n');

    processCommand(command);

  }

}

void updateLCD() {

  lcd.clear();

  if (screenPage == 0) {

    lcd.setCursor(0, 0);
    lcd.print("Temp:");

    if (isnan(temperature)) {

      lcd.print(" ERROR");

    } else {

      lcd.print(temperature, 1);
      lcd.print((char)223);
      lcd.print("C");

    }

    lcd.setCursor(0, 1);

    lcd.print("Vent:");
    lcd.print(fanPercent);
    lcd.print("%");

  }

  else {

    lcd.setCursor(0, 0);

    lcd.print("Calent:");
    lcd.print(heaterPercent);
    lcd.print("%");

    lcd.setCursor(0, 1);

    lcd.print("Niebla:");
    lcd.print(mistState ? "ON" : "OFF");

  }

  screenPage = !screenPage;

}


void sendSerialData() {

  Serial.print("Temperatura:");

  if (isnan(temperature)) {

    Serial.print("ERROR");

  } else {

    Serial.print(temperature, 2);

  }

  Serial.print(" C | Ventilador:");
  Serial.print(fanPercent);
  Serial.print("% | Calentador:");
  Serial.print(heaterPercent);
  Serial.print("% | Niebla:");

  Serial.print(mistState ? "ON" : "OFF");

  Serial.print(" | Salidas:");

  Serial.println(
    ENABLE_OUTPUTS ? "HABILITADAS" : "DESHABILITADAS"
  );

}

void loop() {

  readSerialCommands();

  readPotentiometers();

  readTemperature();

  readMistButton();

  controlOutputs();

  if (millis() - lastLCD >= LCD_INTERVAL) {

    lastLCD = millis();

    updateLCD();

  }

  if (millis() - lastSerial >= SERIAL_INTERVAL) {

    lastSerial = millis();

    sendSerialData();

  }

}
