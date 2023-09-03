# termostato_con_relevancia
termostato con relevancia, detector de Co2 y analizador de particulas

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>
#include <EEPROM.h>

const int dhtPin = 2;
const int mq7AO = A0;
const int boton1Pin = 5;
const int boton2Pin = 6;
const int boton3Pin = 7;
const int boton7Pin = 8;
const int relePin1 = 8;
const int relePin2 = 9;
const int ledPin12 = 12;  // LED para AC2
const int ledPin13 = 13;  // LED para AC1

DHT dht(dhtPin, DHT11);
LiquidCrystal_I2C lcd(0x27, 20, 4);

enum Pantalla {
  PANTALLA1,
  PANTALLA2,
  PANTALLA3,
  PANTALLA4,
  PANTALLA5
};

Pantalla pantallaActual = PANTALLA1;
float temperaturaObjetivo = 21.0;
float relevanciaObjetivo = 1;
unsigned long tiempoUltimoCambio = 0;
unsigned long tiempoDebounce = 200;
unsigned long tiempoUltimaPulsacion = 0;
unsigned long tiempoDeRelevancia = 86400000;
bool seleccionAC1 = true;
const float histereisis = 1.0;
float temperaturaEmergencia = 30.0; // Temperatura de emergencia en grados Celsius

void setup() {
  pinMode(mq7AO, INPUT);
  pinMode(boton1Pin, INPUT_PULLUP);
  pinMode(boton2Pin, INPUT_PULLUP);
  pinMode(boton3Pin, INPUT_PULLUP);
  pinMode(boton7Pin, INPUT_PULLUP);
  pinMode(relePin1, OUTPUT);
  pinMode(relePin2, OUTPUT);
  pinMode(ledPin12, OUTPUT);  // LED para AC2
  pinMode(ledPin13, OUTPUT);  // LED para AC1

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Area de Acidos 1");

  dht.begin();

  EEPROM.get(0, seleccionAC1);
  int tempObjInt;
  EEPROM.get(sizeof(bool), tempObjInt);
  temperaturaObjetivo = static_cast<float>(tempObjInt) / 10.0;
  int relObjInt;
  EEPROM.get(sizeof(bool) + sizeof(int), relObjInt);
  relevanciaObjetivo = static_cast<float>(relObjInt) / 10.0;
}

