# Hari 2 — Blynk IoT + DHT + LED RGB + Buzzer

> **Durasi:** 2 Jam | **Format:** Langsung Praktek

---

## Alur Hari 2

```
[0:00–0:20]  Install Blynk + Buat Akun, Template, Auth Token
      ↓
[0:20–0:40]  Konek ESP32 ke Blynk via WiFi
      ↓
[0:40–1:00]  Kirim Data DHT ke Dashboard Blynk
      ↓
[1:00–1:40]  Aktuator — LED RGB + Buzzer + Update Status di Blynk
      ↓
[1:40–2:00]  Bentuk Kelompok + Tentukan Project
```

---

## Kenapa Blynk?

| Fitur | Keterangan |
|-------|------------|
| Dashboard di HP | Monitor suhu & kelembapan real-time dari Android/iOS |
| Setup cepat | Tidak perlu server, cukup Auth Token |
| Widget drag & drop | Gauge, Label, LED indicator, Button, Chart |
| Notifikasi | Bisa kirim notif ke HP kalau suhu melewati threshold |

---

## 1. Setup Blynk `[0:00 – 0:20]`

### 1.1 Install App & Buat Akun

1. Download **Blynk IoT** di Google Play / App Store
2. Buat akun gratis dengan email
3. Buka **blynk.cloud** di browser → login dengan akun yang sama

### 1.2 Buat Template

1. **Developer Zone → New Template**
2. Name: `Monitor DHT` | Hardware: `ESP32` | Connection: `WiFi` → Done

### 1.3 Buat Datastream (Virtual Pin)

**Template → Datastreams → New Datastream → Virtual Pin**

| Virtual Pin | Nama | Tipe | Unit |
|-------------|------|------|------|
| V0 | Suhu | Double | °C |
| V1 | Kelembapan | Double | % |
| V2 | Status LED | Integer | 0/1 |
| V3 | Status Buzzer | Integer | 0/1 |

### 1.4 Buat Device & Dapat Auth Token

1. **Devices → New Device → From Template** → pilih *Monitor DHT*
2. Copy **Auth Token** yang muncul — simpan, tidak bisa dilihat lagi

### 1.5 Setup Widget di Blynk App

| Widget | Datastream | Setting |
|--------|-----------|---------|
| Gauge | V0 Suhu | Min: 0, Max: 50, Unit: °C |
| Gauge | V1 Kelembapan | Min: 0, Max: 100, Unit: % |
| LED Widget | V2 Status LED | Warna: hijau |
| LED Widget | V3 Status Buzzer | Warna: merah |

---

## 2. Kode: Konek Blynk `[0:20 – 0:40]`

### Install Library

Library Manager → search **Blynk** → by Volodymyr Shymanskyy → Install

> Pastikan Blynk **2.x** (bukan legacy 1.x)

### Kode Dasar Koneksi

```cpp
#define BLYNK_TEMPLATE_ID   "TMPLxxxxxx"
#define BLYNK_TEMPLATE_NAME "Monitor DHT"
#define BLYNK_AUTH_TOKEN    "YourAuthToken"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>

char ssid[] = "NAMA_WIFI";
char pass[] = "PASSWORD_WIFI";

DHT dht(4, DHT11);
BlynkTimer timer;

void setup() {
  Serial.begin(115200);
  dht.begin();
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
}

void loop() {
  Blynk.run();
  timer.run();
}
```

---

## 3. Kirim Data DHT ke Blynk `[0:40 – 1:00]`

```cpp
void kirimSensor() {
  float suhu       = dht.readTemperature();
  float kelembapan = dht.readHumidity();

  if (isnan(suhu) || isnan(kelembapan)) {
    Serial.println("Gagal baca sensor!");
    return;
  }

  Blynk.virtualWrite(V0, suhu);        // → Gauge Suhu di app
  Blynk.virtualWrite(V1, kelembapan);  // → Gauge Kelembapan di app

  Serial.print("Suhu: ");       Serial.println(suhu);
  Serial.print("Kelembapan: "); Serial.println(kelembapan);
}

// Di setup(), daftarkan timer:
timer.setInterval(2000L, kirimSensor);  // kirim tiap 2 detik
```

Buka Blynk app → angka suhu & kelembapan update otomatis di Gauge.

---

## 4. Aktuator — LED RGB + Buzzer `[1:00 – 1:40]`

### Kondisi & Aksi

| Kondisi Suhu | LED RGB | Blynk App |
|-------------|---------|-----------|
| < 28°C | Biru (sejuk) | LED widget biru |
| 28–35°C | Hijau (normal) | LED widget hijau |
| > 35°C | Merah (panas) | LED widget merah |

| Kondisi | Buzzer |
|---------|--------|
| Suhu > 38°C atau Kelembapan < 30% | ON (alarm) |
| Kondisi normal | OFF |

---

### Wiring LED RGB (Common Cathode)

```
R (merah)         →  GPIO 25  via resistor 220Ω
G (hijau)         →  GPIO 26  via resistor 220Ω
B (biru)          →  GPIO 27  via resistor 220Ω
GND (kaki panjang)→  GND
```

### Wiring Buzzer Aktif

```
VCC (+)  →  GPIO 18
GND (-)  →  GND
```

> Buzzer **aktif** langsung bunyi saat dapat sinyal HIGH.
> Buzzer **pasif** perlu fungsi `tone()` dengan frekuensi.

---

### Kode: Define Pin

