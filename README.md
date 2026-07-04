# Home Assistant Blueprint: Rasenbewässerung – dynamischer Morgenstart vor Sonnenaufgang

Dieser Blueprint startet ein vorhandenes Home-Assistant-Script für die morgendliche Rasenbewässerung automatisch so, dass die Bewässerung vor Sonnenaufgang abgeschlossen werden kann.

Die Startzeit wird dynamisch berechnet anhand von:

* Anzahl der aktiven Bewässerungsdurchläufe / Kreise
* eingestellter Dauer des Langprogramms
* fixer Pause von 5 Minuten zwischen den Durchläufen
* nächstem Sonnenaufgang

Der Blueprint ist gedacht für Setups mit:

* 1–6 Gardena Impulsregnern
* Gardena Wasserverteiler / 6-fach-Verteiler
* Gardena Smartventil oder anderem smarten Bewässerungsventil
* Home Assistant
* Wetterintegration
* Sonnenaufgangs-Sensor
* separatem Script für die eigentliche Morgenlogik

---

## Was macht dieser Blueprint?

Der Blueprint entscheidet **nicht selbst**, ob bewässert wird.

Er berechnet nur den richtigen Zeitpunkt, zu dem ein vorhandenes Morgenlogik-Script gestartet werden soll.

Die eigentliche Entscheidung, ob nicht, kurz oder lange bewässert wird, muss in einem separaten Script erfolgen.

Beispiel:

```text
Startzeit = Sonnenaufgang - maximale Bewässerungsdauer
```

Die maximale Bewässerungsdauer wird berechnet aus:

```text
Anzahl Durchläufe × Dauer Langprogramm
+
(Anzahl Durchläufe - 1) × 5 Minuten Pause
```

---

## Beispielrechnung

Beispiel mit 3 Bewässerungskreisen:

```text
3 Durchläufe
Langprogramm: 50 Minuten
Pause zwischen den Durchläufen: 5 Minuten

3 × 50 min + 2 × 5 min = 160 Minuten
```

Die Morgenlogik startet also **160 Minuten vor Sonnenaufgang**.

Wenn Sonnenaufgang um 05:30 Uhr ist, startet die Morgenlogik um ca. 02:50 Uhr.

---

## Warum wird mit dem Langprogramm gerechnet?

Der Blueprint muss den frühestmöglichen sinnvollen Startzeitpunkt berechnen.

Deshalb wird immer mit dem Langprogramm gerechnet, weil dieses die längste mögliche Bewässerungsdauer darstellt.

Wenn die Morgenlogik später nur das Kurzprogramm auswählt, kann das Script intern eine Verzögerung einbauen.

Beispiel:

```text
Kurzprogramm: 30 Minuten
Langprogramm: 50 Minuten
Anzahl Durchläufe: 3

Differenz: 20 Minuten pro Durchlauf
Verzögerung bei Kurzprogramm: 3 × 20 = 60 Minuten
```

Dadurch endet ein Kurzprogramm ungefähr zur gleichen Zielzeit wie ein Langprogramm.

---

## Voraussetzungen

Dieser Blueprint ist nur der dynamische Startmechanismus.

Vor der Nutzung müssen in Home Assistant bereits passende Helfer und Scripts vorhanden sein.

### Benötigte Helfer

Empfohlen werden folgende Helfer:

```text
input_boolean.bewaesserung_automatik
input_number.bewaesserung_anzahl_durchlaeufe
input_select.bewaesserung_kurzprogramm
input_select.bewaesserung_langprogramm
input_datetime.bewaesserung_letzte_bewaesserung
input_number.bewaesserung_letzte_bewaesserungsdauer
```

Die Namen können frei gewählt werden. Beim Erstellen der Automation werden die passenden Entitäten ausgewählt.

---

## Empfohlene Helfer-Konfiguration

### Automatik-Helfer

Typ:

```text
Input Boolean
```

Beispiel:

```text
Bewässerung Automatik
```

Funktion:

```text
Schaltet die automatische Bewässerung ein oder aus.
```

---

### Anzahl der Durchläufe

Typ:

```text
Input Number
```

Empfohlene Werte:

```text
Minimum: 1
Maximum: 6
Schrittweite: 1
```

Funktion:

```text
Legt fest, wie viele Gardena-Wasserverteiler-Ausgänge bzw. Bewässerungskreise nacheinander laufen sollen.
```