void loop() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  int co2Value = analogRead(mq7AO);

  if (digitalRead(boton3Pin) == LOW) {
    cambiarPantalla();
    delay(200);
  }

  unsigned long tiempoActual = millis();

  switch (pantallaActual) {
    case PANTALLA1:
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Area de Acidos 1");
      lcd.setCursor(0, 1);
      lcd.print("Temp: ");
      lcd.print(temperature, 1);
      lcd.print(" C");
      lcd.setCursor(0, 2);
      lcd.print("Humedad: ");
      lcd.print(humidity, 1);
      lcd.print("%");
      lcd.setCursor(0, 3);
      lcd.print("CO2: ");
      lcd.print(co2Value);
      break;

    case PANTALLA2:
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Area de Acidos 1");
      lcd.setCursor(0, 1);
      lcd.print("PM2.5: "); // Agregar la medición de PM2.5
      lcd.setCursor(0, 2);
      lcd.print("PM1: "); // Agregar la medición de PM1
      lcd.setCursor(0, 3);
      lcd.print("PM10: "); // Agregar la medición de P
      break;

    case PANTALLA3:
      if (digitalRead(boton1Pin) == LOW && tiempoActual - tiempoUltimaPulsacion >= tiempoDebounce) {
        temperaturaObjetivo += 0.5;
        tiempoUltimaPulsacion = tiempoActual;
        if (temperaturaObjetivo > 30.0) {
          temperaturaObjetivo = 30.0;
        }
        int tempObjInt = static_cast<int>(temperaturaObjetivo * 10);
        EEPROM.put(sizeof(bool), tempObjInt);
      }

      if (digitalRead(boton2Pin) == LOW && tiempoActual - tiempoUltimaPulsacion >= tiempoDebounce) {
        temperaturaObjetivo -= 0.5;
        tiempoUltimaPulsacion = tiempoActual;
        if (temperaturaObjetivo < 15.0) {
          temperaturaObjetivo = 15.0;
        }
        int tempObjInt = static_cast<int>(temperaturaObjetivo * 10);
        EEPROM.put(sizeof(bool), tempObjInt);
      }

      if (millis() - tiempoUltimaPulsacion >= 30000 || digitalRead(boton7Pin) == LOW) {
        tiempoUltimoCambio = millis();
      }

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Area de Acidos 1");
      lcd.setCursor(0, 1);
      lcd.print("Temperatura Objetivo");
      lcd.setCursor(0, 2);
      lcd.print(temperaturaObjetivo, 1);
      lcd.print(" C");
      break;

case PANTALLA4:
      if (digitalRead(boton1Pin) == LOW) {
        seleccionAC1 = true;  // Establecer AC1 como la selección
        tiempoUltimaPulsacion = tiempoActual;
        // Cambiar el relé solo si está en la pantalla 4
        digitalWrite(relePin1, HIGH);
        digitalWrite(relePin2, LOW);
      }

      if (digitalRead(boton2Pin) == LOW) {
        seleccionAC1 = false;  // Establecer AC2 como la selección
        tiempoUltimaPulsacion = tiempoActual;
        // Cambiar el relé solo si está en la pantalla 4
        digitalWrite(relePin1, LOW);
        digitalWrite(relePin2, HIGH);
      }

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Area de Acidos 1");
      lcd.setCursor(0, 1);
      lcd.print("Selecciona el Aire ");
      lcd.setCursor(0, 2);
      lcd.print(seleccionAC1 ? "> AC1" : "  AC1");
      lcd.setCursor(0, 3);
      lcd.print(seleccionAC1 ? "  AC2" : "> AC2");
      break;

    case PANTALLA5:
      if (digitalRead(boton1Pin) == LOW && tiempoActual - tiempoUltimaPulsacion >= tiempoDebounce) {
        relevanciaObjetivo += 1;
        tiempoUltimaPulsacion = tiempoActual;
        if (relevanciaObjetivo > 15) {
          relevanciaObjetivo = 15;
        }
        int relObjInt = static_cast<int>(relevanciaObjetivo * 10);
        EEPROM.put(sizeof(bool) + sizeof(int), relObjInt);

        // Cambiar el relé solo si está en la pantalla 4
        if (pantallaActual == PANTALLA4) {
          if (seleccionAC1) {
            digitalWrite(relePin1, HIGH);
            digitalWrite(relePin2, LOW);
          } else {
            digitalWrite(relePin1, LOW);
            digitalWrite(relePin2, HIGH);
          }
        }
      }

      if (digitalRead(boton2Pin) == LOW && tiempoActual - tiempoUltimaPulsacion >= tiempoDebounce) {
        relevanciaObjetivo -= 1;
        tiempoUltimaPulsacion = tiempoActual;
        if (relevanciaObjetivo < 1) {
          relevanciaObjetivo = 1;
        }
        int relObjInt = static_cast<int>(relevanciaObjetivo * 10);
        EEPROM.put(sizeof(bool) + sizeof(int), relObjInt);

        // Cambiar el relé solo si está en la pantalla 4
        if (pantallaActual == PANTALLA4) {
          if (seleccionAC1) {
            digitalWrite(relePin1, HIGH);
            digitalWrite(relePin2, LOW);
          } else {
            digitalWrite(relePin1, LOW);
            digitalWrite(relePin2, HIGH);
          }
        }
      }

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Area de Acidos 1");
      lcd.setCursor(0, 1);
      lcd.print("Relevancia");
      lcd.setCursor(0, 2);
      lcd.print(relevanciaObjetivo);
      lcd.print(" Dias");
      break;
  }

  // Control de temperatura de emergencia y relés de refrigeración
  if (temperature >= temperaturaEmergencia) {
    digitalWrite(relePin1, HIGH); // Activar el rele AC1 para refrigerar
    digitalWrite(relePin2, HIGH); // Activar el rele AC2 para refrigerar
  } else {
    // Control de refrigeración según la variable de selección y la pantalla 4
    if (pantallaActual == PANTALLA4) {
      if (seleccionAC1) {
        if (temperature >= temperaturaObjetivo) {
          digitalWrite(relePin1, HIGH); // Activar el rele AC1 para refrigerar
          digitalWrite(relePin2, LOW);  // Desactivar el rele AC2
        } else {
          digitalWrite(relePin1, LOW);  // Desactivar el rele AC1
          digitalWrite(relePin2, LOW);  // Desactivar el rele AC2
        }
      } else {
        if (temperature >= temperaturaObjetivo) {
          digitalWrite(relePin1, LOW);  // Desactivar el rele AC1
          digitalWrite(relePin2, HIGH); // Activar el rele AC2 para refrigerar
        } else {
          digitalWrite(relePin1, LOW);  // Desactivar el rele AC1
          digitalWrite(relePin2, LOW);  // Desactivar el rele AC2
        }
      }
    }
  }

  // Control de LEDs basado en el estado de los relés
  digitalWrite(ledPin12, digitalRead(relePin2)); // LED12 refleja estado de AC2
  digitalWrite(ledPin13, digitalRead(relePin1)); // LED13 refleja estado de AC1

  delay(200);
}

void cambiarPantalla() {
  pantallaActual = static_cast<Pantalla>((static_cast<int>(pantallaActual) + 1) % 5);
  seleccionAC1 = !seleccionAC1;
}



