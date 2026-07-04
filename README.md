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