```cpp
#define PIN_R    25
#define PIN_G    26
#define PIN_B    27
#define PIN_BUZZ 18

// di setup():
pinMode(PIN_R,    OUTPUT);
pinMode(PIN_G,    OUTPUT);
pinMode(PIN_B,    OUTPUT);
pinMode(PIN_BUZZ, OUTPUT);

// Matikan semua dulu:
digitalWrite(PIN_R,    LOW);
digitalWrite(PIN_G,    LOW);
digitalWrite(PIN_B,    LOW);
digitalWrite(PIN_BUZZ, LOW);
```

---

### Kode: Logika Aktuator + Update Blynk

```cpp
void kontrolAktuator(float suhu, float kelembapan) {

  // ── LED RGB ──────────────────────────────────
  if (suhu < 28.0) {                    // Sejuk → Biru
    digitalWrite(PIN_R, LOW);  digitalWrite(PIN_G, LOW);  digitalWrite(PIN_B, HIGH);
    Blynk.virtualWrite(V2, 1);
  } else if (suhu <= 35.0) {            // Normal → Hijau
    digitalWrite(PIN_R, LOW);  digitalWrite(PIN_G, HIGH); digitalWrite(PIN_B, LOW);
    Blynk.virtualWrite(V2, 255);
  } else {                              // Panas → Merah
    digitalWrite(PIN_R, HIGH); digitalWrite(PIN_G, LOW);  digitalWrite(PIN_B, LOW);
    Blynk.virtualWrite(V2, 255);
  }

  // ── Buzzer ───────────────────────────────────
  bool alarm = (suhu > 38.0) || (kelembapan < 30.0);
  digitalWrite(PIN_BUZZ, alarm ? HIGH : LOW);
  Blynk.virtualWrite(V3, alarm ? 255 : 0);  // update LED widget buzzer
}

// Panggil dari kirimSensor() setelah virtualWrite:
kontrolAktuator(suhu, kelembapan);
```

---

## 5. Kode Lengkap (Ringkasan)

```cpp
// ① Defines & includes
#define BLYNK_TEMPLATE_ID   "TMPLxxxxxx"
#define BLYNK_TEMPLATE_NAME "Monitor DHT"
#define BLYNK_AUTH_TOKEN    "YourAuthToken"
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>

// ② Pin & objek
DHT dht(4, DHT11);
BlynkTimer timer;
#define PIN_R 25  
#define PIN_G 26  
#define PIN_B 27  
#define PIN_BUZZ 18

// ③ Fungsi utama
void kirimSensor() {
  float s = dht.readTemperature(), h = dht.readHumidity();
  if (isnan(s) || isnan(h)) return;
  Blynk.virtualWrite(V0, s);
  Blynk.virtualWrite(V1, h);
  kontrolAktuator(s, h);
}

// ④ Setup & loop
void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(PIN_R, OUTPUT); pinMode(PIN_G, OUTPUT);
  pinMode(PIN_B, OUTPUT); pinMode(PIN_BUZZ, OUTPUT);
  Blynk.begin(BLYNK_AUTH_TOKEN, "SSID", "PASS");
  timer.setInterval(2000L, kirimSensor);
}

void loop() {
  Blynk.run();
  timer.run();
}
```

---

## 6. Bentuk Kelompok + Tentukan Project `[1:40 – 2:00]`

```
Nama Project   : _______________
Sensor         : DHT11 / DHT22
Aktuator       : LED RGB + Buzzer + ___
Trigger LED    :
  Biru  < ___ °C
  Hijau ___ – ___ °C
  Merah > ___ °C
Trigger Buzzer : Suhu > ___ °C / Kelembapan < ___ %
Monitoring     : Blynk (V0–V3)

Hardware       : _______________
Kode           : _______________
Presentasi     : _______________
```

### Ide Variasi Project

| Project | Aktuator Tambahan | Twist |
|---------|------------------|-------|
| 🌱 Smart Greenhouse | Relay pompa | LED merah + buzzer = darurat |
| 👶 Baby Room Monitor | - | LED biru = nyaman, buzzer > 32°C |
| 🖥 Server Room Alert | Relay kipas | Buzzer + notif Blynk > 40°C |
| 🏠 Smart Room Comfort | - | LED sesuai comfort score |

---

## Troubleshoot

| Error | Penyebab | Solusi |
|-------|----------|--------|
| `Blynk.begin()` stuck | SSID/password salah / WiFi 5GHz / Token salah | Cek kredensial, pakai hotspot 2.4GHz |
| Gauge tidak update | Datastream V0/V1 belum dibuat | Buat di Template → Datastreams |
| LED RGB satu warna saja | Common Anode/Cathode salah | Common Cathode: GND ke ground, Common Anode: logika terbalik |
| Buzzer tidak bunyi | Buzzer pasif perlu `tone()` | Pastikan pakai buzzer aktif, atau ganti ke `tone(PIN_BUZZ, 1000)` |
| "Template not found" | BLYNK_TEMPLATE_ID salah | Copy persis dari halaman Template di blynk.cloud |

---

## Recap Hari 2

- [x] Setup Blynk — Template, Datastream, Auth Token
- [x] Konek ESP32 ke Blynk via WiFi
- [x] Kirim suhu & kelembapan ke Gauge widget di HP real-time
- [x] LED RGB berubah warna otomatis sesuai kondisi suhu
- [x] Buzzer alarm otomatis + status tampil di Blynk app
- [x] Kelompok terbentuk + project dikonfirmasi

**Hari 3:** Finishing project + Demo hardware hidup + Presentasi kelompok
