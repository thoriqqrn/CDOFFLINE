# Hari 1 — ESP32 + DHT Sensor

> **Durasi:** 2 Jam | **Format:** Langsung Praktek

---

## Tujuan Besar — Project Akhir

```
DHT Sensor → ESP32 → Aktuator → ThingSpeak (Dashboard)
```

| Hari | Yang Dikerjakan |
|------|----------------|
| **Hari 1** | Wiring DHT, baca sensor, LCD, threshold LED |
| **Hari 2** | WiFi → ThingSpeak, pilih aktuator, bentuk kelompok |
| **Hari 3** | Finishing + presentasi kelompok |

---

## Kenapa ESP32 + DHT?

| ESP32 | DHT11 / DHT22 |
|-------|---------------|
| WiFi + Bluetooth built-in | Baca suhu & kelembapan |
| Dual-core 240 MHz | Single-wire protocol, cukup 3 kabel |
| Harga ~50–80 ribu | DHT11 ±2°C ~10rb / DHT22 ±0.5°C ~25rb |
| 40+ GPIO pin | Library siap pakai |
| Support Arduino IDE | Langsung plug & play |

**Bottom line:** 3 komponen + WiFi = sistem monitoring lingkungan yang bisa diakses dari browser. Itulah IoT.

---

## Alur Hari 1

```
[0:00–0:20]  Setup IDE + Library
      ↓
[0:20–0:50]  Wiring DHT → ESP32
      ↓
[0:50–1:20]  Upload Kode + Serial Monitor
      ↓
[1:20–1:40]  Output Visual — LCD I2C
      ↓
[1:40–2:00]  Threshold → LED Nyala Otomatis
```

---

## 1. Setup IDE + Library `[0:00 – 0:20]`

### 1.1 Install Arduino IDE

1. Download di **arduino.cc/en/software** — pilih versi 2.x
2. Install seperti biasa (Windows: .exe, Mac: drag ke Applications)
3. Buka Arduino IDE — pastikan editor kosong muncul

> Gunakan **Arduino IDE 2.x**, bukan 1.8. Autocomplete & library manager lebih baik.

---

### 1.2 Tambah Board ESP32

1. Buka **File → Preferences** (`Ctrl+,` / `Cmd+,`)
2. Kolom **"Additional boards manager URLs"** → paste:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

3. Buka **Boards Manager** (ikon papan di sidebar kiri)
4. Search: `esp32` → pilih by **Espressif Systems** → klik **Install** (tunggu ~1–3 menit)
5. **Tools → Board → ESP32 Arduino → ESP32 Dev Module**

---

### 1.3 Install Library

Buka **Library Manager** (ikon buku di sidebar / Tools → Manage Libraries):

| Library | Author | Keterangan |
|---------|--------|------------|
| `DHT sensor library` | Adafruit | Klik "Install All" saat ditanya dependencies |
| `LiquidCrystal I2C` | Frank de Brabander | Untuk LCD I2C |

---

### 1.4 Pilih Port + Test Upload

1. Hubungkan ESP32 via **kabel USB data** (bukan charge only)
2. **Tools → Port** → pilih port yang muncul
   - Windows: `COM3` / `COM4` / dst
   - Mac/Linux: `/dev/ttyUSB0` atau `/dev/cu.usbserial-xxx`
3. **File → Examples → 01.Basics → Blink** → Upload
4. LED di ESP32 kedip-kedip = board oke

> **Upload gagal?** Tahan tombol **BOOT** di board saat klik Upload, lepas setelah progress bar muncul.
> **Port tidak muncul?** Install driver CP2102 (Silicon Labs) atau CH340 sesuai chip USB di board.

---

## 2. Wiring DHT → ESP32 `[0:20 – 0:50]`

```
DHT VCC   ──→  3.3V    (pin 3V3 ESP32)
DHT GND   ──→  GND
DHT DATA  ──→  GPIO 4
               ↓
         Resistor 10kΩ pull-up dari DATA ke 3.3V
```

> DHT11 dan DHT22 wiring-nya sama persis. Perbedaan hanya di kode `#define DHTTYPE`.

---

## 3. Kode: Baca DHT ke Serial Monitor `[0:20 – 0:50]`

```cpp
#include <DHT.h>

#define DHTPIN   4
#define DHTTYPE  DHT11  // ganti DHT22 kalau pakai DHT22

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  float suhu       = dht.readTemperature();
  float kelembapan = dht.readHumidity();

  if (isnan(suhu) || isnan(kelembapan)) {
    Serial.println("Gagal baca sensor!");
    return;
  }

  Serial.print("Suhu: ");       Serial.print(suhu);        Serial.println(" °C");
  Serial.print("Kelembapan: "); Serial.print(kelembapan);  Serial.println(" %");
  delay(2000);
}
```

Buka **Tools → Serial Monitor** → set baud rate ke **115200**.

---

## 4. Output Visual — LCD I2C 16×2 `[0:50 – 1:20]`

### Wiring LCD I2C

```
SDA  ──→  GPIO 21
SCL  ──→  GPIO 22
VCC  ──→  5V
GND  ──→  GND
```

### Tambahan di Kode

```cpp
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// di setup():
lcd.init();
lcd.backlight();

// di loop(), setelah baca sensor:
lcd.clear();
lcd.setCursor(0, 0);
lcd.print("Suhu: ");
lcd.print(suhu);
lcd.print(" C");

lcd.setCursor(0, 1);
lcd.print("Lembap: ");
lcd.print(kelembapan);
lcd.print(" %");
```

> Alamat I2C biasanya `0x27` atau `0x3F`. LCD gelap? Putar trimpot biru di belakang modul.

---

## 5. Logika Threshold — LED Otomatis `[1:20 – 1:40]`

Preview aktuator → besok upgrade ke relay + kipas sungguhan.

```cpp
#define LED_PIN 2  // LED onboard ESP32

// di setup():
pinMode(LED_PIN, OUTPUT);

// di loop(), setelah baca sensor:
if (suhu > 35.0) {
  digitalWrite(LED_PIN, HIGH);  // suhu tinggi → LED nyala
} else {
  digitalWrite(LED_PIN, LOW);
}
```

**Coba:** tiup / pegang sensor → suhu naik → LED nyala otomatis.

Logika ini **sama persis** dengan relay. Besok cukup ganti:
```cpp
digitalWrite(LED_PIN, ...) → digitalWrite(RELAY_PIN, ...)
```

---

## Troubleshoot `[1:40 – 2:00]`

| Error | Penyebab | Solusi |
|-------|----------|--------|
| Serial tampil `NaN` / "Gagal baca sensor!" | Pull-up resistor tidak ada / kabel goyang | Cek resistor 10kΩ DATA→3.3V, cek kabel |
| Upload gagal "Failed to connect" | Board belum masuk mode download | Tahan tombol **BOOT** saat klik Upload |
| LCD gelap / tidak tampil | Kontras salah / alamat I2C salah | Putar trimpot, coba alamat `0x3F` |
| Port tidak muncul | Driver belum install | Install CP2102 atau CH340 driver |
| Suhu tidak wajar | DHT belum warmup | Delay pertama pakai `3000ms`, DHT11 ±2°C itu normal |

---

## Recap Hari 1

- [x] Setup Arduino IDE + Board ESP32 + Library
- [x] Wiring DHT11/DHT22 ke ESP32 (4 kabel)
- [x] Baca suhu & kelembapan real-time
- [x] Tampilkan data di LCD I2C 16×2
- [x] LED nyala otomatis saat suhu melebihi threshold

**Hari 2:** WiFi → ThingSpeak + Aktuator + Bentuk Kelompok
