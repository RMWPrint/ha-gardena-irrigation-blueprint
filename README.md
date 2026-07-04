# ha-gardena-irrigation-blueprint
Home Assistant blueprint for dynamic lawn irrigation start before sunrise

# Home Assistant Gardena Irrigation Blueprint

Blueprint für eine dynamische Rasenbewässerung mit Home Assistant.

Der Blueprint startet ein vorhandenes Morgenlogik-Script automatisch so früh vor Sonnenaufgang, dass die eingestellte Anzahl an Bewässerungsdurchläufen vor Sonnenaufgang fertig werden kann.

## Gedacht für

- 1-6 Gardena Impulsregner
- Gardena Wasserverteiler / 6-fach-Verteiler
- Gardena Smartventil oder anderes smartes Bewässerungsventil
- Home Assistant
- Wetterintegration
- Sonnenaufgangs-Sensor

## Was macht der Blueprint?

Der Blueprint startet ein vorhandenes Script zu diesem Zeitpunkt:

```text
Startzeit = Sonnenaufgang - maximale Bewässerungsdauer# Home Assistant Gardena Irrigation Blueprint

Blueprint für eine dynamische Rasenbewässerung mit Home Assistant.

Der Blueprint startet ein vorhandenes Morgenlogik-Script automatisch so früh vor Sonnenaufgang, dass die eingestellte Anzahl an Bewässerungsdurchläufen vor Sonnenaufgang fertig werden kann.

## Gedacht für

- 1-6 Gardena Impulsregner
- Gardena Wasserverteiler / 6-fach-Verteiler
- Gardena Smartventil oder anderes smartes Bewässerungsventil
- Home Assistant
- Wetterintegration
- Sonnenaufgangs-Sensor

## Was macht der Blueprint?

Der Blueprint startet ein vorhandenes Script zu diesem Zeitpunkt:

```text
Startzeit = Sonnenaufgang - maximale Bewässerungsdauer

Anzahl Durchläufe × Langprogramm-Dauer
+
(Anzahl Durchläufe - 1) × Pause

3 Durchläufe
Langprogramm: 50 Minuten
Pause: 5 Minuten

3 × 50 + 2 × 5 = 160 Minuten

Die Morgenlogik startet 160 Minuten vor Sonnenaufgang.

!!Wichtig!!

Dieser Blueprint entscheidet nicht selbst, ob bewässert wird.

Er startet nur das ausgewählte Morgenlogik-Script zur richtigen Zeit.

Die eigentliche Logik, also Wetterprüfung, Regenprüfung, Kurz-/Langprogramm und Ventilsteuerung, muss in einem separaten Script liegen.

Benötigte Helfer:

input_boolean.bewaesserung_automatik
input_number.bewaesserung_anzahl_durchlaeufe
input_select.bewaesserung_langprogramm
sensor.sun_next_rising
script.bewaesserung_morgenlogik

Import in Home Assistant

Home Assistant kann Blueprints direkt aus GitHub importieren.

1. In Home Assistant öffnen:
2. Einstellungen → Automatisierungen & Szenen → Blueprints
3. Import Blueprint wählen
4. GitHub-URL zur Blueprint-Datei einfügen
4. Preview wählen
5. Blueprint importieren
6. Automation aus Blueprint erstellen
7. Entitäten auswählen
8. Speichern

Blueprint-Datei
blueprints/automation/rasen_morgenstart_dynamisch.yaml
Empfohlene Startwerte
Pause zwischen Durchläufen: 5 Minuten
Anzahl Durchläufe: 1-6
Langprogramm: 50 Minuten
Sicherheit

Zusätzlich wird eine Sicherheitsabschaltung empfohlen:

Wenn das Ventil länger als die maximale Einzellaufzeit + Reserve offen bleibt,
wird es automatisch geschlossen.

Beispiel:

maximale Langprogramm-Dauer: 100 Minuten
Sicherheitsabschaltung: 110 Minuten
Hinweis

Bei Nutzung eines Gardena Wasserverteilers sollte zwischen den Durchläufen eine Pause eingeplant werden, damit der Verteiler zuverlässig zum nächsten Ausgang weiterschaltet.


Commit message:

```text
Add README
