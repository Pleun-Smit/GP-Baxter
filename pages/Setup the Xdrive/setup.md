---
layout: default
title: Setup XDRIVE MINI
permalink: /setup/
---

# MKS XDrive Mini configureren met hall-sensor (odrivetool 0.5.1)

Stappenplan om een 3-fasen BLDC-motor met hall-sensoren op een MKS XDrive Mini aan de praat te krijgen en in closed loop (snelheidsregeling) te laten draaien.

**Uitgangssituatie:** motor met 6 poolparen en hall-sensoren, odrivetool 0.5.1. Hall-sensoren geven 6 tellingen per elektrische omwenteling, dus `cpr = 6 × poolparen = 36`.

> De waarden hieronder zijn getest op één opstelling. Stroom- en spanningslimieten en regelwinsten zijn motorafhankelijk. Begin altijd laag en controleer zelf.

---

## Overzicht

1. [Motor en encoder instellen](#1-motor-en-encoder-instellen)
2. [Kalibreren](#2-kalibreren)
3. [Opslaan](#3-opslaan)
4. [Eerste closed-loop test](#4-eerste-closed-loop-test)
5. [Regelaar afstellen](#5-regelaar-afstellen)
6. [Echte snelheid meten](#6-echte-snelheid-meten)
7. [Vastleggen](#7-vastleggen)
8. [Problemen oplossen](#8-problemen-oplossen)
9. [Beperkingen van hall-sensoren](#9-beperkingen-van-hall-sensoren)

---

## 1. Motor en encoder instellen

Start `odrivetool` en wacht tot `odrv0` verbonden is. Stel de motor in volgens je eigen motorspecificaties (stroomlimiet, kalibratiestroom, motortype) en daarna de encoder.

```python
odrv0.axis0.requested_state = AXIS_STATE_IDLE

# Motor
odrv0.axis0.motor.config.pole_pairs = 6
# current_lim, calibration_current en motor_type: volgens je motor

# Encoder: hall-sensor
odrv0.axis0.encoder.config.mode = ENCODER_MODE_HALL
odrv0.axis0.encoder.config.cpr = 36
odrv0.axis0.encoder.config.use_index = False
odrv0.axis0.encoder.config.bandwidth = 100

odrv0.save_configuration()
```

**Let op:** laat de encodermodus niet op `ENCODER_MODE_INCREMENTAL` staan. De firmware leest de hall-signalen dan als A/B-kwadratuur en het gevolg is `CONTROL_DEADLINE_MISSED` zodra je closed loop probeert, terwijl `LOCKIN_SPIN` (zonder encoder) wel soepel draait.

## 2. Kalibreren

Draai de motor vrij, zonder belasting. Wacht na elke stap tot de as weer op IDLE staat, anders onderbreekt de volgende opdracht de vorige.

```python
import time

odrv0.axis0.requested_state = AXIS_STATE_MOTOR_CALIBRATION
time.sleep(10)
print(odrv0.axis0.current_state)    # 1 = IDLE
dump_errors(odrv0)                  # moet schoon zijn

odrv0.axis0.requested_state = AXIS_STATE_ENCODER_OFFSET_CALIBRATION
time.sleep(10)
print(odrv0.axis0.current_state)
dump_errors(odrv0)
```

Controleer:

```python
odrv0.axis0.motor.is_calibrated      # True
odrv0.axis0.encoder.is_ready         # True
odrv0.axis0.encoder.config.offset    # niet meer 0
odrv0.axis0.encoder.error            # 0
```

**Firmware 0.5.1:** er bestaat geen `AXIS_STATE_ENCODER_HALL_POLARITY_CALIBRATION` en geen `encoder.config.hall_polarity`. Dat zit pas in latere versies. Hier volstaat `ENCODER_OFFSET_CALIBRATION`.

## 3. Opslaan

```python
odrv0.axis0.motor.config.pre_calibrated = True
odrv0.axis0.encoder.config.pre_calibrated = True
odrv0.save_configuration()
```

Het bord herstart. Verbind odrivetool opnieuw en controleer dat `encoder.config.mode` op hall staat en `cpr` op 36.

## 4. Eerste closed-loop test

Begin met lage winst, zonder integrator en met een helling op de opdracht.

```python
c = odrv0.axis0.controller.config
c.control_mode = CONTROL_MODE_VELOCITY_CONTROL
c.input_mode = INPUT_MODE_VEL_RAMP
c.vel_ramp_rate = 1          # toeren/s per seconde
c.vel_limit = 7
c.vel_gain = 0.01
c.vel_integrator_gain = 0

odrv0.axis0.controller.input_vel = 0
odrv0.axis0.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
odrv0.axis0.controller.input_vel = 3
```

Stoppen:

```python
odrv0.axis0.controller.input_vel = 0
odrv0.axis0.requested_state = AXIS_STATE_IDLE
```

Fouten wissen: `dump_errors(odrv0, True)`.

## 5. Regelaar afstellen

Verander steeds één ding tegelijk en herhaal elke test minstens twee keer.

| Parameter | Waarde | Ervaring |
|---|---|---|
| `vel_gain` | **0,01** | Stabiel |
| `vel_gain` | 0,015 | Draait, maar meer rimpel |
| `vel_gain` | 0,02 | Lus wordt instabiel (overspeed binnen een halve seconde) |
| `vel_integrator_gain` | **0** | Niet nodig, zie stap 6 |
| `vel_integrator_gain` | 0,15 | Veel te hoog, gaf `CONTROLLER_ERROR_OVERSPEED` |
| `encoder.config.bandwidth` | **100** | Beste resultaat |
| `encoder.config.bandwidth` | 200 en hoger | Niet beter, soms onrustiger |
| `encoder.config.bandwidth` | 50 | Te traag, lus faalt bij hogere winst |

Een sprong in `input_vel` van 0 naar de doelsnelheid is het zwaarste wat je de regelaar kunt geven. Gebruik `INPUT_MODE_VEL_RAMP`.

## 6. Echte snelheid meten

`encoder.vel_estimate` is bij een hall-sensor een slechte maatstaf. Met 36 flanken per omwenteling levert elke meting nul, één of twee flanken op, en ongelijke flankafstanden maken dat erger. De schatting rimpelt dan 4 tot 6 toeren/s terwijl de motor gewoon gelijkmatig draait. Meet daarom de telling over een langere tijd:

```python
def echte_snelheid(vel, wacht=5, n=5):
    ax = odrv0.axis0
    ax.controller.input_vel = 0
    ax.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
    ax.controller.input_vel = vel
    time.sleep(wacht + abs(vel))          # helling + opstarten
    res = []
    for _ in range(n):
        c0 = ax.encoder.shadow_count; t0 = time.time()
        time.sleep(1)
        res.append((ax.encoder.shadow_count - c0) / 36 / (time.time() - t0))
    ax.controller.input_vel = 0
    ax.requested_state = AXIS_STATE_IDLE
    return sum(res) / len(res), res

for v in [1, 2, 3, 4, 5, 6]:
    gem, _ = echte_snelheid(v)
    print(v, round(gem, 3))
```

**Resultaat bij opdracht 3 toeren/s:** tien metingen van een seconde gaven gemiddeld 2,995, binnen 0,3% van de opdracht. Er is dus geen restfout en je hebt geen integrator nodig.

Een gemiddelde over een seconde bewijst niet dat de motor binnen één omwenteling gelijkmatig loopt. Beoordeel schokken daarom ook op gehoor en gevoel, en bij voorkeur met belasting.

## 7. Vastleggen

```python
c = odrv0.axis0.controller.config
c.vel_gain = 0.01
c.vel_integrator_gain = 0
c.vel_limit = 7
c.input_mode = INPUT_MODE_VEL_RAMP
c.vel_ramp_rate = 1
odrv0.axis0.encoder.config.bandwidth = 100
odrv0.save_configuration()
```

Na de herstart verbind je opnieuw en test je kort met `input_vel = 3`. Daarmee bevestig je ook dat de kalibratie de herstart overleeft.

## 8. Problemen oplossen

| Melding | Betekenis en aanpak |
|---|---|
| `MOTOR_ERROR_CONTROL_DEADLINE_MISSED` | De regelinterrupt was niet op tijd klaar. Controleer de encodermodus (hall, niet incremental). Sluit de liveplotter en vermijd zware `dump`-aanroepen tijdens het testen. Kom je hem tegen terwijl de snelheidsschatting netjes verloopt, controleer dan de hall-bedrading (kort, geschermd, pull-ups). |
| `CONTROLLER_ERROR_OVERSPEED` | Gemeten snelheid boven `vel_limit` × tolerantie (standaard 1,2). Meestal te hoge `vel_gain` of integrator, of een sprong in `input_vel`. Verlaag de winst en gebruik een helling. Staat het koppel de verkeerde kant op, dan klopt de offset of de fasevolgorde niet: herhaal de offsetkalibratie of draai twee motorfasen om. |
| `AXIS_ERROR_CONTROLLER_FAILED` | Gevolg van een controllerfout, kijk naar `odrv0.axis0.controller.error`. |
| `NameError` op `AXIS_STATE_...` of `AttributeError` op `hall_polarity` | Bestaat niet in 0.5.1, zie stap 2. |
| Motor begint pas te draaien vanaf ~2 toeren/s | Eigenschap van de hall-sensor met 36 tellingen per omwenteling. Meer winst helpt niet, zie stap 5. |

Fouten tonen: `dump_errors(odrv0)`. Fouten wissen: `dump_errors(odrv0, True)`.

## 9. Beperkingen van hall-sensoren

- Resolutie: 36 tellingen per omwenteling is 10° per telling. Voor snelheidsregeling in het middengebied (hier ongeveer 3 tot 5 toeren/s) is dat prima.
- Bij lage snelheid komen te weinig flanken binnen voor een goede schatting, en positieregeling is grof (paar graden).
- Voor soepel gedrag bij lage snelheid of fijne positieregeling is een incrementele encoder of absolute magneetencoder (bijvoorbeeld een AS5047P) nodig.

---

*Getest met: MKS XDrive Mini, odrivetool 0.5.1, motor met 6 poolparen en hall-sensoren (cpr 36).*