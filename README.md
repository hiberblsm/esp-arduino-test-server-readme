# IoT Test Sunucusu — Ücretsiz & Herkese Açık

ESP32, ESP8266, Arduino ve benzeri IoT cihazları için herkese açık, ücretsiz bir test ortamı.  
Token al, cihazını bağla, dashboard'dan akışı gerçek zamanlı izle — sıfır kurulum, sıfır ücret.

**Sunucu:** `test.hibersoft.com.tr`  
**Dashboard:** `https://test.hibersoft.com.tr`

---

## İçindekiler

- [Nasıl Çalışır](#nasıl-çalışır)
- [Portlar](#portlar)
- [1 Dakikada Başla](#1-dakikada-başla)
- [Token Alma](#token-alma)
- [HTTP ile Veri Gönderme](#http-ile-veri-gönderme)
- [TCP ile Veri Gönderme](#tcp-ile-veri-gönderme)
- [UDP ile Veri Gönderme](#udp-ile-veri-gönderme)
- [MQTT ile Veri Gönderme ve Geri Bildirim Alma](#mqtt-ile-veri-gönderme-ve-geri-bildirim-alma)
- [Dashboard'dan Cihaza Komut Gönderme](#dashboarddan-cihaza-komut-gönderme)
- [Arduino / ESP Kod Örnekleri](#arduino--esp-kod-örnekleri)
- [Dashboard Kullanımı](#dashboard-kullanımı)
- [API Referansı](#api-referansı)
- [Limitler](#limitler)

---

## Nasıl Çalışır

```
Cihazın                  Sunucu                    Dashboard
   │                        │                          │
   │── token al ───────────►│                          │
   │◄─ token + clientId ────│                          │
   │                        │                          │
   │── veri gönder ────────►│── trafik kaydedilir ────►│ (canlı izle)
   │◄─ ACK / özel yanıt ────│                          │
   │                        │◄── downlink komutu ───── │ (sen gönderirsin)
   │◄─ MQTT downlink ───────│                          │
```

- Her token benzersiz bir **clientId** ile eşleşir; dashboard yalnızca kendi token'ına ait trafiği gösterir.
- MQTT mesajlarının dashboard'a bağlanması için payload içinde `token` alanı zorunludur.
- Token'lar **24 saat** geçerlidir.

---

## Portlar

| Protokol  | Port   |
|-----------|-------:|
| HTTP API  | `2884` |
| TCP       | `2885` |
| UDP       | `2886` |
| MQTT      | `2887` |

Dashboard (port `5010`) doğrudan erişilemez; `https://test.hibersoft.com.tr` üzerinden reverse proxy ile sunulur.

---

## 1 Dakikada Başla

```bash
# 1. Token al
curl -s -X POST http://test.hibersoft.com.tr:2884/token

# 2. Veri gönder
curl -s -X POST http://test.hibersoft.com.tr:2884/ingest \
  -H "Content-Type: application/json" \
  -H "x-api-key: BURAYA_TOKEN" \
  -d '{"deviceId":"esp32-test","data":{"temp":24.5,"relay":0}}'

# 3. https://test.hibersoft.com.tr adresini aç, tokeni yapıştır, trafiği izle
```

---

## Token Alma

Token; cihazını ve dashboard oturumunu birbirine bağlayan 48 karakterlik rastgele bir anahtardır.

### Dashboard üzerinden

`https://test.hibersoft.com.tr` adresini aç, **"24 saatlik token oluştur"** butonuna bas.

### HTTP API ile

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/token
```

Yanıt:

```json
{
  "ok": true,
  "token": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
  "clientId": "c_0123456789abcdef",
  "expiresAt": "2026-06-15T10:00:00.000Z",
  "ttlSec": 86400,
  "activeTokenCount": 42,
  "maxActiveTokens": 1000
}
```

### Token Doğrulama

```bash
curl -s http://test.hibersoft.com.tr:2884/token/verify \
  -H "x-api-key: BURAYA_TOKEN"
```

Yanıt:

```json
{
  "ok": true,
  "clientId": "c_0123456789abcdef",
  "token": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
  "expiresAt": "2026-06-15T10:00:00.000Z",
  "ttlSec": 82340
}
```

Token geçersizse `401` döner.

---

## HTTP ile Veri Gönderme

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/ingest \
  -H "Content-Type: application/json" \
  -H "x-api-key: BURAYA_TOKEN" \
  -d '{"deviceId":"esp32-salon","data":{"sicaklik":23.1,"nem":58}}'
```

Yanıt:

```json
{
  "ok": true,
  "protocol": "http",
  "deviceId": "esp32-salon",
  "ackAt": "2026-06-14T10:00:00.000Z"
}
```

> Dashboard'dan o cihaz için HTTP yanıt kuyruğuna bir komut eklediysen, cihaz bir sonraki `/ingest` isteğinde normal ACK yerine o komutu alır. Bkz. [HTTP Yanıt Kuyruğu](#1-http-yanıt-kuyruğu).

---

## TCP ile Veri Gönderme

Her mesaj `\n` ile biten tek satır JSON'dur. Token, body içinde `token` alanında gider.

```bash
printf '{"token":"BURAYA_TOKEN","deviceId":"esp32-tcp","data":{"voltaj":3.3}}\n' \
  | nc test.hibersoft.com.tr 2885
```

Yanıt:

```json
{"ok":true,"protocol":"tcp","deviceId":"esp32-tcp","ackAt":"2026-06-14T10:00:00.000Z"}
```

Bağlantı açık tutulabilir; arka arkaya birden fazla satır gönderilebilir.  
30 saniye veri gelmezse sunucu bağlantıyı kapatır.

---

## UDP ile Veri Gönderme

```bash
echo -n '{"token":"BURAYA_TOKEN","deviceId":"esp32-udp","data":{"rssi":-67}}' \
  | nc -u -w1 test.hibersoft.com.tr 2886
```

Her pakette `token` zorunludur; UDP'de kalıcı bağlantı olmadığından kimlik doğrulama paket başına yapılır.

---

## MQTT ile Veri Gönderme ve Geri Bildirim Alma

MQTT hem uplink (cihaz → sunucu) hem downlink (sunucu → cihaz) için kullanılır.

### Bağlantı Bilgileri

| Alan      | Değer                   |
|-----------|-------------------------|
| Host      | `test.hibersoft.com.tr` |
| Port      | `2887`                  |
| Kullanıcı | `testuser`              |
| Parola    | `PUBLIC_MQTT_2026_PASS` |

### İzin Verilen Topic Ön Ekleri

```
test/
devices/
```

### Uplink — Cihazdan Sunucuya

```bash
mosquitto_pub \
  -h test.hibersoft.com.tr -p 2887 \
  -u "testuser" -P "PUBLIC_MQTT_2026_PASS" \
  -t "devices/esp32-mqtt/uplink" \
  -m '{"token":"BURAYA_TOKEN","deviceId":"esp32-mqtt","data":{"nem":62,"sicaklik":21}}'
```

> Payload'da `token` alanı yoksa mesaj sunucuya ulaşır ama dashboard'da **görünmez**.

### Downlink — Sunucudan Cihaza

Cihazın dinlemesi gereken topic: `devices/<deviceId>/downlink`

```bash
mosquitto_sub \
  -h test.hibersoft.com.tr -p 2887 \
  -u "testuser" -P "PUBLIC_MQTT_2026_PASS" \
  -t "devices/esp32-mqtt/downlink"
```

Dashboard veya API üzerinden downlink gönderildiğinde cihaz şu formatı alır:

```json
{
  "deviceId": "esp32-mqtt",
  "data": { "command": "relay_on", "value": 1 },
  "sentAt": "2026-06-14T10:00:00.000Z"
}
```

### Tam MQTT Akışı

```
Cihaz                                   Broker (:2887)
  │── CONNECT (testuser / parola) ─────►│
  │◄─ CONNACK ──────────────────────────│
  │── SUBSCRIBE devices/esp32/downlink ►│
  │◄─ SUBACK ───────────────────────────│
  │── PUBLISH devices/esp32/uplink ────►│  → dashboard'a kayıt
  │                                     │◄── dashboard downlink gönderir
  │◄─ PUBLISH devices/esp32/downlink ───│  ← komut cihaza gelir
```

---

## Dashboard'dan Cihaza Komut Gönderme

### 1. HTTP Yanıt Kuyruğu

HTTP cihazlar için. Cihazın bir sonraki `/ingest` isteğinde özel bir yanıt almasını sağlar.

**Dashboard'dan:**
1. `HTTP cevap kuyruğu` alanına `deviceId` yaz.
2. HTTP status kodunu seç (örn. `202`).
3. JSON body alanına cihazın alacağı veriyi yaz.
4. **"Cevabı sıraya al"** butonuna bas.

**API ile:**

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/http-response \
  -H "Content-Type: application/json" \
  -H "x-api-key: BURAYA_TOKEN" \
  -d '{"deviceId":"esp32-salon","status":202,"payload":{"command":"relay_on"}}'
```

`esp32-salon` bir sonraki `/ingest`'te şunu alır:

```json
{"command": "relay_on"}
```

> Her kayıt ilgili cihaz tarafından **bir kez** alındıktan sonra kuyruktan silinir.

### 2. MQTT Downlink

MQTT cihazlar için. Cihaz `devices/<deviceId>/downlink` topic'ini dinliyorsa mesajı anında alır.

**Dashboard'dan:**
1. `MQTT downlink` alanına `deviceId` yaz.
2. JSON data alanına komutu yaz.
3. **"MQTT gönder"** butonuna bas.

**API ile:**

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/mqtt-publish \
  -H "Content-Type: application/json" \
  -H "x-api-key: BURAYA_TOKEN" \
  -d '{"deviceId":"esp32-mqtt","payload":{"command":"ping","relay":1}}'
```

---

## Arduino / ESP Kod Örnekleri

> Örnekler **ArduinoJson v6** ile yazılmıştır. v7 kullanıyorsan `StaticJsonDocument<N>` yerine `JsonDocument`, `createNestedObject` yerine `doc["key"].to<JsonObject>()` kullan.

### ESP32 — HTTP ile Veri Gönderme ve Komut Alma

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

const char* SSID       = "WiFi_Adiniz";
const char* WIFI_PASS  = "WiFi_Sifreniz";
const char* TOKEN      = "BURAYA_TOKEN";
const char* DEVICE_ID  = "esp32-salon";
const char* INGEST_URL = "http://test.hibersoft.com.tr:2884/ingest";

void setup() {
  Serial.begin(115200);
  WiFi.begin(SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi bağlandı");
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) return;

  HTTPClient http;
  http.begin(INGEST_URL);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("x-api-key", TOKEN);

  StaticJsonDocument<256> doc;
  doc["deviceId"] = DEVICE_ID;
  JsonObject data = doc.createNestedObject("data");
  data["sicaklik"] = 23.5;
  data["nem"]      = 60;
  data["relay"]    = 0;

  String body;
  serializeJson(doc, body);

  int statusCode = http.POST(body);
  if (statusCode > 0) {
    String response = http.getString();
    Serial.println("Yanıt [" + String(statusCode) + "]: " + response);

    StaticJsonDocument<256> resp;
    if (deserializeJson(resp, response) == DeserializationError::Ok) {
      if (resp.containsKey("command")) {
        String cmd = resp["command"].as<String>();
        if (cmd == "relay_on")  { /* röleyi aç */ }
        if (cmd == "relay_off") { /* röleyi kapat */ }
      }
    }
  } else {
    Serial.println("HTTP hatası: " + http.errorToString(statusCode));
  }

  http.end();
  delay(10000);
}
```

---

### ESP32 — MQTT Uplink + Downlink Geri Bildirim

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

const char* SSID      = "WiFi_Adiniz";
const char* WIFI_PASS = "WiFi_Sifreniz";
const char* TOKEN     = "BURAYA_TOKEN";
const char* DEVICE_ID = "esp32-mqtt-01";

const char* MQTT_HOST = "test.hibersoft.com.tr";
const int   MQTT_PORT = 2887;
const char* MQTT_USER = "testuser";
const char* MQTT_PASS = "PUBLIC_MQTT_2026_PASS";

const char* UPLINK_TOPIC   = "devices/esp32-mqtt-01/uplink";
const char* DOWNLINK_TOPIC = "devices/esp32-mqtt-01/downlink";

WiFiClient   wifiClient;
PubSubClient mqtt(wifiClient);

void onMqttMessage(char* topic, byte* payload, unsigned int length) {
  String message;
  for (unsigned int i = 0; i < length; i++) message += (char)payload[i];
  Serial.println("Downlink: " + message);

  StaticJsonDocument<512> doc;
  if (deserializeJson(doc, message) != DeserializationError::Ok) return;

  if (doc["data"].containsKey("command")) {
    String cmd = doc["data"]["command"].as<String>();
    if (cmd == "relay_on")  digitalWrite(2, HIGH);
    if (cmd == "relay_off") digitalWrite(2, LOW);
    if (cmd == "ping")      Serial.println("Pong!");
  }
}

void connectMqtt() {
  while (!mqtt.connected()) {
    Serial.print("MQTT bağlanıyor...");
    String clientId = "esp32-" + String(random(0xffff), HEX);
    if (mqtt.connect(clientId.c_str(), MQTT_USER, MQTT_PASS)) {
      Serial.println(" bağlandı");
      mqtt.subscribe(DOWNLINK_TOPIC);
    } else {
      Serial.println(" başarısız (rc=" + String(mqtt.state()) + "), 5sn bekle");
      delay(5000);
    }
  }
}

void sendUplink() {
  StaticJsonDocument<512> doc;
  doc["token"]    = TOKEN;
  doc["deviceId"] = DEVICE_ID;
  JsonObject data = doc.createNestedObject("data");
  data["sicaklik"] = 22.8;
  data["nem"]      = 55;

  char buffer[512];
  serializeJson(doc, buffer);
  mqtt.publish(UPLINK_TOPIC, buffer);
}

void setup() {
  Serial.begin(115200);
  pinMode(2, OUTPUT);

  WiFi.begin(SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi bağlandı");

  mqtt.setServer(MQTT_HOST, MQTT_PORT);
  mqtt.setCallback(onMqttMessage);
  connectMqtt();
}

void loop() {
  if (!mqtt.connected()) connectMqtt();
  mqtt.loop();

  static unsigned long lastSend = 0;
  if (millis() - lastSend > 10000) {
    sendUplink();
    lastSend = millis();
  }
}
```

> `mqtt.loop()` çağrısı olmadan cihazın downlink mesajı alması mümkün değildir.

---

### ESP8266 — TCP ile Veri Gönderme

```cpp
#include <ESP8266WiFi.h>
#include <ArduinoJson.h>

const char* SSID      = "WiFi_Adiniz";
const char* WIFI_PASS = "WiFi_Sifreniz";
const char* TOKEN     = "BURAYA_TOKEN";
const char* DEVICE_ID = "esp8266-tcp-01";
const char* TCP_HOST  = "test.hibersoft.com.tr";
const int   TCP_PORT  = 2885;

WiFiClient client;

void setup() {
  Serial.begin(115200);
  WiFi.begin(SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi bağlandı");
}

void loop() {
  if (client.connect(TCP_HOST, TCP_PORT)) {
    StaticJsonDocument<256> doc;
    doc["token"]    = TOKEN;
    doc["deviceId"] = DEVICE_ID;
    doc.createNestedObject("data")["voltaj"] = 3.28;

    String line;
    serializeJson(doc, line);
    line += "\n";
    client.print(line);

    unsigned long t = millis();
    while (client.connected() && millis() - t < 3000) {
      if (client.available()) {
        Serial.println("ACK: " + client.readStringUntil('\n'));
        break;
      }
    }
    client.stop();
  }

  delay(15000);
}
```

---

## Dashboard Kullanımı

Dashboard `https://test.hibersoft.com.tr` adresinde açılır.

**İlk açılışta:**
- **"24 saatlik token oluştur"** — yeni token + clientId otomatik gelir.
- Mevcut token varsa **"Var olan token ile devam et"** formuna yapıştır.

**Oturum açıkken:**

| Panel                   | Ne Gösterir                                                        |
|-------------------------|--------------------------------------------------------------------|
| Oturum bilgisi          | `clientId`, token, kalan süre                                      |
| Gelen akış              | HTTP / TCP / UDP / MQTT ile gelen tüm mesajlar, protokol + payload |
| Giden akış              | Cihaza gönderilen HTTP yanıtları ve MQTT downlink mesajları        |
| Token cihazları         | Bu token ile veri gönderen cihazlar, protokoller, son payload      |
| Bekleyen HTTP yanıtları | Henüz cihaz tarafından alınmamış HTTP yanıt kuyruğu               |

**Filtreleme:**
- **Gelen / Giden** sekmesiyle yönü ayır.
- Protokol filtresiyle `HTTP`, `TCP`, `UDP`, `MQTT` veya tümünü gör.
- **"İzlenen deviceId"** alanına cihaz adı yazarak tek cihazın trafiğini izole et.
- Cihaz veya trafik satırındaki **"Seç"** butonu; deviceId'yi filtreye, HTTP ve MQTT formlarına otomatik taşır.

Dashboard her ~2 saniyede bir otomatik güncellenir.

---

## API Referansı

Token `x-api-key` header'ı veya `Authorization: Bearer TOKEN` şeklinde gönderilir.

### Cihaz API — `http://test.hibersoft.com.tr:2884`

| Method | Endpoint        | Auth        | Açıklama                            |
|--------|-----------------|:-----------:|-------------------------------------|
| POST   | `/token`        | —           | Yeni token + clientId oluştur       |
| GET    | `/token/verify` | `x-api-key` | Token doğrula, kalan süreyi gör     |
| POST   | `/ingest`       | `x-api-key` | HTTP ile veri gönder                |
| GET    | `/messages`     | `x-api-key` | Token'a ait mesaj geçmişini listele |
| GET    | `/health`       | —           | Sunucu durumu ve port bilgisi       |

### Dashboard API — `https://test.hibersoft.com.tr`

| Method | Endpoint                   | Auth        | Açıklama                       |
|--------|----------------------------|:-----------:|-------------------------------|
| GET    | `/dashboard/data`          | `x-api-key` | Trafik, cihazlar, istatistikler|
| POST   | `/dashboard/http-response` | `x-api-key` | HTTP yanıt kuyruğuna ekle      |
| POST   | `/dashboard/mqtt-publish`  | `x-api-key` | MQTT downlink gönder           |

### Sağlık Kontrolü

```bash
curl -s http://test.hibersoft.com.tr:2884/health
```

```json
{
  "ok": true,
  "env": "production",
  "host": "0.0.0.0",
  "ports": { "http": 2884, "tcp": 2885, "udp": 2886, "mqtt": 2887 },
  "uptimeSec": 3600
}
```

---

## Limitler

| Parametre                   | Değer                          |
|-----------------------------|--------------------------------|
| Max body / paket boyutu     | `4096` byte                    |
| HTTP istek limiti (IP/dak.) | `120` istek (`/health` hariç)  |
| Token oluşturma (IP/dak.)   | `20` istek                     |
| Token geçerlilik süresi     | `24 saat`                      |
| Max eş zamanlı aktif token  | `1000`                         |
| Max mesaj geçmişi           | `200` kayıt                    |
| Max bekleyen HTTP yanıtı    | `25` / cihaz                   |
