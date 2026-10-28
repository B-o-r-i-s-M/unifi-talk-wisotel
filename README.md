# Wisotel SIP-Trunk mit UniFi Talk (UCG Fiber)

Erfahrungsbericht und Schritt-für-Schritt-Anleitung, wie ein **Wisotel-SIP-Anschluss mit mehreren Rufnummern** in **UniFi Talk** auf einem **UniFi Cloud Gateway Fiber (UCG Fiber)** eingerichtet wird – inklusive eines Drittanbieter-WLAN-Telefons (Gequdio GW-30) und der Lösung für Sprachaussetzer im WLAN.

> Getestet im Oktober 2026 mit UniFi OS 5.1.x, UniFi Talk 5.3.2, UniFi Network 10.6.x.
> Keine offizielle Anleitung von Ubiquiti oder Wisotel – Nutzung auf eigene Verantwortung.

---

## Inhalt

1. [Ausgangslage](#1-ausgangslage)
2. [Technische Eckdaten von Wisotel](#2-technische-eckdaten-von-wisotel)
3. [Vorbereitung am Gateway](#3-vorbereitung-am-gateway)
4. [SIP-Provider in UniFi Talk anlegen](#4-sip-provider-in-unifi-talk-anlegen)
5. [Rufnummern und Gruppen](#5-rufnummern-und-gruppen)
6. [Drittanbieter-Telefon anbinden (Beispiel Gequdio GW-30)](#6-drittanbieter-telefon-anbinden-beispiel-gequdio-gw-30)
7. [Testen](#7-testen)
8. [Sprachaussetzer bei WLAN-Telefonen](#8-sprachaussetzer-bei-wlan-telefonen)
9. [Stolpersteine / Fehlersuche](#9-stolpersteine--fehlersuche)

---

## 1. Ausgangslage

- Wisotel-Anschluss mit **mehreren Rufnummern**, jede Rufnummer hat **eigene SIP-Zugangsdaten** (Benutzername + Passwort), Domain `sip.wisotel.de`.
- Wisotel stellt den SIP-Zugang standardmäßig in einem eigenen VoIP-VLAN bereit. In diesem Fall wurde er **auf Absprache im selben Netz wie der Internetzugang** geschaltet – das UCG braucht dann keine zweite WAN-Schnittstelle bzw. kein separates VLAN für VoIP. Ob das bei dir möglich ist, klärst du mit Wisotel.
- Ziel: Alle Rufnummern kommen in UniFi Talk an, jede Nummer ist einzeln ansprechbar, abgehend wird mit einer festgelegten Nummer telefoniert.

## 2. Technische Eckdaten von Wisotel

Zusammengefasst aus der technischen Dokumentation von Wisotel (bitte immer mit der aktuellen Doku deines Vertrags abgleichen):

| Parameter | Wert |
|---|---|
| Registrar / Proxy | `sip.wisotel.de` |
| Modus | Registered Mode (Registrierung pro Rufnummer) |
| SIP-Port | 5060 UDP/TCP, 5061 TLS |
| Registrierung als | `<SIP-Benutzer>@sip.wisotel.de` |
| RTP-Ports (Wisotel-Seite) | 30000–44999 |
| Codecs | G.711a, G.711µ, G.722 |
| DTMF | RFC 2833 |
| Ptime | 20 ms |
| STUN | nicht nötig / nicht empfohlen |
| SIP-ALG | **deaktivieren** |
| Signalisierungs-IPs von Wisotel | `195.42.245.0/25` |

## 3. Vorbereitung am Gateway

- **SIP-ALG** in UniFi Network deaktivieren, falls aktiv (Einstellungen → Internet bzw. erweiterte Routing-/Firewall-Optionen, je nach Version).
- Keine Portweiterleitungen nötig – UniFi Talk registriert sich ausgehend und hält die Verbindung offen.

## 4. SIP-Provider in UniFi Talk anlegen

UniFi Talk → **Settings → SIP Providers → Add New SIP Provider**

| Feld | Einstellung |
|---|---|
| Provider | `Other (set up manually)` |
| Provider Name | frei, z. B. `wisotel` |
| SIP Proxy | `sip.wisotel.de` |
| Username / Password (global) | **leer lassen** |
| ✅ Each phone number registers with its own credentials | **aktivieren** |
| ✅ Route Incoming Calls By Username | **aktivieren** (siehe Hinweis unten) |
| Register with Provider | `Yes` |
| Transport | `UDP` |
| Realm | `sip.wisotel.de` |
| ✅ Restrict signaling to provider IPs | aktivieren, IP-Bereich `195.42.245.0/25` |
| Destination Countries | `Germany (+49)` |
| Use as default for outbound calls… | nach Bedarf |

Unter **Add parameter** zusätzlich:

| Parameter | Wert |
|---|---|
| From Domain | `sip.wisotel.de` |
| Outbound Proxy | `sip.wisotel.de` |

> ⚠️ **Kein globales „Auth Username“ setzen.** In unserem Fall war dort zunächst ein Benutzer einer einzelnen Rufnummer eingetragen. Folge: Nur diese Nummer registrierte sich korrekt, die anderen nicht. Mit „Each phone number registers with its own credentials“ bringt jede Nummer ihre eigenen Zugangsdaten mit – das globale Feld muss leer bleiben.

### Warum „Route Incoming Calls By Username“?

Bei Einzel-SIP-Benutzern pro Rufnummer schickt Wisotel im `To`-Header eingehender Anrufe den **SIP-Benutzernamen** (eine Ziffernfolge) statt der Rufnummer. UniFi Talk findet dann zunächst keine passende Nummer, und der Anruf läuft ins Leere. Mit **Route Incoming Calls By Username** ordnet Talk den Anruf anhand des Benutzernamens der richtigen Rufnummer zu. Einen Workaround-Eintrag mit dem Benutzernamen als „Rufnummer“ braucht man damit nicht mehr.

## 5. Rufnummern und Gruppen

### Rufnummern im Provider

Im Abschnitt **Phone Numbers** des Providers jede Nummer einzeln hinzufügen und die zugehörigen Zugangsdaten eintragen:

| Phone Number | Username | Password |
|---|---|---|
| `0711xxxxxxx` (nationale Schreibweise mit Vorwahl) | SIP-Benutzer aus dem Wisotel-Kundenportal | zugehöriges Passwort |

Nach dem Speichern sollte jede Nummer als registriert angezeigt werden.

### Gruppen (Call Routing → Groups)

Damit jede Nummer gezielt auf bestimmten Telefonen klingelt, pro Rufnummer eine Gruppe anlegen:

| Gruppe | Nummer | Mitglieder | Call Handling |
|---|---|---|---|
| z. B. „Haus 1“ | Rufnummer 1 | Telefon(e) | Simultaneous |
| „Haus 2“ | Rufnummer 2 | Telefon(e) | Simultaneous |
| „Haus 3“ | Rufnummer 3 | Telefon(e) | Simultaneous |

Die **abgehende Rufnummer** eines Telefons bzw. Benutzers legst du in den Benutzereinstellungen fest (Outbound Number).

## 6. Drittanbieter-Telefon anbinden (Beispiel Gequdio GW-30)

In UniFi Talk: **Phones → Add Third-Party Device**. Talk erzeugt eine Durchwahl (z. B. `0003`) samt SIP-Passwort.

Am Telefon (Weboberfläche) eine SIP-Leitung anlegen:

| Feld | Wert |
|---|---|
| Benutzer / Auth-Benutzer | Durchwahl aus Talk, z. B. `0003` |
| Passwort | SIP-Passwort aus Talk |
| SIP-Server / Proxy | IP des UCG, Port `5060`, UDP |
| Registrierung abgelaufen (Expiry) | **300 s** (statt 3600 s – schnellere Wiederanmeldung nach Störungen) |
| STUN | aus |
| „Benutze VPN“ | **aus** (sofern kein VPN-Tunnel genutzt wird) |
| Keepalive | UDP, 30 s |
| Codecs (Reihenfolge) | **G.722 → G.711A → G.711U**; G.729, Opus, iLBC u. Ä. deaktivieren |
| DTMF | RFC 2833 / Auto |
| Wählplan „Senden nach“ | 4 s (statt 10 s) |

Außerdem: **Admin-Passwort des Telefons ändern** – die Werkseinstellung ist bei vielen Geräten aktiv.

## 7. Testen

- **Abgehend:** Vom Telefon eine externe Nummer anrufen (z. B. Handy). In Talk erscheint der Anruf im Call Log, auf dem Handy sollte die eingestellte Absendernummer angezeigt werden.
- **Eingehend:** Von **extern** (Handy) jede Rufnummer einzeln anrufen. Im Talk-Call-Log muss jeder Anruf bei der richtigen Gruppe landen.
- ⚠️ Ein Anruf vom eigenen Telefon auf eine **eigene** Rufnummer testet den Trunk **nicht** – Talk erkennt die eigene Nummer und vermittelt intern, ohne über Wisotel zu gehen.
- Viele SIP-Telefone haben in der Weboberfläche eine Funktion „Web Wahl“ / Click-to-Dial – praktisch für Tests ohne das Telefon in der Hand.

## 8. Sprachaussetzer bei WLAN-Telefonen

**Symptom:** Während längerer Gespräche fällt die Sprache in **beide Richtungen** für 10–15 Sekunden aus, das Gespräch bleibt aber bestehen. Die Aussetzer werden mit der Zeit häufiger.

**Ursache (in unserem Fall):** Das Telefon stand zwischen zwei Access Points mit fast gleich starkem Signal und wechselte ständig hin und her („Ping-Pong-Roaming“). Bei jedem Wechsel schlug die schnelle Anmeldung fehl – im Verbindungsprotokoll des Clients in UniFi Network als *„Failed to connect … due to an authentication failure“* direkt neben *„Roamed from AP A to AP B“* sichtbar. Auslöser war das Zusammenspiel von **Fast Roaming (802.11r)**, **WPA2/WPA3-Mischbetrieb** und **PMF**, mit dem das einfache WLAN-Telefon nicht sauber umgehen konnte. Statt in Millisekunden zu wechseln, meldete es sich jedes Mal komplett neu an.

**So findest du es:** UniFi Network → Client-Geräte → Telefon → *Observability* / Verbindungsprotokoll. Zeitstempel der Roaming- und Fehlermeldungen mit den Zeiten der Aussetzer vergleichen.

**Lösung:** Eine eigene SSID für Telefon (und IoT-Geräte):

| Einstellung | Wert |
|---|---|
| Frequenzband | nur 2,4 GHz |
| Sicherheit | nur WPA2 (kein WPA3-Mischbetrieb) |
| PMF | aus |
| Fast Roaming (802.11r) | aus |
| BSS Transition | aus |

Zusätzlich am Telefon das Roaming beruhigen (sofern einstellbar): Wechsel erst unter ca. −70 dBm und bei mindestens 10 dB besserem Signal, Suche alle 30 s.

Schnelle Alternative: Das Telefon in UniFi Network fest an einen Access Point binden („Lock to AP“) – dann gibt es keine Wechsel mehr, aber auch kein Roaming, wenn man mit dem Telefon durchs Haus läuft.

Hängt das Telefon danach in einem eigenen VLAN, muss die Firewall SIP (UDP 5060) und RTP vom Telefon zum UCG erlauben.

## 9. Stolpersteine / Fehlersuche

| Problem | Ursache / Lösung |
|---|---|
| Nur eine Rufnummer registriert sich | Globales „Auth Username“ im Provider gesetzt → leeren, „Each phone number registers with its own credentials“ aktivieren |
| Eingehende Anrufe kommen nicht an / landen nirgends | „Route Incoming Calls By Username“ aktivieren |
| Testanruf auf eigene Nummer kommt „nicht an“ | Talk vermittelt intern; mit externem Telefon testen |
| Einseitige Sprache / kein Ton | SIP-ALG deaktivieren, keine STUN-Einstellungen am Telefon |
| Telefon registriert, aber Aussetzer im Gespräch | WLAN-Roaming prüfen (Abschnitt 8) |
| Telefon nach Störung lange nicht erreichbar | Registrierungsintervall am Telefon auf 300 s senken |

---

## Lizenz

Text unter [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.de) – gerne teilen, verbessern und per Issue/Pull Request ergänzen.

Wisotel, UniFi und Gequdio sind Marken ihrer jeweiligen Inhaber. Die technischen Daten stammen aus der Dokumentation von Wisotel; maßgeblich ist immer die aktuelle Dokumentation deines Anbieters.
