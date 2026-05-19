# ESP/Arduino Test Server (Public)

Hiber Bilisim Test Server, ESP32, ESP8266 ve Arduino tabanli IoT cihazlar icin HTTP, TCP, UDP ve MQTT protokollerinde hizli baglanti ve veri gonderim testi yapmanizi saglayan public bir test ortamidir. Bu sayfa, Hiber Bilisim IoT test sunucusuna baglanmak, 24 saat gecerli token olusturmak ve cihaz haberlesmesini dogrulamak icin gerekli tum adimlari icerir.

Bu repo, kullanicilarin hizli protokol testi yapmasi icin hazirlanmis basit bir test sunucusudur.

Sunucu: `test.hibersoft.com.tr`

## Desteklenen Protokoller
- HTTP `2884`
- TCP `2885`
- UDP `2886`
- MQTT `2887`

## Kullanim Ozeti
1. Tarayicida `https://test.hibersoft.com.tr/` adresini ac
2. Ilk ekranda `Token Olustur` butonu ile 24 saat gecerli token al
3. Ayni tokeni cihazinda HTTP, TCP, UDP veya MQTT testleri icin kullan
4. Dashboard ekraninda gelen ve giden akisleri izle
5. Istersen secili cihaz icin ozel HTTP cevabi hazirla veya MQTT downlink gonder

Dashboard akisi ozellikle basit tutulmustur:

- Ilk acilista sadece token olusturma / mevcut token girme ekrani gelir
- Token olusmadan calisma ekrani acilmaz
- Dashboard oturumu ile cihaz testleri ayni token uzerinden baglanir
- Token ve clientId otomatik olusur; kullanici sadece tokeni cihaza yazar
- Portlar sabittir ve dashboardda otomatik gosterilir

## Token Alma

Token iki farkli yoldan alinabilir:

1. En kolay yol dashboard uzerindeki `Token Olustur` butonudur
2. API ile almak istersen su endpoint kullanilir:

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
  "ttlSec": 86400
}
```

Not: Tokenlar 24 saat gecerlidir.
Not: `/messages` endpointi sadece tokeni olusturan kullanicinin mesajlarini listeler.

## Dashboard

Kullanici dashboard adresi:

```bash
https://test.hibersoft.com.tr/
```

Dashboard acildiginda ilk olarak token ekrani gelir. Burada ya yeni token olusturulur ya da mevcut token girilir. Token dogrulaninca dashboard calisma ekrani acilir. clientId otomatik atanir; kullanici clientId girmez.

- HTTP, TCP, UDP ve token ile iliskilendirilmis MQTT trafiklerini gelen/giden sekmelerinde gosterir
- Secilen `deviceId` icin bir sonraki HTTP `/ingest` cevabini tamamen sizin verdiginiz JSON ile doner
- `devices/<deviceId>/downlink` topic'ine MQTT cihaz mesaji gonderir
- Kullanici tokeni, clientId bilgisi ve sabit portlari tek ekranda gorebilir

Kullanici tarafinda dashboard ic portu gorunmez. Web erisimi dogrudan alan adi uzerinden yapilir.

Ozel HTTP cevap kuyrugu icin dashboard'in kullandigi API ornegi:

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/http-response \
  -H "content-type: application/json" \
  -H "x-api-key: <TEMP_TOKEN>" \
  -d '{"deviceId":"esp32-http","status":202,"payload":{"ok":true,"echo":"custom","reply":"from-dashboard"}}'
```

Bu istekten sonra ayni token ile gelen ilk `/ingest` istegi asagidaki gibi sizin verdiginiz JSON ile cevaplanir:

```json
{
  "ok": true,
  "echo": "custom",
  "reply": "from-dashboard"
}
```

MQTT cihaz mesaji gonderimi icin API ornegi:

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/mqtt-publish \
  -H "content-type: application/json" \
  -H "x-api-key: <TEMP_TOKEN>" \
  -d '{"deviceId":"esp32-mqtt","payload":{"command":"ping"}}'
```

MQTT publish'lerinin dashboard akisinda gorunmesi icin payload icine ayni token da eklenebilir:

```json
{
  "deviceId": "esp32-mqtt",
  "token": "<TEMP_TOKEN>",
  "data": {
    "humidity": 55
  }
}
```

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
