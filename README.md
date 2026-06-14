# ESP / Arduino IoT Test Server

Bu proje ESP32, ESP8266, Arduino ve benzeri IoT cihazlari icin public test sunucusudur. HTTP, TCP, UDP ve MQTT ile veri alir; dashboard uzerinden token bazli cihaz trafigini gosterir ve cihaza HTTP cevabi ya da MQTT downlink mesaji gondermeyi saglar.

Public sunucu: `test.hibersoft.com.tr`

## Portlar

| Protokol | Port |
| --- | ---: |
| HTTP API | `2884` |
| TCP | `2885` |
| UDP | `2886` |
| MQTT | `2887` |
| Dashboard ic port | `5010` |

Dashboard web adresi normalde reverse proxy ile alan adindan acilir:

```text
https://test.hibersoft.com.tr/
```

## Hizli Kullanim

1. `https://test.hibersoft.com.tr/` adresini ac.
2. `24 saatlik token olustur` butonu ile token al.
3. Tokeni cihaza yaz.
4. Cihaz HTTP, TCP, UDP veya MQTT ile veri gondersin.
5. Dashboard'da `Gelen akis`, `Giden akis`, `Token cihazlari` ve `Bekleyen HTTP cevaplari` alanlarindan trafigi izle.
6. Gerekirse dashboard'dan ayni `deviceId` icin HTTP cevabi kuyrukla veya MQTT downlink gonder.

Tokenlar 24 saat gecerlidir. Token suresi dolunca yeni token alinmalidir.

## Token Alma

### Otomatik token

Dashboard acildiginda `24 saatlik token olustur` butonuna bas. Sistem token ve `clientId` bilgisini otomatik olusturur. Cihaza sadece token yazman yeterlidir.

### Manuel token API

HTTP API ile token almak icin:

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/token
```

Ornek cevap:

```json
{
  "ok": true,
  "token": "TEMP_TOKEN",
  "clientId": "c_0123456789abcdef",
  "expiresAt": "2026-06-15T10:00:00.000Z",
  "ttlSec": 86400,
  "activeTokenCount": 1,
  "maxActiveTokens": 1000
}
```

Var olan tokeni kontrol etmek icin:

```bash
curl -s http://test.hibersoft.com.tr:2884/token/verify \
  -H "x-api-key: TEMP_TOKEN"
```

## Dashboard Kullanimi

Dashboard token olmadan acilmaz. Yeni token olusturabilir veya var olan tokeni girebilirsin. Token dogrulaninca:

- `clientId`, token bitis zamani ve sabit portlar gorunur.
- `Gelen akis` token ile gelen HTTP, TCP, UDP ve tokenli MQTT mesajlarini listeler.
- `Giden akis` dashboard'dan cihaza gonderilen HTTP cevaplarini ve MQTT downlink mesajlarini listeler.
- `Token cihazlari` bu token ile veri gonderen cihazlari, kullandiklari protokolleri ve son payload'i gosterir.
- `Izlenen deviceId` filtresi ile tek cihaz trafigi izlenebilir.
- Cihaz satirindaki `Sec` butonu deviceId bilgisini filtreye, HTTP cevap formuna ve MQTT formuna tasir.

## HTTP ile Veri Gonderme

Endpoint:

```text
POST http://test.hibersoft.com.tr:2884/ingest
```

Token `x-api-key` header'i ile gonderilir:

```bash
curl -s -X POST http://test.hibersoft.com.tr:2884/ingest \
  -H "content-type: application/json" \
  -H "x-api-key: TEMP_TOKEN" \
  -d '{"deviceId":"esp32-http","data":{"temp":24.5,"relay":0}}'
```

Varsayilan cevap:

```json
{
  "ok": true,
  "protocol": "http",
  "deviceId": "esp32-http",
  "ackAt": "2026-06-14T10:00:00.000Z"
}
```

Dashboard'da ayni `deviceId` icin ozel HTTP cevabi kuyruklanmissa, cihaz bir sonraki `/ingest` isteginde bu JSON'u alir. Bu yontem HTTP cihazlara komut veya ayar cevabi gondermek icindir.

## Dashboard'dan HTTP Cevabi Gonderme

Dashboard formundan:

1. `HTTP cevap kuyrugu` alanina `deviceId` yaz.
2. Donmesini istedigin HTTP status kodunu sec.
3. JSON body alanina cihazin alacagi cevabi yaz.
4. `Cevabi siraya al` butonuna bas.

API ile ayni islem:

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/http-response \
  -H "content-type: application/json" \
  -H "x-api-key: TEMP_TOKEN" \
  -d '{"deviceId":"esp32-http","status":202,"payload":{"ok":true,"command":"relay_on"}}'
```

Bundan sonra `esp32-http` deviceId'si ayni token ile `/ingest` yaptiginda cevap:

```json
{
  "ok": true,
  "command": "relay_on"
}
```

Not: HTTP cevaplari kuyruktadir. Her kuyruk kaydi ilgili cihaz tarafindan bir kez alininca silinir.

