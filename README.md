# Ulanzi TC001 als Fronius-Solaranzeige (ESPHome)

Verwandle den **Ulanzi TC001 Pixel Clock** (ESP32, 32×8-LED-Matrix) in eine
eigenständige **Solar-Anzeige für Fronius-Wechselrichter**:

- Fragt den Wechselrichter direkt über dessen **Solar-API** im lokalen
  Netzwerk ab – **kein Home Assistant, kein Raspberry Pi, keine Cloud**.
- Läuft komplett standalone: Ulanzi an ein USB-Netzteil, fertig.
- Alle Einstellungen (IPs, Abfrageintervall, Helligkeit, Schrift, …) sind
  bequem **im Browser** änderbar und überleben Neustarts.


---

## Inhaltsverzeichnis

1. Funktionen
2. Benötigte Hardware
3. Wie funktioniert es?
4. Installation
5. Konfiguration
6. Erstes Flashen & Updates
7. Bedienung am Gerät
8. Einstellungen in der Web-UI
9. Die Anzeigemodi im Detail
10. IP-Logik: Standard & Fallback
11. Auto-Helligkeit kalibrieren
12. Störungsanzeige: WAIT / OFF
13. Problemlösung
14. FAQ
15. Projektstruktur
16. Datenschutz
17. Lizenz

---

## Funktionen

### Anzeige

- **5 Anzeigemodi** mit einstellbarer automatischer Rotation:
   Modus | Anzeige | Farbe |
 |---|---|---|
 | Netz | Strombezug (`230W`) bzw. Einspeisung (`1.2kW`) | rot beim Bezug, grün bei Einspeisung |
 | Verbrauch | momentaner Hausverbrauch | cyan |
 | PV | aktuelle Erzeugung der PV-Anlage | gelb |
 | Batterie | Ladestand des Speichers in Prozent | grau |
 | Uhr | Uhrzeit `HH:MM` | weiß |

- Ab **1000 Watt** wird automatisch auf kW-Anzeige umgeschaltet
  (`1.2kW` statt `1234W`), damit alles auf die 32 Pixel passt.
- **PV-Modus nachts automatisch deaktiviert:** Die Rotation überspringt
  den PV-Modus, sobald die Sonne unter dem Horizont steht – kein
  müdes `0W` die halbe Nacht lang.
- **Uhr optional:** Der Uhr-Modus lässt sich in der Web-UI ausblenden
  (Rotation, Tasten und Dropdown passen sich automatisch an).
- Bis zur ersten Zeitsynchronisation zeigt die Uhr `--:--`.

### Komfort

- **Alles im Browser einstellbar** (Web-UI, alle Werte überleben
  Neustart/Update):
  Fronius-IPs, Abfrageintervall, Wechselrichter-Nr., Auto-Rotation,
  Rotationsintervall, Helligkeit, Schriftstil, Textposition u. v. m.
- **Automatische IP-Umschaltung:** Antwortet die Standard-IP des
  Wechselrichters nicht, wird automatisch die Fallback-IP genutzt – und
  wieder zurückgeschaltet, sobald die Standard-IP erreichbar ist.
- **Automatische Helligkeit** über den eingebauten Helligkeitssensor (LDR),
  individuell kalibrierbar (siehe unten).
- **IP-Laufschrift:** Nach dem ersten WLAN-Verbindungsauflauf läuft die
  IP-Adresse des Geräts einmal über die Matrix (~20 Sekunden) – man sieht
  sofort, wo das Gerät im Netzwerk zu finden ist. Auch manuell startbar.
- **4 Schriftstile** für die Matrix, per Web-UI umschaltbar:
 | Schriftstil | Beschreibung |
 |---|---|
 | Silkscreen (klein) | kompakter Pixel-Look |
 | Handgezeichnet 5×8 | nutzt die volle Höhe der Matrix |
 | Roboto Mono | groß und klar, sehr gut lesbar |
 | Klassisch 5×7 | der Standard-Retro-Look |

- **Tastensteuerung** am Gerät (siehe Bedienung).
- **Status-Anzeige bei Störungen:** `WAIT` (kurz nicht erreichbar) bzw.
  `OFF` (nach 3 Fehlversuchen) – siehe
  Störungsanzeige.

Getestet mit **ESPHome 2026.10**.

---

## Benötigte Hardware

