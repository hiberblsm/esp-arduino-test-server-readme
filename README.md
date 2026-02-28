# ESP/Arduino Test Server (Public)

Hiber Bilisim Test Server, ESP32, ESP8266 ve Arduino tabanli IoT cihazlar icin HTTP, TCP, UDP ve MQTT protokollerinde hizli baglanti ve veri gonderim testi yapmanizi saglayan public bir test ortamidir. Bu sayfa, Hiber Bilisim IoT test sunucusuna baglanmak, gecici token uretmek ve cihaz haberlesmesini dogrulamak icin gerekli tum adimlari icerir.

Bu repo, kullanicilarin hizli protokol testi yapmasi icin hazirlanmis basit bir test sunucusudur.

Sunucu: `test.hibersoft.com.tr`

## Desteklenen Protokoller
- HTTP `2884`
- TCP `2885`
- UDP `2886`
- MQTT `2887`

## Test Akisi
1. `POST /token` ile gecici token al
2. Bu token ile HTTP/TCP/UDP testlerini yap
3. `GET /messages` ile gelen mesajlari kontrol et

## Token Alma

Her kullanici kendi benzersiz tokenini olusturur:

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/token
```

Ornek cevap:

```json
{
  "ok": true,
  "clientId": "c_xxxxxxxxxxxxxxxx",
  "token": "<TEMP_TOKEN>",
  "expiresAt": "2026-02-28T11:14:15.036Z",
  "ttlSec": 3600
}
```

Not: Tokenlar surelidir.
Not: `/messages` endpointi sadece tokeni olusturan kullanicinin mesajlarini listeler.

## HTTP Test (2884)

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/ingest \
  -H "content-type: application/json" \
  -H "x-api-key: <TEMP_TOKEN>" \
  -d '{"deviceId":"esp32-http","data":{"temp":24.5}}'
```

## TCP Test (2885)

Her mesaj JSON satiri olmali ve `\n` ile bitmelidir.

```bash
printf '{"token":"<TEMP_TOKEN>","deviceId":"esp32-tcp","data":{"voltage":3.3}}\n' | nc test.hibersoft.com.tr 2885
```

## UDP Test (2886)

```bash
echo -n '{"token":"<TEMP_TOKEN>","deviceId":"esp32-udp","data":{"rssi":-67}}' | nc -u -w1 test.hibersoft.com.tr 2886
```

## MQTT Test (2887)

MQTT kimlik bilgileri:
- Username: `testuser`
- Password: `PUBLIC_MQTT_2026_PASS`
- Topic: `test/` veya `devices/` ile baslamali

```bash
mosquitto_pub -h test.hibersoft.com.tr -p 2887 \
  -u "testuser" -P "PUBLIC_MQTT_2026_PASS" \
  -t "test/esp32-mqtt" \
  -m '{"deviceId":"esp32-mqtt","data":{"humidity":55}}'
```

## Mesajlari Listeleme

```bash
curl -s http://test.hibersoft.com.tr:2884/messages \
  -H "x-api-key: <TEMP_TOKEN>"
```