## TCP ile Veri Gonderme

TCP mesajlari JSON satiri olmali ve `\n` ile bitmelidir. Token body icinde `token` alaninda gider.

```bash
printf '{"token":"TEMP_TOKEN","deviceId":"esp32-tcp","data":{"voltage":3.3}}\n' \
  | nc test.hibersoft.com.tr 2885
```

Basarili cevap:

```json
{"ok":true,"protocol":"tcp","deviceId":"esp32-tcp","ackAt":"2026-06-14T10:00:00.000Z"}
```

## UDP ile Veri Gonderme

UDP paketi JSON olmali ve token body icinde `token` alaninda gider.

```bash
echo -n '{"token":"TEMP_TOKEN","deviceId":"esp32-udp","data":{"rssi":-67}}' \
  | nc -u -w1 test.hibersoft.com.tr 2886
```

Sunucu basarili pakete UDP ACK datagram'i dondurur.

## MQTT ile Veri Gonderme

MQTT kimlik bilgileri:

```text
username: testuser
password: PUBLIC_MQTT_2026_PASS
```

Izinli topic prefixleri:

```text
test/
devices/
```

MQTT publish ornegi:

```bash
mosquitto_pub -h test.hibersoft.com.tr -p 2887 \
  -u "testuser" -P "PUBLIC_MQTT_2026_PASS" \
  -t "devices/esp32-mqtt/uplink" \
  -m '{"token":"TEMP_TOKEN","deviceId":"esp32-mqtt","data":{"humidity":55}}'
```

MQTT mesajinin dashboard'da ilgili token altinda gorunmesi icin payload icinde `token` bulunmalidir. Token yoksa MQTT mesaj kabul edilir ama dashboard oturumuna baglanamaz.

## Dashboard'dan MQTT Downlink Gonderme

Dashboard formundan:

1. `MQTT downlink` alanina `deviceId` yaz.
2. JSON data alanina cihaza gidecek komutu yaz.
3. `MQTT gonder` butonuna bas.

Sunucu su topic'e publish eder:

```text
devices/<deviceId>/downlink
```

API ile gonderim:

```bash
curl -s -X POST https://test.hibersoft.com.tr/dashboard/mqtt-publish \
  -H "content-type: application/json" \
  -H "x-api-key: TEMP_TOKEN" \
  -d '{"deviceId":"esp32-mqtt","payload":{"command":"ping","relay":1}}'
```

Cihazin dinlemesi gereken topic:

```bash
mosquitto_sub -h test.hibersoft.com.tr -p 2887 \
  -u "testuser" -P "PUBLIC_MQTT_2026_PASS" \
  -t "devices/esp32-mqtt/downlink"
```

Downlink payload format:

```json
{
  "deviceId": "esp32-mqtt",
  "data": {
    "command": "ping",
    "relay": 1
  },
  "sentAt": "2026-06-14T10:00:00.000Z"
}
```

## Gelen Verileri Listeleme

HTTP API ile tokena ait mesajlari almak icin:

```bash
curl -s http://test.hibersoft.com.tr:2884/messages \
  -H "x-api-key: TEMP_TOKEN"
```

Dashboard'un kullandigi canli veri endpoint'i:

```bash
curl -s https://test.hibersoft.com.tr/dashboard/data \
  -H "x-api-key: TEMP_TOKEN"
```

Bu endpoint session, portlar, istatistikler, trafik, bekleyen HTTP cevaplari ve token cihazlarini dondurur.

## Saglik Kontrolu

```bash
curl -s http://test.hibersoft.com.tr:2884/health
```

Ornek cevap:

```json
{
  "ok": true,
  "env": "production",
  "host": "0.0.0.0",
  "ports": {
    "http": 2884,
    "tcp": 2885,
    "udp": 2886,
    "mqtt": 2887
  },
  "uptimeSec": 3600
}
```

## Sunucu Yonetimi

aapanel icin calistirma komutu:

```bash
bash /www/wwwroot/node_project/test-server/scripts/start.sh
```

`start.sh` davranisi:

- Sunucu calismiyorsa start eder.
- PID dosyasi veya portlardan calistigini gorurse once durdurur, sonra tekrar baslatir.
- PID dosyasini `.runtime/iot-server.pid` icinde tutar.
- Eski `/tmp/iot-server.pid` dosyasi varsa stop tarafinda yine okunur.
- Loglari `/www/wwwlogs/iot-test-server.log` dosyasina yazar ve son loglari komut ciktisinda gosterir.

Manuel komutlar:

```bash
npm run start
npm run stop
npm run restart
```

Canli log:

```bash
tail -f /www/wwwlogs/iot-test-server.log
```

## Limitler

- Max body/paket: `4096` byte
- HTTP rate limit: IP basina dakikada `120` istek
- Token olusturma rate limit: IP basina dakikada `20` istek
- Token TTL: `86400` saniye
- Max aktif token: `1000`
- Max mesaj gecmisi: `200`
