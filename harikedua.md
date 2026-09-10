# Hari 2 — WiFi + ThingSpeak + Aktuator

> **Durasi:** 2 Jam | **Format:** Langsung Praktek

---

## Alur Hari 2

```
[0:00–0:20]  Buat Akun ThingSpeak + Channel + API Key
      ↓
[0:20–0:50]  Konek WiFi + Kirim Data ke ThingSpeak
      ↓
[0:50–1:20]  Pilih & Hubungkan Aktuator
      ↓
[1:20–1:40]  Integrasi Penuh — Satu Sketch
      ↓
[1:40–2:00]  Bentuk Kelompok + Tentukan Project
```

---

## 1. Setup ThingSpeak `[0:00 – 0:20]`

### 1.1 Buat Akun & Channel

1. Buka **thingspeak.com** → klik **Sign Up** (gratis, pakai akun MathWorks)
2. **Channels → My Channels → New Channel**
3. Isi:
   - Name: nama project kamu
   - Field 1: `Suhu`
   - Field 2: `Kelembapan`
4. Klik **Save Channel**

### 1.2 Dapat API Key

- Tab **API Keys** → copy **Write API Key**
- Contoh: `ABCDEFGH12345678`
- Catat juga **Channel ID** (tertera di atas halaman)

### 1.3 Cara Kerja

```
ESP32 → HTTP GET → api.thingspeak.com/update?api_key=...&field1=suhu&field2=lembap
                                    ↓
                            ThingSpeak simpan → tampil grafik di browser
```

> Limit update: **minimal 15 detik** per request. Response 200 = sukses.

---

## 2. Kode: WiFi + ThingSpeak `[0:20 – 0:50]`

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <DHT.h>

#define DHTPIN   4
#define DHTTYPE  DHT11

const char* ssid     = "NAMA_WIFI";
const char* password = "PASSWORD_WIFI";
String apiKey        = "YOUR_API_KEY";   // dari tab API Keys ThingSpeak

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500); Serial.print(".");
  }
  Serial.println("\nWiFi Terhubung!");
}

void loop() {
  float suhu       = dht.readTemperature();
  float kelembapan = dht.readHumidity();

  if (!isnan(suhu) && !isnan(kelembapan)) {
    HTTPClient http;
    String url = "http://api.thingspeak.com/update?api_key=" + apiKey
               + "&field1=" + String(suhu)
               + "&field2=" + String(kelembapan);
    http.begin(url);
    int code = http.GET();
    Serial.println("HTTP: " + String(code));  // 200 = sukses
    http.end();
  }
  delay(15000);  // ThingSpeak limit: min 15 detik
}
```

---

## 3. Pilih Aktuator `[0:50 – 1:20]`

| Aktuator | Trigger | Komponen |
|----------|---------|----------|
| Relay + Kipas DC | Suhu > 33°C | Relay module + kipas |
| Buzzer Alarm | Suhu > 40°C | Buzzer aktif/pasif |
| Servo Motor | Suhu > 35°C → buka | SG90 servo |
| Pompa Air | Kelembapan < 40% | Mini pump + relay |
| LED RGB | Indikator warna suhu | LED RGB / WS2812 |
| Relay + Pemanas | Suhu < 37°C | Bohlam kecil + relay |

> Semua logika sama: `if (kondisi) → digitalWrite(PIN, HIGH/LOW)` — beda hanya threshold dan pin.

### Kode Relay (Paling Umum)

Wiring:
```
VCC  →  5V
GND  →  GND
IN   →  GPIO 5
```

Kode:
```cpp
#define RELAY_PIN 5

// di setup():
pinMode(RELAY_PIN, OUTPUT);
digitalWrite(RELAY_PIN, HIGH);  // relay OFF dulu (active LOW)

// di loop(), setelah baca sensor:
if (suhu > 33.0) {
  digitalWrite(RELAY_PIN, LOW);   // relay ON → kipas nyala
} else {
  digitalWrite(RELAY_PIN, HIGH);  // relay OFF
}
```

> **Active LOW:** LOW = relay ON, HIGH = relay OFF. Kebalikan dari LED biasa.

---

## 4. Integrasi Penuh — Satu Sketch `[1:20 – 1:40]`

Struktur kode yang rapi — pisah jadi fungsi:

```cpp
// Deklarasi global: DHT, WiFi, relay, apiKey

float suhu, kelembapan;

void bacaSensor() {
  suhu       = dht.readTemperature();
  kelembapan = dht.readHumidity();
}

void kontrolAktuator() {
  digitalWrite(RELAY_PIN, suhu > 33.0 ? LOW : HIGH);
}

void kirimData() {
  if (WiFi.status() != WL_CONNECTED) return;
  HTTPClient http;
  String url = "http://api.thingspeak.com/update?api_key=" + apiKey
             + "&field1=" + String(suhu)
             + "&field2=" + String(kelembapan);
  http.begin(url); http.GET(); http.end();
}

void loop() {
  bacaSensor();
  if (!isnan(suhu)) {
    kontrolAktuator();
    kirimData();
  }
  delay(15000);
}
```

---

## 5. Bentuk Kelompok + Tentukan Project `[1:40 – 2:00]`

Isi template ini sekarang:

```
Nama Project  : _______________
Sensor        : DHT11 / DHT22
Aktuator      : _______________
Kondisi ON    : Suhu > ___ °C / Kelembapan < ___ %
Monitoring    : ThingSpeak
Channel ID    : _______________

Pembagian Kerja:
  - Hardware  : _______________
  - Kode      : _______________
  - Presentasi: _______________
```

### Ide Project Siap Pakai

| Project | Aktuator | Trigger |
|---------|----------|---------|
| 🌱 Smart Greenhouse | Pompa + kipas | Kelembapan < 40% / Suhu > 35°C |
| 🏠 Smart Room Comfort | Kipas + LED RGB | Skor kenyamanan |
| 👶 Baby Room Monitor | Buzzer | Suhu < 20°C atau > 32°C |
| 🖥 Server Room Cooler | Relay kipas | Suhu > 40°C |
| 🧫 Inkubator | Relay pemanas + kipas | Suhu < 37°C / > 39°C |

---

## Troubleshoot

| Error | Penyebab | Solusi |
|-------|----------|--------|
| WiFi terus print `"....."` | SSID/password salah, atau jaringan 5GHz | Cek kredensial, pakai hotspot HP (2.4GHz) |
| HTTP response `-1` atau `0` | Tidak ada internet / API Key salah | Cek koneksi, pastikan pakai Write Key bukan Read Key |
| ThingSpeak grafik tidak update | Delay terlalu cepat < 15 detik | Set `delay(15000)` minimal |
| Relay ON terus tidak bisa mati | Lupa active LOW | `HIGH` = OFF, `LOW` = ON untuk relay module |
| ESP32 restart terus (watchdog) | Loop diblok terlalu lama | Tambahkan `delay()` atau `yield()` |

---

## Recap Hari 2

- [x] Buat channel ThingSpeak + dapat API Key
- [x] Konek ESP32 ke WiFi
- [x] Kirim data suhu & kelembapan ke ThingSpeak real-time
- [x] Hubungkan relay / buzzer / aktuator pilihan
- [x] Integrasi penuh dalam satu sketch terstruktur
- [x] Kelompok terbentuk + project sudah ditentukan

**Hari 3:** Finishing project + Demo hardware hidup + Presentasi kelompok
