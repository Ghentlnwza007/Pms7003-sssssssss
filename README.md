#include <WiFi.h>
#include <ThingSpeak.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <HardwareSerial.h>
// Wi-Fi Credentials
const char* ssid = "YOUR_WIFI_SSID";      // ชื่อ Wi-Fi
const char* password = "YOUR_WIFI_PASSWORD"; // รหัสผ่าน Wi-Fi
// ThingSpeak Settings
const char* server = "api.thingspeak.com";
unsigned long channelID = YOUR_CHANNEL_ID; // Channel ID ของคุณ
const char* writeAPIKey = "YOUR_WRITE_API_KEY"; // Write API Key ของคุณ
// LCD และ PMS7003
LiquidCrystal_I2C lcd(0x27, 16, 2); // ที่อยู่ I2C ของ LCD
HardwareSerial pmsSerial(2);       // PMS7003 TX=16, RX=17
WiFiClient client;
void setup() {
  // เริ่มต้น LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi");
﻿
  // เริ่มต้น PMS7003
  pmsSerial.begin(9600, SERIAL_8N1, 16, 17);
  Serial.begin(115200);
  // เชื่อมต่อ Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
    lcd.setCursor(0, 1);
    lcd.print(".");
  }
  lcd.clear();
  lcd.print("WiFi Connected");
  // เริ่มต้น ThingSpeak
  ThingSpeak.begin(client);
}
void loop() {
  if (pmsSerial.available() >= 32) { // PMS7003 ส่งข้อมูล 32 ไบต์
    byte buffer[32];
    pmsSerial.readBytes(buffer, 32);
    if (buffer[0] == 0x42 && buffer[1] == 0x4D) { // ตรวจสอบ Header
      int pm2_5 = (buffer[12] << 8) + buffer[13]; // ค่า PM2.5
      int pm10 = (buffer[14] << 8) + buffer[15];  // ค่า PM10
﻿
      // แสดงผลบน LCD
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("PM2.5: ");
      lcd.print(pm2_5);
      lcd.print(" ug/m3");
      lcd.setCursor(0, 1);
      lcd.print("PM10: ");
      lcd.print(pm10);
      lcd.print(" ug/m3");
﻿
      // ส่งค่าขึ้น ThingSpeak
      ThingSpeak.setField(1, pm2_5); // Field 1: PM2.5
      ThingSpeak.setField(2, pm10);  // Field 2: PM10
      int responseCode = ThingSpeak.writeFields(channelID, writeAPIKey);
      // ตรวจสอบการส่งข้อมูล
      if (responseCode == 200) {
        Serial.println("Update successful");
      } else {
        Serial.print("Update failed, code: ");
        Serial.println(responseCode);
      }
      delay(15000); // ThingSpeak จำกัดการส่งข้อมูลทุก 15 วินาที
    }
  }
}
﻿