---

### Kurzprogramm

Typ:

```text
Input Select
```

Empfohlene Optionen:

```text
30
40
50
60
```

Funktion:

```text
Dauer des kurzen Bewässerungsprogramms in Minuten pro Durchlauf.
```

---

### Langprogramm

Typ:

```text
Input Select
```

Empfohlene Optionen:

```text
50
60
70
80
90
100
```

Funktion:

```text
Dauer des langen Bewässerungsprogramms in Minuten pro Durchlauf.
```

---

### Letzte Bewässerung

Typ:

```text
Input Datetime
```

Empfohlene Konfiguration:

```text
Datum und Uhrzeit aktivieren
```

Funktion:

```text
Speichert, wann zuletzt automatisch bewässert wurde.
```

---

### Letzte Bewässerungsdauer

Typ:

```text
Input Number
```

Empfohlene Werte:

```text
Minimum: 0
Maximum: 600
Schrittweite: 1
Einheit: min
```

Funktion:

```text
Speichert die zuletzt ausgeführte Bewässerungsdauer in Minuten.
```

---

## Benötigtes Script: Bewässerungslauf

Dieses Script öffnet und schließt das Ventil für die eingestellte Anzahl an Durchläufen.

Es ist der eigentliche technische Bewässerungslauf.

Beispielname:

```text
script.bewaesserung_durchlauf
```

Beispielscript:

```yaml
alias: Bewässerung Durchlauf
mode: single

fields:
  laufzeit_minuten:
    name: Laufzeit pro Durchlauf
    description: Laufzeit je Bewässerungsdurchlauf in Minuten
    example: 50
    required: true
    selector:
      number:
        min: 1
        max: 120
        unit_of_measurement: min

sequence:
  - variables:
      anzahl_durchlaeufe: "{{ states('input_number.bewaesserung_anzahl_durchlaeufe') | int(1) }}"

  - repeat:
      count: "{{ anzahl_durchlaeufe | int }}"
      sequence:
        - service: valve.open_valve
          target:
            entity_id: valve.bewaesserungsventil

        - delay:
            minutes: "{{ laufzeit_minuten | int }}"

        - service: valve.close_valve
          target:
            entity_id: valve.bewaesserungsventil

        - if:
            - condition: template
              value_template: "{{ repeat.index < anzahl_durchlaeufe | int }}"
          then:
            - delay:
                minutes: 5
```

Wichtig:

```text
valve.bewaesserungsventil
```

muss durch das eigene Ventil ersetzt werden.

Beispiele:

```text
valve.garten_bewaesserung
valve.gardena_smart_ventil
switch.bewaesserungspumpe
```

Wenn statt eines Ventils ein Switch verwendet wird, müssen die Services entsprechend geändert werden:

```text
valve.open_valve  → switch.turn_on
valve.close_valve → switch.turn_off
```

---

## Benötigtes Script: Morgenlogik

Der Blueprint benötigt ein vorhandenes Morgenlogik-Script.

Beispiel:

```text
script.bewaesserung_morgenlogik
```

Dieses Script enthält die eigentliche Entscheidung:

```text
nicht bewässern
Kurzprogramm bewässern
Langprogramm bewässern
```

Es prüft beispielhaft:

```text
Regenprognose
Regenwahrscheinlichkeit
Temperatur
letzte Bewässerung
Kurzprogramm
Langprogramm
```

Beispielscript:

```yaml
alias: Bewässerung Morgenlogik
mode: single

sequence:
  - condition: state
    entity_id: input_boolean.bewaesserung_automatik
    state: "on"

  - service: weather.get_forecasts
    target:
      entity_id: weather.forecast_home
    data:
      type: hourly
    response_variable: wetter

  - variables:
      forecast: "{{ wetter['weather.forecast_home'].forecast }}"

      anzahl_durchlaeufe: "{{ states('input_number.bewaesserung_anzahl_durchlaeufe') | int(1) }}"

      kurzprogramm_minuten: "{{ states('input_select.bewaesserung_kurzprogramm') | int(30) }}"
      langprogramm_minuten: "{{ states('input_select.bewaesserung_langprogramm') | int(50) }}"

      regen_12h: >
        {% set ns = namespace(total=0) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 12 * 3600 %}
            {% set ns.total = ns.total + (f.precipitation | default(0) | float(0)) %}
          {% endif %}
        {% endfor %}
        {{ ns.total | round(1) }}

      regenwahrscheinlichkeit_12h: >
        {% set ns = namespace(max=0) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 12 * 3600 %}
            {% set p = f.precipitation_probability | default(0) | float(0) %}
            {% if p > ns.max %}
              {% set ns.max = p %}
            {% endif %}
          {% endif %}
        {% endfor %}
        {{ ns.max | round(0) }}

      max_temp_18h: >
        {% set ns = namespace(max=-99) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 18 * 3600 %}
            {% set t = f.temperature | default(-99) | float(-99) %}
            {% if t > ns.max %}
              {% set ns.max = t %}
            {% endif %}
          {% endif %}
        {% endfor %}
        {{ ns.max | round(1) }}

      stunden_seit_letzter_bewaesserung: >
        {% set last = as_timestamp(states('input_datetime.bewaesserung_letzte_bewaesserung'), 0) %}
        {{ ((as_timestamp(now()) - last) / 3600) | round(1) }}

      laufzeit_pro_durchlauf: >
        {% if regen_12h | float >= 3 %}
          0
        {% elif regenwahrscheinlichkeit_12h | float >= 70 %}
          0
        {% elif max_temp_18h | float < 24 %}
          0
        {% elif max_temp_18h | float >= 32 %}
          {{ langprogramm_minuten | int }}
        {% elif max_temp_18h | float >= 28 %}
          {{ langprogramm_minuten | int }}
        {% elif max_temp_18h | float >= 24 and stunden_seit_letzter_bewaesserung | float >= 60 %}
          {{ kurzprogramm_minuten | int }}
        {% else %}
          0
        {% endif %}

      startverzoegerung_minuten: >
        {% if laufzeit_pro_durchlauf | int == kurzprogramm_minuten | int %}
          {{ anzahl_durchlaeufe | int * ((langprogramm_minuten | int) - (kurzprogramm_minuten | int)) }}
        {% else %}
          0
        {% endif %}

  - service: persistent_notification.create
    data:
      title: "Bewässerung Morgen Debug"
      message: >
        Regen 12h: {{ regen_12h }} mm
        Regenwahrscheinlichkeit 12h: {{ regenwahrscheinlichkeit_12h }} %
        Max Temp 18h: {{ max_temp_18h }} °C
        Stunden seit letzter Bewässerung: {{ stunden_seit_letzter_bewaesserung }}
        Durchläufe: {{ anzahl_durchlaeufe }}
        Kurzprogramm: {{ kurzprogramm_minuten }} Minuten
        Langprogramm: {{ langprogramm_minuten }} Minuten
        Laufzeit pro Durchlauf: {{ laufzeit_pro_durchlauf }} Minuten
        Startverzögerung: {{ startverzoegerung_minuten }} Minuten

  - condition: template
    value_template: "{{ laufzeit_pro_durchlauf | int > 0 }}"

  - if:
      - condition: template
        value_template: "{{ startverzoegerung_minuten | int > 0 }}"
    then:
      - delay:
          minutes: "{{ startverzoegerung_minuten | int }}"

  - service: script.bewaesserung_durchlauf
    data:
      laufzeit_minuten: "{{ laufzeit_pro_durchlauf | int }}"

  - service: input_datetime.set_datetime
    target:
      entity_id: input_datetime.bewaesserung_letzte_bewaesserung
    data:
      datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"

  - service: input_number.set_value
    target:
      entity_id: input_number.bewaesserung_letzte_bewaesserungsdauer
    data:
      value: >
        {{ (laufzeit_pro_durchlauf | int) * (states('input_number.bewaesserung_anzahl_durchlaeufe') | int(1)) }}
```

Wichtig:

Diese Entitäten müssen an das eigene System angepasst werden:

```text
input_boolean.bewaesserung_automatik
weather.forecast_home
input_number.bewaesserung_anzahl_durchlaeufe
input_select.bewaesserung_kurzprogramm
input_select.bewaesserung_langprogramm
input_datetime.bewaesserung_letzte_bewaesserung
input_number.bewaesserung_letzte_bewaesserungsdauer
script.bewaesserung_durchlauf
```

---

## Blueprint-Inputs

