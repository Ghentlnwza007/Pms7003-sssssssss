#include <WiFi.h>
#include <HTTPClient.h>
#include <SoftwareSerial.h>
#include <PMS.h>

// WiFi Credentials
const char* ssid = "Your_SSID";           // ใส่ชื่อ WiFi ของคุณ
const char* password = "Your_PASSWORD";   // ใส่รหัสผ่าน WiFi

// LINE Notify Token
String lineToken = "Your_LINE_Token";     // ใส่ LINE Notify Token

// PMS7003 Pin Configuration
#define PMS_RX 16  // ESP32 GPIO RX (เชื่อมกับ TX ของ PMS7003)
#define PMS_TX 17  // ESP32 GPIO TX (เชื่อมกับ RX ของ PMS7003)

// Smoke Sensor Pin (MQ-2 or MQ-7)
#define SMOKE_SENSOR_PIN 34 // ขา Analog สำหรับเซ็นเซอร์ควัน

// Buzzer Pin
#define BUZZER_PIN 26 // ขา Digital สำหรับ Buzzer

// PMS7003 Object
SoftwareSerial pmsSerial(PMS_RX, PMS_TX); // กำหนดขาสำหรับ PMS7003
PMS pms(pmsSerial);
PMS::DATA data;

// ค่าแจ้งเตือน PM2.5 และ ควัน (ปรับได้)
const int PM25_ALERT_THRESHOLD = 50; // µg/m³
const int SMOKE_ALERT_THRESHOLD = 300; // ค่า Analog Sensor (0-1023)

void setup() {
  Serial.begin(115200); // เปิด Serial Monitor
  pmsSerial.begin(9600); // ตั้งค่า Serial สำหรับ PMS7003

  // กำหนด Buzzer Pin เป็น Output
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // ปิดเสียงเริ่มต้น

  // เชื่อมต่อ Wi-Fi
  Serial.print("Connecting to WiFi");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected!");
}

void sendLineNotify(String message) {
  if (WiFi.status() == WL_CONNECTED) { // เช็คว่า Wi-Fi เชื่อมต่ออยู่หรือไม่
    HTTPClient http;
    http.begin("https://notify-api.line.me/api/notify"); // URL สำหรับ LINE Notify
    http.addHeader("Content-Type", "application/x-www-form-urlencoded");
    http.addHeader("Authorization", "Bearer " + lineToken);
    int httpResponseCode = http.POST("message=" + message);

    if (httpResponseCode > 0) {
      Serial.println("LINE Notify Sent! Response: " + String(httpResponseCode));
    } else {
      Serial.println("Error sending LINE Notify: " + String(httpResponseCode));
    }
    http.end();
  } else {
    Serial.println("WiFi Disconnected. Cannot send LINE Notify.");
  }
}

void alertBuzzer(int duration) {
  digitalWrite(BUZZER_PIN, HIGH); // เปิดเสียง Buzzer
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);  // ปิดเสียง Buzzer
}

void loop() {
  // อ่านค่าฝุ่นจาก PMS7003
  if (pms.readUntil(data)) {
    Serial.println("------ PM Values ------");
    Serial.print("PM1.0: "); Serial.print(data.PM_AE_UG_1_0); Serial.println(" µg/m³");
    Serial.print("PM2.5: "); Serial.print(data.PM_AE_UG_2_5); Serial.println(" µg/m³");
    Serial.print("PM10: "); Serial.print(data.PM_AE_UG_10_0); Serial.println(" µg/m³");

    // แจ้งเตือนถ้าค่า PM2.5 เกินเกณฑ์
    if (data.PM_AE_UG_2_5 > PM25_ALERT_THRESHOLD) {
      String message = "⚠️ ค่าฝุ่น PM2.5 สูงเกินมาตรฐาน! (" + String(data.PM_AE_UG_2_5) + " µg/m³)";
      sendLineNotify(message);
      delay(60000); // เว้นการแจ้งเตือน 1 นาที
    }
  }

  // อ่านค่าควันจาก MQ-2
  int smokeValue = analogRead(SMOKE_SENSOR_PIN);
  Serial.print("Smoke Sensor Value: ");
  Serial.println(smokeValue);

  // แจ้งเตือนถ้าค่าควันเกินเกณฑ์
  if (smokeValue > SMOKE_ALERT_THRESHOLD) {
    String message = "⚠️ ควันสูงเกินมาตรฐาน! (" + String(smokeValue) + ")";
    sendLineNotify(message);

    // เปิดเสียงแจ้งเตือน
    alertBuzzer(1000); // เปิดเสียงเป็นเวลา 1 วินาที
    delay(4000);       // พักเสียง 4 วินาที (รวมเป็น 5 วินาที)

    delay(60000); // เว้นการแจ้งเตือน 1 นาที
  }

  delay(5000); // อ่านค่าทุก 5 วินาที
}
