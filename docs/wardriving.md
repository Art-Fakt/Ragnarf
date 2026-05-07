# Wardriving — Ragnar + HuginnESP

## Översikt

Ragnars wardriving-motor samlar WiFi-nätverk, BLE-enheter, mobilmaster och GPS-positioner under körning. Data lagras i SQLite per session och kan exporteras till WiGLE CSV eller KML.

Systemet har två datakällor:
- **WiFi-adaptrar (wlan)** — Linux-adaptrar i monitor/managed mode, skannar direkt via `iw`
- **HuginnESP** — ESP32-S3 via USB serial, skannar WiFi + BLE + hot-detektion

Allt som loggas får automatiskt GPS-koordinater om en GPS-mottagare är ansluten.

---

## Hårdvara

### HuginnESP (ESP32-S3)

| Egenskap | Värde |
|----------|-------|
| Kort | Waveshare ESP32-S3 Smart 86 Box |
| Display | 4" 480×480 RGB IPS, GT911 touch (I2C) |
| Processor | ESP32-S3, 240 MHz |
| Flash | 16 MB |
| PSRAM | 8 MB OPI |
| Serial | USB CDC, 115200 baud |
| Firmware | [HuginnESP](https://github.com/PierreGode/HuginnESP) (PlatformIO Arduino) |
| Bibliotek | LovyanGFX 1.1.16, NimBLE-Arduino 1.4.1 |

### GPS

Valfri USB GPS-mottagare (NMEA via pyserial). Auto-detekteras vid start.

---

## Arkitektur

```
┌─────────────────────┐     USB Serial      ┌──────────────────┐
│  Raspberry Pi / PC  │◄───────────────────►│   HuginnESP      │
│                     │   115200 baud        │   ESP32-S3       │
│  Ragnar             │                      │                  │
│  ├─ wardriving.py   │   JSON + alerts      │  ├─ WiFi scan    │
│  ├─ webapp_modern.py│◄────────────────────│  ├─ BLE scan     │
│  └─ web UI          │                      │  ├─ AirTag det.  │
│                     │   Kommandon          │  ├─ Flipper det. │
│  GPS ◄──────────┐   │──────────────────►  │  ├─ Skimmer det. │
│  (USB NMEA)     │   │   scanap, blescan   │  └─ Pineapple    │
│                 ▼   │                      │                  │
│  SQLite session DB  │                      │  480×480 display │
└─────────────────────┘                      └──────────────────┘
```

---

## HuginnESP Serial-protokoll

### Kommandon (Ragnar → ESP)

| Kommando | Beskrivning |
|----------|-------------|
| `scanap` | Starta WiFi-skanning |
| `blescan -f` | BLE filtered (Flipper + AirTag) |
| `blescan -a` | BLE all (alla enheter + hot-detektion) |
| `capture -skimmer` | BLE skimmer-detektion |
| `pineap` | Pineapple/Evil Twin-detektion |
| `stop` | Stoppa aktiv skanning, återgå till auto-cykel |
| `capture -stop` | Stoppa BLE capture |
| `status` | Hämta aktuellt läge (JSON) |

### Scan-cykel

Ragnar roterar genom dessa steg:

| Steg | Kommando | Duration | Mode-etikett |
|------|----------|----------|--------------|
| 1 | `scanap` | 15s | wifi |
| 2 | `blescan -f` | 8s | ble-filtered |
| 3 | `scanap` | 15s | wifi |
| 4 | `blescan -a` | 8s | ble-all |
| 5 | `scanap` | 15s | wifi |
| 6 | `capture -skimmer` | 8s | ble-skimmer |
| 7 | `scanap` | 15s | wifi |
| 8 | `pineap` | 10s | pineap |

Total cykel: ~94 sekunder.

### Serial-output (ESP → Ragnar)

#### WiFi-nätverk (JSON, en rad per AP)
```json
{"type":"WIFI","mac":"38:E1:F4:F6:79:26","ssid":"Tele2_F67926","rssi":-84,"channel":1,"auth":"WPA2"}
```

#### BLE-enheter (JSON, ALL-mode, en rad per enhet)
```json
{"type":"BLE","mac":"AA:BB:CC:DD:EE:FF","name":"DeviceName","rssi":-60}
```

#### AirTag (multi-line, FILTERED + ALL mode)
```
AirTag found!
Tag: 1
MAC Address: D3:59:9B:E4:2C:3A
RSSI: -89
```

#### Flipper Zero (multi-line, FILTERED + ALL mode)
```
Found White Flipper Device:
MAC: AA:BB:CC:DD:EE:FF,
Name: Flipper-X,
RSSI: -70
```

#### Skimmer (multi-line, SKIMMER + ALL mode)
```
POTENTIAL SKIMMER DETECTED!
Device Name: HC-05
MAC Address: 11:22:33:44:55:66
RSSI: -55
Reason: Suspicious BLE module near payment terminal
```

Kända skimmer-namn: `HC-05`, `HC-06`, `HC-08`, `BT05`, `BT06`, `JDY-30`, `JDY-31`, `JDY-33`, `SPP-CA`

#### Pineapple/Evil Twin (multi-line)
```
Pineapple detected: NetworkName
BSSID: AA:BB:CC:DD:EE:FF
Channel: 6
```

#### BLE Spam (en rad)
```
BLE Spam detected from AA:BB:CC:DD:EE:FF
```
Triggas vid 20+ advertisements från samma MAC inom 5 sekunder.

#### Status (JSON, svar på `status`-kommando)
```json
{"mode":"auto","wifi_count":5,"ble_count":12}
```

#### Boot-meddelanden
```
[BOOT] HuginnESP starting...
[BOOT] Free heap: 234567
[BOOT] PSRAM: 8388608
[BOOT] Init WiFi...
[BOOT] WiFi OK
[BOOT] Init BLE...
[BOOT] BLE OK
[BOOT] All tasks started — entering main loop
```

---

## Ragnar Parser

`wardriving.py → _parse_serial_line()` hanterar all output:

| Datatyp | Detektering | Lagring | GPS |
|---------|-------------|---------|-----|
| WiFi JSON | `line.startswith('{')` → `type == WIFI` | `upsert_network()` | ✅ |
| BLE JSON | `line.startswith('{')` → `type == BLE` | `upsert_bluetooth()` | ✅ |
| AirTag | `line.startswith('AirTag found')` → buffer | `upsert_bluetooth('AirTag')` | ✅ |
| Flipper | `re.match('Found .* Flipper')` → buffer | `upsert_bluetooth('Flipper')` | ✅ |
| Skimmer | `'POTENTIAL SKIMMER' in line` → buffer | `upsert_bluetooth('Skimmer')` | ✅ |
| Pineapple | `'Pineapple detected' in line` | `_esp_alerts` lista | ✅ |
| BLE Spam | `'BLE Spam detected' in line` | `_esp_alerts` lista | ✅ |
| WiGLE CSV | Komma-separerad med MAC-format | `upsert_network()` / `upsert_bluetooth()` | ✅ |
| Multi-line WiFi | `[N] SSID: ...` → buffer | `upsert_network()` | ✅ |

Rader som ignoreras:
- `huginn>`, `Wardrive:`, `Registered`, `Unsupported`
- `WiFi scan`, `Started`, `Stopped`, `Usage:`, `BLE initialized`, etc.
- `[BOOT]`-prefix (ej explicit men matchas inte av någon parser)

---

## API-endpoints

| Metod | Endpoint | Beskrivning |
|-------|----------|-------------|
| GET | `/api/wardriving/status` | Fullständig status inkl. GPS, serial, räknare |
| POST | `/api/wardriving/start` | Starta wardriving-session |
| POST | `/api/wardriving/stop` | Stoppa session |
| GET | `/api/wardriving/networks` | Lista fångade WiFi-nätverk |
| GET | `/api/wardriving/bluetooth` | Lista BLE-enheter |
| GET | `/api/wardriving/cells` | Lista mobilmaster |
| GET | `/api/wardriving/sessions` | Lista sessioner |
| GET | `/api/wardriving/export/<id>` | Exportera session (WiGLE CSV / KML) |
| POST | `/api/wardriving/import` | Importera WiGLE CSV |
| GET | `/api/wardriving/gps` | GPS-status |
| GET | `/api/wardriving/interfaces` | Tillgängliga WiFi-gränssnitt |
| GET | `/api/wardriving/serial/detect` | Auto-detektera ESP32 port |
| GET/POST | `/api/wardriving/serial` | Serial-status / starta/stoppa listener |
| GET | `/api/wardriving/track` | GPS-spår (lat/lon historik) |
| POST | `/api/wardriving/device_name` | Sätt enhetsnamn |
| GET/POST | `/api/wardriving/on_boot` | Auto-start vid boot |

---

## Status-objekt

`GET /api/wardriving/status` returnerar:

```json
{
  "running": true,
  "session_id": "uuid",
  "interfaces": ["wlan1"],
  "scans_completed": 42,
  "total_networks": 156,
  "gps": {
    "connected": true,
    "port": "/dev/ttyACM0",
    "has_fix": true
  },
  "serial_connected": true,
  "serial_port": "/dev/ttyACM1",
  "serial_networks": 23,
  "esp_mode": "wifi",
  "esp_ble_count": 87,
  "esp_alerts": [
    {"time": 1683456789, "alert": "AirTag found!"}
  ],
  "serial_unique": 12,
  "bluetooth_count": 87,
  "cell_count": 3,
  "stats": { ... }
}
```

---

## Datalagring

Varje session skapar en SQLite-databas i `data/networks/`.

### Tabell: networks
| Kolumn | Typ | Beskrivning |
|--------|-----|-------------|
| bssid | TEXT | MAC-adress (primärnyckel) |
| ssid | TEXT | Nätverksnamn |
| security | TEXT | WPA2, WPA3, Open, etc. |
| channel | INT | WiFi-kanal |
| frequency | INT | Frekvens i MHz |
| rssi | INT | Signalstyrka (dBm) |
| lat | REAL | GPS latitud |
| lon | REAL | GPS longitud |
| alt | REAL | GPS altitud |
| interface | TEXT | `wlan1`, `esp32-serial`, etc. |

### Tabell: bluetooth
| Kolumn | Typ | Beskrivning |
|--------|-----|-------------|
| mac | TEXT | BLE MAC-adress |
| name | TEXT | Enhetsnamn |
| rssi | INT | Signalstyrka |
| device_type | TEXT | `BLE`, `AirTag`, `Flipper`, `Skimmer` |
| lat/lon/alt | REAL | GPS-position |

---

## Export

### WiGLE CSV
Standardformat för uppladdning till wigle.net. Innehåller MAC, SSID, AuthMode, kanal, RSSI, GPS-koordinater.

### KML
Google Earth-format med nätverkspositioner som markörer.

---

## Setup

### 1. Flasha HuginnESP

```bash
cd HuginnESP
pio run --target upload     # COM8 på Windows, /dev/ttyACM* på Linux
```

### 2. Anslut till Ragnar

**Auto-detect (Linux):**
Klicka 🔍 Search i web UI → hittar ESP32 automatiskt via `udevadm`.

**Manuellt:**
Ange port (`/dev/ttyACM0` eller `COM8`) i serial-fältet och klicka Connect.

### 3. GPS (valfritt)

Anslut USB GPS-mottagare. Ragnar auto-detekterar NMEA-enheter.

### 4. Starta wardriving

Klicka **Start Wardriving** i web UI. Ragnar börjar skanna med alla aktiva gränssnitt + ESP32 serial.

---

## Kameraigenkänning

Ragnar identifierar övervakningskameror baserat på MAC OUI-prefix (tillverkare):
Axis, Hikvision, Dahua, Vivotek, Bosch, Samsung, Reolink, Amcrest, Foscam m.fl.

Kameror markeras i nätverkslistan med typ och tillverkare.