- **[Ulanzi TC001 Pixel Clock](https://www.ulanzi.de/products/ulanzi-pixel-clock-2882)**
  (32×8 WS2812-LEDs, ESP32, eingebaute 3 Tasten, Helligkeitssensor,
  Mikrofon – wird hier nicht genutzt)
- **Fronius-Wechselrichter** mit Solar-API, z. B. GEN24, Symo, Primo,
  Galvo – mit oder ohne Speicher. Der Wechselrichter muss über WLAN
  oder ein Netzwerkkabel im selben Netzwerk erreichbar sein.
- **USB-C-Kabel** (für das erste Flashen) und ein **USB-Netzteil**
  für den Betrieb
- WLAN-Netzwerk, in dem Ulanzi und Wechselrichter sich „sehen“

---

## Wie funktioniert es?

Der Ulanzi fragt den Wechselrichter selbstständig per HTTP ab:

```
Ulanzi TC001  ──HTTP GET──▶  Fronius-Wechselrichter
     │                        http://<IP>/solar_api/v1/
     │                        GetPowerFlowRealtimeData.fcgi
     ◀──── JSON-Antwort ──────
```

Die Solar-API liefert alle relevanten Werte in einer JSON-Struktur:
PV-Erzeugung (`PV`), Netzbezug/-einspeisung (`Grid`), Hausverbrauch
(`Load`) und – falls vorhanden – den Ladestand des Speichers
(`BAT`/`Storage`). Aus dieser Antwort werden die angezeigten Werte
abgeleitet.

Der Quellcode der Logik steckt komplett in der einen Datei
`ulanzi-fronius.yaml`. Es wird **nur das lokale Netzwerk** benutzt –
kein Cloud-Dienst, keine externe Abhängigkeit.

---

## Installation

### 1. ESPHome installieren

**Windows (PowerShell)** – empfohlen mit Python 3.14:

```powershell
py -3.14 -m pip install esphome
py -3.14 -m esphome version
```

> **Hinweis:** Aktuell ist Python 3.14 die zuverlässig funktionierende
> Version für ESPHome 2026.10; mit Python 3.15 schlägt die Installation
> von Abhängigkeiten fehl.

**Alternativen:** ESPHome als Home-Assistant-Add-on oder als
Docker-Container – die `ulanzi-fronius.yaml` funktioniert überall gleich.

### 2. Projektdateien herunterladen

Alle Dateien dieses Repos herunterladen (Code → Download ZIP) oder
klonen. Wichtig ist nur `ulanzi-fronius.yaml`.

### 3. YAML-Konfiguration

**Bevor du kompilierst**, lies den Abschnitt
Konfiguration – dort steht, was du ausfüllen musst.

### 4. Konfiguration prüfen und kompilieren

```powershell
py -3.14 -m esphome config .\ulanzi-fronius.yaml     # nur Pruefung
py -3.14 -m esphome compile .\ulanzi-fronius.yaml   # Firmware bauen
```

Läuft beides ohne Fehler durch, geht es zum
ersten Flashen.

---

## Konfiguration

Öffne `ulanzi-fronius.yaml` und fülle **nur den Block `substitutions:`**
am Dateianfang aus – der Rest bleibt unverändert:
 | Platzhalter | Bedeutung | Beispiel |
 |---|---|---|
 | `wifi_ssid` | Name (SSID) des eigenen WLAN | `MeinWLAN` |
 | `wifi_password` | WLAN-Passwort | `geheim` |
 | `ap_password` | Passwort des Notfall-Hotspots (SSID `Ulanzi-Fronius-Setup`), falls das WLAN einmal nicht erreichbar ist | `12345678` |
 | `fronius_ip` | Standard-IP-Adresse des Wechselrichters | `192.168.1.5` |
 | `latitude` | Breitengrad des Standorts (für den Sonnenstand) | `52.5208` |
 | `longitude` | Längengrad des Standorts | `13.4049` |

> **⚠️ Sonderzeichen im WLAN-Passwort:** Enthält das Passwort ein
> `$`, muss es in der YAML als `$$` geschrieben werden
> (`Pa$$wort` → `Pa$$wort` → YAML: `Pa$$$$wort`).
> Praktisch: einfach **jedes** `$` im Passwort verdoppeln.

Die Koordinaten (`latitude`/`longitude`) braucht die Firmware für den
Sonnenstand: Nur damit weiß sie zuverlässig, wann es Nacht ist und der
PV-Modus übersprungen werden kann. Wer keine exakten Werte angeben will,
nutzt die Werte der nächsten Stadt – auf die Minute kommt es nicht an.

**Zweites WLAN als Backup ergänzen** (optional):

```yaml
wifi:
  networks:
    - ssid: "${wifi_ssid}"
      password: "${wifi_password}"
    - ssid: "ZWEITES-WLAN"
      password: "ZWEITES-PASSWORT"   # auch hier: $ -> $$
```

Das Gerät verbindet sich dann mit dem zuerst erreichbaren Netz –
praktisch, wenn der Wechselrichter z. B. im Gästenetzwerk steht oder
der Ulanzi zwischen Wohnungs- und Garten-WLAN wechseln soll.

> **Nachtrag ohne Neukompilierung:** Alle weiteren Werte – Fronius-IPs,
> Abfrageintervall, Wechselrichter-Nr., Helligkeit, Schriftstil usw. –
> werden **in der Web-UI** geändert und brauchen kein erneutes Flashen.

---

## Erstes Flashen & Updates

### Erstes Flashen (per USB-Kabel)

```powershell
py -3.14 -m esphome run .\ulanzi-fronius.yaml
```

Den seriellen Port des Ulanzi auswählen (unter Windows meist
`COM3`, `COM5` o. Ä.; in der Geräteverwaltung nachsehen).

> **Boot-Modus:** Startet das Flashen nicht von selbst, den Ulanzi bei
> **gedrückter mittlerer Taste** mit dem USB-Kabel verbinden und den
> Befehl erneut ausführen.

Nach dem Start verbindet sich das Gerät mit dem WLAN und ist unter

- `http://ulanzi-fronius.local` oder
- der IP-Adresse, die beim ersten Start **als Laufschrift** über die
  Matrix läuft

im Browser erreichbar.

### Spätere Updates (per WLAN / OTA)

1. In der Web-UI die neueste `ulanzi-fronius.yaml` kompilieren.
2. In der Web-UI des Geräts → **Installation / OTA** → die fertige
   `.bin` hochladen.

Alle Einstellungen und gespeicherten Werte (IPs, Intervalle,
Helligkeit …) bleiben dabei erhalten.

---

## Bedienung am Gerät

Der Ulanzi hat drei Tasten auf der Oberseite:
 | Taste | Funktion |
 |---|---|
 | **Links** | vorheriger Anzeigemodus (schaltet die Auto-Rotation aus) |
 | **Rechts** | nächster Anzeigemodus (schaltet die Auto-Rotation aus) |
 | **Mitte kurz** | Auto-Rotation ein/aus |
 | **Mitte lang (≥ 1 s)** | Display ganz ein/aus (z. B. nachts) |

Alle Funktionen sind zusätzlich in der Web-UI verfügbar.

> **Hinweis:** Nach jedem Start sind Auto-Rotation und Auto-Helligkeit
> bewusst **eingeschaltet** – so verhält sich das Gerät nach einem
> Stromausfall wieder wie gewohnt. Alle anderen Einstellungen bleiben
> genau so, wie du sie zuletzt hattest.

---

## Einstellungen in der Web-UI

Die Weboberfläche ist in drei Gruppen unterteilt.

### Gruppe „Anzeige"
 | Einstellung | Bereich / Werte | Bedeutung |
 |---|---|---|
 | Anzeigemodus | Netz / Verbrauch / PV / Batterie / Uhr | Modus direkt wählen; schaltet die Auto-Rotation aus |
 | Uhr anzeigen | an / aus | blendet den Uhr-Modus überall ein oder aus |
 | Auto-Rotation | an / aus | wechselt automatisch zwischen den Modi |
 | Rotationsintervall | 1–30 s (Standard **4 s**) | Dauer pro Modus bei Auto-Rotation |
 | Auto-Helligkeit | an / aus | passt die LED-Helligkeit ans Umgebungslicht an |
 | Manuelle Helligkeit | 0.05–1.0 (Standard **0.5**) | gilt, wenn Auto-Helligkeit aus ist |
 | Lichtwert dunkel (V) | 0–3.3 | Kalibrierwert „dunkel" des Helligkeitssensors |
 | Lichtwert hell (V) | 0–3.3 | Kalibrierwert „hell" des Helligkeitssensors |
 | Schriftstil | 4 Stile | siehe Funktionen |
 | Text-Position Y | 0–7 | vertikale Ausrichtung des Textes auf der Matrix |

### Gruppe „Fronius Werte"
 | Einstellung | Bereich / Werte | Bedeutung |
 |---|---|---|
 | PV-Erzeugung / Netz / Hausverbrauch / Batterie | Live-Werte | aktuelle Messwerte aus der Solar-API (nur Anzeige) |
 | **Fronius IP (Standard)** | IP-Adresse | wird immer zuerst abgefragt; eine Änderung wirkt sofort |
 | **Fronius IP (Fallback)** | IP-Adresse | wird nur genutzt, wenn die Standard-IP nicht antwortet |
 | **Abfrageintervall** | 3–60 s (Standard **5 s**) | wie oft der Wechselrichter abgefragt wird |
 | **Wechselrichter-Nr.** | 0–9 (Standard **1**) | Nummer des Wechselrichters in der Solar-API; relevant für den Batterie-SOC bei mehreren Geräten |

### Gruppe „Diagnose"
 | Einstellung | Bedeutung |
 |---|---|
 | Umgebungslicht | aktueller Spannungswert des Helligkeitssensors – Grundlage der Kalibrierung |
 | IP-Adresse | aktuelle IP-Adresse des Ulanzi im WLAN |
 | Aktive Fronius IP | die gerade abgefragte IP (Standard oder Fallback) |
 | IP-Laufschrift starten | zeigt die IP-Adresse nochmals als Laufschrift an |

---

## Die Anzeigemodi im Detail

- **Netz:** Zeigt den Austausch mit dem Stromnetz. Bezieht das Haus
  Strom (positiver Bezug), leuchtet die Zahl **rot**; speist die Anlage
  ein (negativer Bezug), leuchtet sie **grün**. So siehst du auf einen
  Blick, ob du gerade Strom kaufst oder verkaufst.
- **Verbrauch:** Der Gesamthausverbrauch – unabhängig davon, ob der
  Strom aus PV, Batterie oder Netz kommt. Ideal, um den Momentanverbrauch
  im Blick zu behalten.
- **PV:** Die aktuelle Erzeugung der PV-Anlage. Nach Sonnenuntergang
  wird dieser Modus in der Rotation automatisch übersprungen.
- **Batterie:** Der Ladestand des Speichers in Prozent. Ohne Speicher
  bzw. bei falscher Wechselrichter-Nr. bleibt die Anzeige leer/`0%`.
  Hast du nur einen Wechselrichter, ist die Standard-Nummer **1** korrekt.
- **Uhr:** Die Uhrzeit (`HH:MM`), bis die Zeitsynchronisation steht
  `--:--`. Für die Synchronisation braucht das Gerät Internet-Zugang
  (NTP über das Gateway/WLAN).

---

## IP-Logik: Standard & Fallback

Viele Wechselrichter sind über zwei IPs erreichbar (z. B. über den
Fronius-Access-Point und über das Heimnetzwerk) oder die IP ändert sich
nach einem Router-Neustart. Damit das Display trotzdem zuverlässig
arbeitet:

1. Es wird immer die **Standard-IP** abgefragt (Vorgabewert aus der
   YAML, in der Web-UI änderbar – Änderungen wirken sofort).
2. Schlägt die **Verbindung** fehl (Wechselrichter aus, falsche IP,
   Netzwerkprobleme), wird automatisch die **Fallback-IP** probiert.
3. Antwortet die aktive IP nicht mehr, wird wieder zurückgewechselt –
   in beide Richtungen, ohne dass du etwas tun musst.

Welche IP gerade aktiv ist, steht in der Gruppe „Diagnose"
(**Aktive Fronius IP**).

> Die Umschaltung passiert nur bei **Verbindungsfehlern**. Antwortet der
> Wechselrichter mit gültigen Daten (auch mit „0-Werte"-Daten), bleibt
> die aktive IP unverändert.

---

## Auto-Helligkeit kalibrieren

Der eingebaute Helligkeitssensor (LDR) liefert eine Spannung, die mit
mehr Licht **steigt**. Die Firmware glättet den Wert, damit kurze
Schatten (vorübergehende Wolken, jemand läuft vorbei) die Anzeige nicht
flackern lassen.

> **Wichtig:** „Lichtwert hell" muss **größer** sein als
> „Lichtwert dunkel" – eine automatische Richtungserkennung gibt es
> nicht. Passt die Reihenfolge nicht, greift die Automatik nicht.

So kalibrierst du das Display einmalig:

1. In der Web-UI (Gruppe „Diagnose") den Sensorwert **Umgebungslicht**
   beobachten.
2. **Abends / im dunklen Raum:** Wert ablesen und bei
   **Lichtwert dunkel (V)** eintragen.
3. **Am hellen Tag:** Wert ablesen und bei **Lichtwert hell (V)**
   eintragen.
4. **Auto-Helligkeit** einschalten.

Unter „Lichtwert dunkel" fährt das Display auf Mindesthelligkeit
(leuchtet aber noch sichtbar), oberhalb von „Lichtwert hell" läuft es
auf volle Helligkeit; dazwischen wird stufenlos interpoliert.

---

## Störungsanzeige: WAIT / OFF
 | Anzeige | Farbe | Bedeutung |
 |---|---|---|
 | `WAIT` | gelb | Der Wechselrichter ist gerade **kurz** nicht erreichbar (eine fehlgeschlagene Abfrage). Das Display versucht es beim nächsten Intervall erneut bzw. über die Fallback-IP. |
 | `OFF` | rot | Der Wechselrichter ist seit **3 Abfragen in Folge** nicht erreichbar – typischerweise nachts oder bei Netzproblemen. |

Sobald wieder gültige Daten empfangen werden, verschwindet die
Störungsanzeige von selbst und die Modi laufen wieder normal.

---

## Problemlösung
 | Problem | Lösung |
 |---|---|
 | Display zeigt dauerhaft `OFF` / `WAIT` | „Aktive Fronius IP" prüfen: Erreichtst du diese IP im Browser (`http://<IP>/solar_api/v1/GetPowerFlowRealtimeData.fcgi`)? Falls nicht: Standard-IP und Fallback-IP in der Web-UI korrigieren. |
 | Batterie zeigt `0%` oder gar nichts | „Wechselrichter-Nr." prüfen (Standard 1) – oder der Anlage fehlt ein Speicher. |
 | Uhr zeigt dauerhaft `--:--` | Das Gerät braucht Internet für die Zeitsynchronisation (NTP). Gateway/DNS im Router prüfen. Direkt nach dem Start kann die Synchronisation 1–2 Minuten dauern. |
 | Anzeige zu dunkel / zu hell | Auto-Helligkeit kalibrieren oder auf manuelle Helligkeit umschalten. |
 | Kein Zugriff auf die Web-UI | IP über die **IP-Laufschrift** ablesen (Diagnose → „IP-Laufschrift starten" bzw. Neustart) oder im Router nach dem Gerät suchen. Klappt WLAN gar nicht: Notfall-Hotspot `Ulanzi-Fronius-Setup` (Passwort aus `ap_password`) öffnen. |
 | Werte aktualisieren sich langsam | „Abfrageintervall" herauf-/heruntersetzen (3–60 s). |
 | PV-Modus fehlt in der Rotation | Nach Sonnenuntergang ist das normal – er wird automatisch übersprungen. Tagsüber prüfen, ob die Koordinaten in der YAML stimmen. |
 | Compile-Fehler `light_id` o. Ä. | ESPHome-Version prüfen: benötigt wird **2026.10 oder neuer** (`py -3.14 -m esphome version`). |
 | Flashen per USB klappt nicht | Ulanzi bei gedrückter mittlerer Taste anschließen (Boot-Modus), richtigen COM-Port wählen. |

---

## FAQ

**Brauche ich Home Assistant?**
Nein. Das Gerät arbeitet komplett eigenständig. Wer Home Assistant
nutzt, kann es aber trotzdem einbinden, da die Firmware die
ESPHome-API bereitstellt – die Sensoren und Schalter tauchen dort
automatisch auf.

**Funktioniert das auch ohne Speicher / ohne Batterie?**
Ja. Der Batterie-Modus zeigt dann einfach `0%` bzw. bleibt leer –
alle anderen Modi funktionieren normal.

**Kann ich mehrere Ulanzi betreiben?**
Ja, aber vergib jedem Gerät einen eigenen Namen (`name:`) und damit
eigene Hostname/Web-UI-Adresse, sonst stoßen sich die Devices im
Netzwerk (`ulanzi-fronius-kueche.local` o. Ä.).

**Verbrennt die Firmware meine Einstellungen bei einem Update?**
Nein. Alle per Web-UI gesetzten Werte werden dauerhaft gespeichert
und überstehen Neustart und OTA-Update. Nur Auto-Rotation und
Auto-Helligkeit sind nach jedem Start bewusst aktiv.

**Geht auch ein anderer Wechselrichter?**
Diese Firmware ist gezielt auf die Fronius Solar-API zugeschnitten
(`GetPowerFlowRealtimeData`). Für andere Hersteller müsste die
Abfrage-Logik im `http_request`-Block entsprechend angepasst werden.

---

## Projektstruktur

```
├── ulanzi-fronius.yaml   # ESPHome-Konfiguration (Komplett-Firmware)
├── README.md             # diese Anleitung
├── LICENSE               # MIT-Lizenz
```

---

## Datenschutz

Die Firmware kommuniziert ausschließlich im **lokalen Netzwerk** mit
dem Wechselrichter und (für die Uhrzeit) mit einem NTP-Server.
Es werden keine Daten an Cloud-Dienste gesendet, keine externen
Server kontaktiert und keine Nutzungsstatistiken erhoben.

---

## Lizenz

MIT 