Beim Erstellen der Automation aus diesem Blueprint müssen folgende Entitäten ausgewählt werden:

### Automatik-Helfer

Ein `input_boolean`, der die Bewässerungsautomatik aktiviert oder deaktiviert.

Beispiel:

```text
input_boolean.bewaesserung_automatik
```

---

### Sonnenaufgangs-Sensor

Ein Sensor, der den nächsten Sonnenaufgang als Datum/Uhrzeit liefert.

Beispiel:

```text
sensor.sun_next_rising
```

---

### Anzahl Durchläufe

Ein `input_number`, der die Anzahl der aktiven Bewässerungskreise enthält.

Beispiel:

```text
input_number.bewaesserung_anzahl_durchlaeufe
```

---

### Kurzprogramm

Ein `input_select` oder `input_number` mit der Kurzprogramm-Dauer in Minuten.

Beispiel:

```text
input_select.bewaesserung_kurzprogramm
```

---

### Langprogramm

Ein `input_select` oder `input_number` mit der Langprogramm-Dauer in Minuten.

Beispiel:

```text
input_select.bewaesserung_langprogramm
```

---

### Morgenlogik-Script

Das Script, das durch den Blueprint gestartet werden soll.

Beispiel:

```text
script.bewaesserung_morgenlogik
```

---

## Import in Home Assistant

### Variante 1: Import über GitHub-URL

1. In Home Assistant öffnen:

```text
Einstellungen → Automatisierungen & Szenen → Blueprints
```

2. Auf **Blueprint importieren** klicken.

3. Die URL zur Blueprint-YAML-Datei einfügen.

Beispiel:

```text
https://github.com/DEIN_USERNAME/ha-gardena-irrigation-blueprint/blob/main/blueprints/automation/rasen_morgenstart_dynamisch.yaml
```

4. Auf **Vorschau / Preview** klicken.

5. Blueprint importieren.

6. Danach aus dem Blueprint eine neue Automation erstellen.

---

### Variante 2: Manuelle Installation

Die YAML-Datei herunterladen und in Home Assistant hier ablegen:

```text
/config/blueprints/automation/community/rasen_morgenstart_dynamisch.yaml
```

Danach Home Assistant neu laden oder neu starten.

Anschließend sollte der Blueprint unter folgendem Menü erscheinen:

```text
Einstellungen → Automatisierungen & Szenen → Blueprints
```

---

## Einrichtung nach dem Import

Nach dem Import:

1. Blueprint öffnen
2. **Automation erstellen** auswählen
3. Entitäten auswählen
4. Automation speichern
5. Alten fixen Morgen-Zeitplan deaktivieren oder löschen

Wichtig:

Wenn bisher ein fixer Morgen-Scheduler verwendet wurde, muss dieser deaktiviert oder gelöscht werden, damit die Bewässerung nicht doppelt gestartet wird.

---

## Beispielauswahl für die Automation

Beispielhafte Zuordnung:

```text
Automatik-Helfer:
input_boolean.bewaesserung_automatik

Sonnenaufgangs-Sensor:
sensor.sun_next_rising

Anzahl Durchläufe:
input_number.bewaesserung_anzahl_durchlaeufe

Kurzprogramm:
input_select.bewaesserung_kurzprogramm

Langprogramm:
input_select.bewaesserung_langprogramm

Morgenlogik-Script:
script.bewaesserung_morgenlogik
```

---

## Gardena Wasserverteiler

Der Blueprint geht von einer fixen Pause von 5 Minuten zwischen den Durchläufen aus.

Diese Pause ist wichtig, damit ein Gardena Wasserverteiler nach dem Schließen des Ventils zuverlässig auf den nächsten Ausgang weiterschalten kann.

Ablauf pro Durchlauf:

```text
Ventil öffnen
Bewässern
Ventil schließen
5 Minuten Pause
nächster Ausgang am Wasserverteiler
```

Beim letzten Durchlauf wird keine weitere Pause benötigt.

---

## Optionales Script: Abendlogik

Optional kann zusätzlich ein Abendprogramm gebaut werden.

Dieses Script ist kein Teil des Blueprints, kann aber ergänzend verwendet werden.

Zweck:

```text
Bei extremer Hitze abends zusätzlich kurz bewässern.
```

Beispielscript:

