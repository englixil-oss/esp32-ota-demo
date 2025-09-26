# ESP32 OTA ENG
#include <WiFi.h>
#include <HTTPClient.h>
#include <HTTPUpdate.h>
#include <WiFiClientSecure.h>
#include "time.h"

#define WIFI_SSID      "ENG"
#define WIFI_PASSWORD  "99999999"

#define VERSION_URL "https://raw.githubusercontent.com/englixil-oss/esp32-ota-demo/main/DEMO/version.txt"
#define FIRMWARE_URL "https://raw.githubusercontent.com/englixil-oss/esp32-ota-demo/main/DEMO/OTA_GITHUB.ino.bin"
#define CURRENT_VERSION "0.9"
#define CHECK_INTERVAL 86400000UL

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 7 * 3600;
const int   daylightOffset_sec = 0;

unsigned long lastCheck = 0;


void updateStarted() { Serial.println("🔄 Bắt đầu cập nhật..."); }
void updateFinished() { Serial.println("✅ Cập nhật hoàn tất!"); }
void updateProgress(int cur, int total) {
  int percent = (cur * 100) / total;
  Serial.printf("Đang cập nhật: %d%% (%d/%d bytes)\n", percent, cur, total);
}
void updateError(int err) {
  Serial.printf("❌ Lỗi cập nhật: %d\n", err);
}


void syncTime() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  WiFi.setTxPower(WIFI_POWER_15dBm);
  WiFi.setAutoReconnect(true);
  Serial.print("Kết nối WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ WiFi connected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());
  long rssi = WiFi.RSSI();
  Serial.printf("📶 WiFi RSSI: %ld dBm\n", rssi);
  Serial.print("Đồng bộ thời gian...");
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("❌ Lỗi!");
    esp_restart();
    return;
  }
  Serial.println("✅ OK");
  Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
}


void checkAndUpdate() {
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  if (!http.begin(client, VERSION_URL)) {
    Serial.println("❌ Không thể kết nối đến VERSION_URL.");
    return;
  }

  int httpCode = http.GET();
  if (httpCode == HTTP_CODE_OK) {
    String newVersion = http.getString();
    newVersion.trim();

    Serial.printf("Current version: %s\n", CURRENT_VERSION);
    Serial.printf("Latest version : %s\n", newVersion.c_str());

    if (newVersion != CURRENT_VERSION && newVersion.length() > 0) {
      Serial.println("🔄 Có bản mới, bắt đầu update...");

      httpUpdate.onStart(updateStarted);
      httpUpdate.onEnd(updateFinished);
      httpUpdate.onProgress(updateProgress);
      httpUpdate.onError(updateError);

      WiFiClientSecure updateClient;
      updateClient.setInsecure();
      t_httpUpdate_return ret = httpUpdate.update(updateClient, FIRMWARE_URL);
    } else {
      Serial.println("Đang chạy bản mới nhất.");
    }
  } else {
    Serial.printf("❌ Không tải được version.txt. HTTP Code: %d\n", httpCode);
  }
  http.end();
}

void setup() {
  Serial.begin(115200);
  syncTime();
  checkAndUpdate();
  lastCheck = millis();
  Serial.printf("ESP32 Firmware Version: %s\n", CURRENT_VERSION);
}

void loop() {
  if (millis() - lastCheck >= CHECK_INTERVAL) {
    checkAndUpdate();
    lastCheck = millis();
  }
}

