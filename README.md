# Husqvarna Automower 220AC — Home Assistant integration

Komplett Home Assistant-paket för Husqvarna Automower 220AC (gen-2, ~2010–2012) som anslutits till HA via en ESP32-baserad [prozzerg/AMConnect](https://github.com/prozzerg/AMConnect)-modul.

Innehåller MQTT-sensorer, lawn_mower-entitet, väderstyrd hem-skickning, robust offline-hantering, sessionstidsbaserad batterihälsa-tracking och en färdig Mushroom-baserad Lovelace-dashboard.

## Förkrävt

**Hårdvara:**
- Husqvarna Automower 220AC (eller 230 ACX — samma gen-2-protokoll)
- ESP32 (LOLIN32 eller liknande) flashad med [prozzerg/AMConnect](https://github.com/prozzerg/AMConnect) firmware
- Optional: NEO-8M GPS-modul för positionsspårning

**Home Assistant:**
- HA 2024.6+ (testat på 2026.6)
- MQTT-broker konfigurerad (Mosquitto-add-on funkar)
- SMHI Weather-integration aktiverad (`weather.smhi_home`)
- Custom cards: `mushroom`, `mini-graph-card`, `hourly-weather`
- Mobile App integration för notiser

## Installation

### 1. Förbered MQTT-broker

ESP32:n publicerar på `AM220AC/*`-topics. Säkerställ att MQTT-brokern lyssnar på samma broker som HA.

### 2. Kopiera filerna

```
/config/
├── automower.yaml              # huvudpaketet med all mower-config
└── ui-lovelace.yaml            # eller lägg dashboarden via UI
```

I `configuration.yaml`, lägg till:

```yaml
homeassistant:
  packages:
    automower: !include automower.yaml

# notify-grupper används av paketet. owner = primär användare (alla
# mower-relaterade notiser går hit). household = alla mobiler i hushållet
# (oanvänt av detta paket men finns med som mönster för säkerhetsalarm).
notify:
  - name: owner
    platform: group
    services:
      - service: mobile_app_DIN_TELEFON
  - name: household
    platform: group
    services:
      - service: mobile_app_DIN_TELEFON
      # Lägg till fler mobiler för säkerhetsalarm
```

### 3. Anpassa entiteter

`notify.owner` och `notify.household` används för push-notiser. Byt ut `mobile_app_DIN_TELEFON` mot dina egna mobile_app-entitet (Settings → Devices & Services → Mobile App).

Om du inte använder SMHI, ändra `weather.smhi_home` till din väder-entitet i sektion 4 (template-sensorer) i `automower.yaml`.

### 4. Importera dashboarden

**Alternativ A — Via UI:** Settings → Dashboards → Add Dashboard → "From YAML" → klistra in innehållet i `automower_dashboard.yaml`.

**Alternativ B — Via YAML:** Lägg till i `ui-lovelace.yaml` eller motsvarande.

### 5. Starta om HA

Settings → System → Restart Home Assistant.

### 6. Initial-konfiguration

På mowerns fysiska menyer:
- **Klocka** — säkerställ rätt tid (Menu 9 eller liknande)
- **Schema** — sätt klipptider (Menu 2-2 vardag, dag-toggles)
- **Stationsavstånd** — om du har områden mowern har svårt att hitta

I HA-dashboarden:
- Klicka "Läs schemat från mower" för att initialisera schema-sensorerna

## Features

### Styrning
- Lawn_mower-entitet med start/paus/dock
- Robust hem-kommando med retry vid dålig WiFi (3× burst + 8 min retry-loop)
- Sammankopplad styrning via Auto/Hem/Stop-knappar

### Block-villkor
- 🌧 **Väderblock** (automatisk, styrs av SMHI: regn/frost/snö)
- 🖐 **Manuellt block** (du klickar, oberoende av väder)
- 🐕 **Hund ute** (du klickar)

Resume sker bara när ALLA tre är OFF.

### Sensorer
- Status (svensk översättning av tysk firmware-text)
- Batteri% (kalibrerat för NiMH 2200 mAh)
- Batterihälsa (peak-mAh tracking)
- Söker-hem-tröskel, laddtid, charging temperature
- Sessionslängd (filtrerar bort sessions med fel)
- Position (lat/long via GPS-modul)
- Klippmönster på karta
- 30 dygns statistik (snitt/min/max/antal)

### Larm
- 🤖 Stannat / fel (5 min debounce)
- 🆘 Mowern svarar inte (efter 10 min retry-misslyckande, 1h cooldown)
- ⚠️ Klipparen körde ut trots block (re-skickar hem)

### Batterihälsa-trend
- Sessionslängd-tracking med baseline-snapshot
- "Session-hälsa %" jämfört med baseline
- Automatisk filtrering av fel-sessions (festgefahren, fehler m.m.)
- Sanity-check 0.5–100 min (rejekt om utanför)

## Konfiguration

### MQTT-topics

| Topic | Riktning | Innehåll |
|---|---|---|
| `AM220AC/status` | Mower → HA | Aktuell status (tysk) |
| `AM220AC/location` | Mower → HA | GPS lat,long |
| `AM220AC/debug` | Mower → HA | Sensor-data, svar på get-kommandon |
| `AM220AC/rssi` | Mower → HA | WiFi RSSI |
| `AM220AC/lwt` | Mower → HA | Online/Offline |
| `AM220AC/command` | HA → Mower | Kommandon (get*, setMode*, etc.) |

### Justerbara parametrar

| Vad | Var i `automower.yaml` |
|---|---|
| Batterikalibrering | `AMConnect Battery Capacity %` (sektion 4) |
| Väderblock-villkor | `AMConnect Weather Stop Triggered` (sektion 4) |
| Retry-tider för hem-kommando | `script.amconnect_gps_home` (sektion 5) |
| Sessionslängd-max | `< 100` i trigger-sensorn (sektion 4) |
| Notis-cooldowns | `> 3600` i automationerna (sektion 6) |

## Kända begränsningar

Dessa är firmware/protokoll-begränsningar, inte fel i denna integration:

1. **Schema kan inte sättas från HA** — endast läsas. Sätt schemat på mowerns fysiska panel.
2. **Klock-drift kan inte detekteras** — 220AC:s protokoll-RTC är inte synkad med displayens klocka.
3. **Total drifttid kan inte läsas** — finns inget kommando i protokollet. Läs manuellt från mowerns display.
4. **WE1/WE2 timer-fält oanvända** — 220AC har bara ett schema (W1/W2) som gäller alla dagar.

## Troubleshooting

**`lawn_mower.automower_220ac` är "unknown"**
- Kontrollera att mower publicerar på `AM220AC/status` (MQTT Explorer hjälper)
- Kolla att ESP32:n är ansluten till WiFi och MQTT-broker

**Sensorer fyller inte på**
- Tryck manuellt på `script.update_amconnect_sensors` i Developer Tools → Actions
- Kontrollera `AM220AC/debug` topic för svar

**Schemat visar konstiga värden (t.ex. "08:2048")**
- Detta är en känd firmware-bugg där minuter packas med timme i samma byte
- Lösningen finns inbyggd via `% 256` i template-sensorerna

**Långa "fantom-sessions" registreras**
- Säkerställ att YAML är senaste versionen — `sensor.amconnect_gps_status` används för session-detection
- Tröskeln är 100 min — högre värden avvisas automatiskt

## Hur du läser mowerns interna scheman

På mowerns fysiska panel:
- **Menu 2-2** — klipptider för dag-passen (Pass 1, Pass 2)
- **Menu 2-3** — dag-toggles (vilka dagar mowern ska klippa)
- **Stationsavstånd (Menu 4 eller 5)** — släpp-punkter längs guide/begränsningskabel

## Credits

- Hårdvara/ESP32-firmware: [prozzerg/AMConnect](https://github.com/prozzerg/AMConnect)
- Alternativ ESPhome-firmware: [gbrd/esphome-husqvarna-automower-220ac](https://github.com/gbrd/esphome-husqvarna-automower-220ac)
- Protokoll-reverse-engineering: tysk Automower-community (komplett kommandolista i gbrd:s repo)
- Blog-post med setup-guide: [iambobbytables: Making an old Automower smartier](https://iambobbytables.com/2024/03/06/making-and-old-automower-smartier/)

## Licens

MIT — använd, modifiera och dela fritt. Inga garantier — om du gör nåt korkat och brickar mowern är det inte mitt fel.

## Bidrag

PRs välkomna. Speciellt intresserad av:
- Bekräftelse att paketet funkar med 230 ACX (samma generation)
- Förbättringar av sessions-detektionen
- Stöd för fler firmware-forks (dixie007 m.fl.)