```yaml
alias: Bewässerung Abendlogik
mode: single

sequence:
  - condition: state
    entity_id: input_boolean.bewaesserung_automatik
    state: "on"

  - service: weather.get_forecasts
    target:
      entity_id: weather.forecast_home
    data:
      type: hourly
    response_variable: wetter

  - variables:
      forecast: "{{ wetter['weather.forecast_home'].forecast }}"

      kurzprogramm_minuten: "{{ states('input_select.bewaesserung_kurzprogramm') | int(30) }}"

      regen_24h: >
        {% set ns = namespace(total=0) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 24 * 3600 %}
            {% set ns.total = ns.total + (f.precipitation | default(0) | float(0)) %}
          {% endif %}
        {% endfor %}
        {{ ns.total | round(1) }}

      regenwahrscheinlichkeit_24h: >
        {% set ns = namespace(max=0) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 24 * 3600 %}
            {% set p = f.precipitation_probability | default(0) | float(0) %}
            {% if p > ns.max %}
              {% set ns.max = p %}
            {% endif %}
          {% endif %}
        {% endfor %}
        {{ ns.max | round(0) }}

      max_temp_24h: >
        {% set ns = namespace(max=-99) %}
        {% for f in forecast %}
          {% if as_timestamp(f.datetime) <= as_timestamp(now()) + 24 * 3600 %}
            {% set t = f.temperature | default(-99) | float(-99) %}
            {% if t > ns.max %}
              {% set ns.max = t %}
            {% endif %}
          {% endif %}
        {% endfor %}
        {{ ns.max | round(1) }}

      aktuelle_temp: "{{ state_attr('weather.forecast_home', 'temperature') | float(0) }}"

      laufzeit_pro_durchlauf: >
        {% if regen_24h | float >= 3 %}
          0
        {% elif regenwahrscheinlichkeit_24h | float >= 70 %}
          0
        {% elif max_temp_24h | float >= 32 and aktuelle_temp | float >= 26 %}
          {{ kurzprogramm_minuten | int }}
        {% else %}
          0
        {% endif %}

  - service: persistent_notification.create
    data:
      title: "Bewässerung Abend Debug"
      message: >
        Regen 24h: {{ regen_24h }} mm
        Regenwahrscheinlichkeit 24h: {{ regenwahrscheinlichkeit_24h }} %
        Max Temp 24h: {{ max_temp_24h }} °C
        Aktuelle Temperatur: {{ aktuelle_temp }} °C
        Kurzprogramm: {{ kurzprogramm_minuten }} Minuten
        Laufzeit pro Durchlauf: {{ laufzeit_pro_durchlauf }} Minuten

  - condition: template
    value_template: "{{ laufzeit_pro_durchlauf | int > 0 }}"

  - service: script.bewaesserung_durchlauf
    data:
      laufzeit_minuten: "{{ laufzeit_pro_durchlauf | int }}"

  - service: input_datetime.set_datetime
    target:
      entity_id: input_datetime.bewaesserung_letzte_bewaesserung
    data:
      datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"

  - service: input_number.set_value
    target:
      entity_id: input_number.bewaesserung_letzte_bewaesserungsdauer
    data:
      value: >
        {{ (laufzeit_pro_durchlauf | int) * (states('input_number.bewaesserung_anzahl_durchlaeufe') | int(1)) }}
```

Dieses Script muss über einen eigenen Zeitplan oder eine eigene Automation gestartet werden.

Beispiel:

```yaml
alias: Bewässerung Abendprüfung
mode: single

trigger:
  - platform: time
    at: "20:00:00"

condition:
  - condition: state
    entity_id: input_boolean.bewaesserung_automatik
    state: "on"

action:
  - service: script.bewaesserung_abendlogik
```

---

## Optionales Script: Bewässerung aktivieren

Optional kann ein Script zum Aktivieren der Bewässerung erstellt werden.

Beispiel:

```yaml
alias: Bewässerung aktivieren
mode: single

sequence:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.bewaesserung_automatik

  - service: persistent_notification.create
    data:
      title: "Bewässerung aktiviert"
      message: >
        Die Bewässerungsautomatik wurde aktiviert.
```

Wenn zusätzlich eine bestimmte Automation eingeschaltet werden soll, kann diese ergänzt werden:

