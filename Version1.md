//# Intelligente-house
//Intelligente House with Esp32
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <LiquidCrystal_I2C.h>
#include "DHT.h"
#include <SPI.h>
#include <MFRC522.h>
#include <ESP32Servo.h>
#include <WiFi.h>
#include <WebServer.h>

// === Crédentials AP (Hotspot de l’ESP32) ===
const char* ssid = "ESP32_Hotspot";
const char* password = "12345678";

// === Création serveur Web sur port 80 ===
WebServer server(80);

// === LCD ===
LiquidCrystal_I2C lcd(0x27, 16, 2);  


// === DHT11 ===
#define DHTPIN 33
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// === Infrarouge ===
#define IR_PIN 34
#define SERVO_IR_PIN 14
Servo servoIR;

// === Capteurs / Actionneurs ===
const int mq2Pin    = 32;
const int relaisPin = 25;
const int buzzerPin = 27;
const int pirPin    = 12;  // ⚠ changé pour éviter conflit avec IR sur 27

// === RFID RC522 ===
#define SS_PIN 5
#define RST_PIN 4
MFRC522 mfrc522(SS_PIN, RST_PIN);

// === Servo RFID ===
Servo servoRFID;

// === Servo et relais via WiFi ===
Servo servoWiFi;
const int RELAY_WIFI_PIN = 23; // relais piloté via page web

// === Seuils / UID autorisé ===
int seuil = 1000; // calibrer
byte badgeAutorise[] = {0xEF, 0x6C, 0xE9, 0x1E};

// ==== Vérif UID ====
bool verifierBadge(byte *uid, byte taille) {
  if (taille != sizeof(badgeAutorise)) return false;
  for (byte i = 0; i < taille; i++) {
    if (uid[i] != badgeAutorise[i]) return false;
  }
  return true;
}

// === Pages Web ===
void handleRoot() {
  String html = "<!DOCTYPE html><html><head><title>ESP32 Control</title></head><body>";
  html += "<h1>Contrôle Servo + Relais</h1>";
  html += "<p><a href=\"/servo/on\"><button>Servo2 ON</button></a></p>";
  html += "<p><a href=\"/servo/off\"><button>Servo2 OFF</button></a></p>";
  html += "<p><a href=\"/relais/on\"><button>Relais ON</button></a></p>";
  html += "<p><a href=\"/relais/off\"><button>Relais OFF</button></a></p>";
  html += "</body></html>";
  server.send(200, "text/html", html);
}
void handleServoOn() {
  servoWiFi.write(90);
  server.sendHeader("Location", "/"); server.send(303);
}
void handleServoOff() {
  servoWiFi.write(0);
  server.sendHeader("Location", "/"); server.send(303);
}
void handleRelaisOn() {
  digitalWrite(RELAY_WIFI_PIN, HIGH);
  server.sendHeader("Location", "/"); server.send(303);
}
void handleRelaisOff() {
  digitalWrite(RELAY_WIFI_PIN, LOW);
  server.sendHeader("Location", "/"); server.send(303);
}

// === Setup ===
void setup() {
  Serial.begin(115200);

  // Capteurs
  pinMode(relaisPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  pinMode(pirPin,    INPUT);
  pinMode(IR_PIN,    INPUT);

  digitalWrite(relaisPin, LOW);
  digitalWrite(buzzerPin, LOW);

  dht.begin();

  // Servos
  servoIR.attach(SERVO_IR_PIN);
  servoIR.write(0);
  servoRFID.attach(13);   // servo RFID
  servoRFID.write(0);
  servoWiFi.attach(15);   // servo commandé par WiFi
  servoWiFi.write(0);
  pinMode(RELAY_WIFI_PIN, OUTPUT);
  digitalWrite(RELAY_WIFI_PIN, LOW);

  // SPI + RC522
  SPI.begin();
  mfrc522.PCD_Init();

  // OLED

   lcd.init();           // Initialisation
   lcd.backlight();      // Activer le rétroéclairage
   lcd.setCursor(0,0);
   lcd.print("Systeme Incendie");
   delay(1500);
   lcd.clear();


  // WiFi Hotspot
  WiFi.softAP(ssid, password);
  Serial.print("Hotspot actif : "); Serial.println(WiFi.softAPIP());
  server.on("/", handleRoot);
  server.on("/servo/on", handleServoOn);
  server.on("/servo/off", handleServoOff);
  server.on("/relais/on", handleRelaisOn);
  server.on("/relais/off", handleRelaisOff);
  server.begin();
}

// === Loop ===
void loop() {
  server.handleClient();

  // Lectures capteurs
  int   valeurMQ2   = analogRead(mq2Pin);
  int   mouvement   = digitalRead(pirPin);
  float humidite    = dht.readHumidity();
  float temperature = dht.readTemperature();

  // RFID
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    if (verifierBadge(mfrc522.uid.uidByte, mfrc522.uid.size)) {
      Serial.println("Badge autorise ✅ : Ouverture porte");
      servoRFID.write(0); delay(2000); servoRFID.write(90);
    } else {
      Serial.println("Acces refuse ❌ !");
    }
    mfrc522.PICC_HaltA();
    mfrc522.PCD_StopCrypto1();
  }

  // Capteur IR
  if (digitalRead(IR_PIN) == LOW) {
    Serial.println("Obstacle detecte → Servo IR");
    servoIR.write(90); delay(2000); servoIR.write(0);
  }

  // Détection feu
  if (valeurMQ2 > seuil) {
    Serial.println("🔥 Feu detecte !");
    digitalWrite(relaisPin, HIGH);
    digitalWrite(buzzerPin, HIGH);
    lcd.setCursor(0,0); 
    delay(5000);
    digitalWrite(relaisPin, LOW);
    digitalWrite(buzzerPin, LOW);
  }

  // LCD affichage
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("T:");
  lcd.print(temperature);
  lcd.print("C H:");
  lcd.print(humidite);

  lcd.setCursor(0,1);
  lcd.print("PIR:");
  lcd.print(mouvement ? "OUI" : "NON");
  lcd.print(" MQ-2:");
  lcd.print(valeurMQ2);
  delay(500);
}