```yaml
alias: Bewässerung aktivieren
mode: single

sequence:
  - service: input_boolean.turn_on
    target:
      entity_id: input_boolean.bewaesserung_automatik

  - service: automation.turn_on
    target:
      entity_id: automation.bewaesserung_morgenstart_dynamisch

  - service: persistent_notification.create
    data:
      title: "Bewässerung aktiviert"
      message: >
        Die Bewässerungsautomatik und die dynamische Morgenautomation wurden aktiviert.
```

---

## Optionales Script: Bewässerung deaktivieren

Optional kann ein Script zum Deaktivieren der Bewässerung erstellt werden.

Beispiel:

```yaml
alias: Bewässerung deaktivieren
mode: single

sequence:
  - service: input_boolean.turn_off
    target:
      entity_id: input_boolean.bewaesserung_automatik

  - service: valve.close_valve
    target:
      entity_id: valve.bewaesserungsventil

  - service: persistent_notification.create
    data:
      title: "Bewässerung deaktiviert"
      message: >
        Die Bewässerungsautomatik wurde deaktiviert und das Ventil wurde geschlossen.
```

Wenn statt eines Ventils ein Switch verwendet wird:

```yaml
alias: Bewässerung deaktivieren
mode: single

sequence:
  - service: input_boolean.turn_off
    target:
      entity_id: input_boolean.bewaesserung_automatik

  - service: switch.turn_off
    target:
      entity_id: switch.bewaesserungspumpe

  - service: persistent_notification.create
    data:
      title: "Bewässerung deaktiviert"
      message: >
        Die Bewässerungsautomatik wurde deaktiviert und der Bewässerungsschalter wurde ausgeschaltet.
```

---

## Sicherheitsabschaltung empfohlen

Zusätzlich zu diesem Blueprint sollte eine Sicherheitsabschaltung eingerichtet werden.

Empfehlung:

```text
Wenn das Bewässerungsventil länger als die maximale Einzellaufzeit + Reserve offen ist,
wird es automatisch geschlossen.
```

Beispiel:

```text
Maximale Langprogramm-Dauer: 100 Minuten
Sicherheitsabschaltung: 110 Minuten
```

Beispielautomation für ein Ventil:

```yaml
alias: Bewässerung Sicherheitsabschaltung
description: Schließt das Bewässerungsventil, wenn es länger als 110 Minuten offen ist.
mode: restart

trigger:
  - platform: state
    entity_id: valve.bewaesserungsventil
    to:
      - "open"
      - "opening"
    for:
      hours: 0
      minutes: 110
      seconds: 0

action:
  - service: valve.close_valve
    target:
      entity_id: valve.bewaesserungsventil

  - service: persistent_notification.create
    data:
      title: "Bewässerung Sicherheitsabschaltung"
      message: >
        Das Bewässerungsventil war länger als 110 Minuten offen und wurde automatisch geschlossen.
```

Beispielautomation für einen Switch:

```yaml
alias: Bewässerung Sicherheitsabschaltung
description: Schaltet die Bewässerung aus, wenn sie länger als 110 Minuten aktiv ist.
mode: restart

trigger:
  - platform: state
    entity_id: switch.bewaesserungspumpe
    to: "on"
    for:
      hours: 0
      minutes: 110
      seconds: 0

action:
  - service: switch.turn_off
    target:
      entity_id: switch.bewaesserungspumpe

  - service: persistent_notification.create
    data:
      title: "Bewässerung Sicherheitsabschaltung"
      message: >
        Die Bewässerung war länger als 110 Minuten aktiv und wurde automatisch ausgeschaltet.
```

---

## Wichtiger Hinweis zur Bewässerungsdauer

Dieser Blueprint legt keine Bewässerungsdauer fest.

Die Bewässerungsdauer wird über die Helfer für Kurzprogramm und Langprogramm eingestellt und muss vom Morgenlogik-Script ausgewertet werden.

Empfohlene Startwerte:

```text
Kurzprogramm: 30 Minuten
Langprogramm: 50 Minuten
```

Wenn der Rasen trotz Bewässerung trocken oder braun wird, können die Zeiten schrittweise erhöht werden.

Beispiel:

```text
Kurzprogramm: 40 / 50 / 60 Minuten
Langprogramm: 60 / 70 / 80 / 90 / 100 Minuten
```

---

## Empfohlener Bechertest

Um die tatsächliche Wassermenge zu prüfen, können mehrere gerade Behälter oder Becher auf der Rasenfläche verteilt werden.

Ablauf:

```text
1. 6–10 Behälter auf dem Rasen verteilen
2. Bewässerung 30 Minuten laufen lassen
3. Wasserhöhe in jedem Behälter messen
4. Durchschnitt bilden
```

Faustregel:

```text
1 mm Wasserhöhe im Behälter = 1 mm Niederschlag = 1 Liter pro m²
```

Damit lässt sich abschätzen, ob die gewählten Laufzeiten ausreichen.

---

## Was dieser Blueprint nicht macht

Dieser Blueprint:

```text
steuert nicht direkt das Ventil
prüft nicht selbst das Wetter
prüft nicht selbst die Regenprognose
entscheidet nicht selbst Kurz- oder Langprogramm
erstellt keine Helfer
erstellt kein Dashboard
ersetzt kein Bewässerungsscript
```

Er macht ausschließlich:

```text
dynamischen Start der Morgenlogik vor Sonnenaufgang
```

---

## Typischer Gesamtaufbau

Ein vollständiges Setup besteht aus mehreren Bausteinen:

```text
1. Helfer für Automatik, Durchläufe, Kurzprogramm und Langprogramm
2. Script für den eigentlichen Bewässerungslauf
3. Script für die Morgenlogik
4. Dieser Blueprint für den dynamischen Morgenstart
5. Optional: Abendlogik
6. Optional: Sicherheitsabschaltung
7. Optional: Dashboard
```

---

## Fehlerbehebung

### Blueprint erscheint nicht

Prüfen:

```text
Liegt die Datei im richtigen Ordner?
Ist die YAML-Datei gültig?
Wurde Home Assistant neu geladen oder neu gestartet?
```

---

### Blueprint-Import zeigt „Unknown error“

Mögliche Ursachen:

```text
falsche GitHub-URL
Repository nicht öffentlich
YAML-Datei enthält Formatierungsfehler
URL zeigt nicht direkt auf die YAML-Datei
```

Die URL sollte auf die Blueprint-Datei zeigen, zum Beispiel:

```text
https://github.com/DEIN_USERNAME/ha-gardena-irrigation-blueprint/blob/main/blueprints/automation/rasen_morgenstart_dynamisch.yaml
```

Nicht nur auf das Repository:

```text
https://github.com/DEIN_USERNAME/ha-gardena-irrigation-blueprint
```

---

### Automation startet nicht

Prüfen:

```text
Ist der Automatik-Helfer eingeschaltet?
Ist der Sonnenaufgangs-Sensor verfügbar?
Ist die Anzahl der Durchläufe korrekt gesetzt?
Ist das Langprogramm korrekt gesetzt?
Ist das ausgewählte Morgenlogik-Script vorhanden?
Ist die Automation aktiviert?
```

---

### Bewässerung startet zu früh

Prüfen:

```text
Ist die Anzahl der Durchläufe zu hoch eingestellt?
Ist das Langprogramm zu lang eingestellt?
Wird wirklich der richtige Sonnenaufgangs-Sensor verwendet?
```

---

### Bewässerung startet doppelt

Prüfen:

```text
Alter fixer Morgen-Zeitplan noch aktiv?
Alte Morgen-Automation noch aktiv?
Mehrere Automationen aus dem Blueprint erstellt?
```

Wenn dieser Blueprint genutzt wird, sollte ein alter fixer Morgen-Scheduler deaktiviert oder gelöscht werden.

---

## Lizenz

Dieses Projekt kann frei privat genutzt, angepasst und geteilt werden.

Wenn du den Blueprint weiterentwickelst oder veröffentlichst, wäre ein Hinweis auf die ursprüngliche Idee bzw. Quelle fair, ist aber nicht zwingend erforderlich.

---

## Kurzfassung

Dieser Blueprint startet ein vorhandenes Morgenlogik-Script dynamisch vor Sonnenaufgang.

Die Startzeit richtet sich nach:

```text
Anzahl Durchläufe
Langprogramm-Dauer
5 Minuten Pause zwischen den Durchläufen
Sonnenaufgang
```

Er eignet sich besonders für Rasenbewässerung mit Gardena Wasserverteiler, 1–6 Impulsregnern und Home Assistant.
