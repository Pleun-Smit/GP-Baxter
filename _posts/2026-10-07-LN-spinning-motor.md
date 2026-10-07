---
title: "Eerste draaiende motor - MKS XDRIVE MINI"
published: true
category: lab-notes
permalink: /spinning-motor-ln.html
summary: "Experimenten met het configureren en aansturen van een Baxter-motor met een MKS XDRIVE MINI en hall-sensoren."
tags: [lab-notes]
---

# Eerste draaiende motor - MKS XDRIVE MINI

**Datum:** 05/10/2026
**Hardware:** MKS XDRIVE MINI
**Motor:** Baxter-motor
**Software:** `odrivetool 0.5.1`

## Doel

Het doel van dit experiment was om één van de motoren van Baxter voor het eerst succesvol te laten draaien met een **MKS XDRIVE MINI**.

Er was vrijwel geen documentatie beschikbaar over de gebruikte motor. Daardoor moesten verschillende parameters, zoals het aantal pole pairs, de CPR en de juiste configuratie van de hall-sensoren, experimenteel worden bepaald.

---

## Hardware

Gebruikt:

* MKS XDRIVE MINI
* Baxter-motor
* Hall-sensoren van de motor
* Motorvoeding
* Computer met `odrivetool 0.5.1`

### Bedrading

De motor moest eerst worden aangesloten op de controller. Omdat de bestaande bedrading niet direct geschikt was, moesten er eerst draden worden gesoldeerd.

Dit ging niet direct soepel doordat het soldeertin geoxideerd was.

Een belangrijk punt tijdens het aansluiten was dat er geen betrouwbare documentatie beschikbaar was over welke draad bij welk hall-signaal hoorde.

---

# Motorconfiguratie

Na verschillende tests lijken de volgende waarden het beste bij deze motor te passen:

| Instelling               | Waarde |
| ------------------------ | -----: |
| Pole pairs               |    `6` |
| Encoder type             |   Hall |
| CPR                      |   `36` |
| Encoder bandwidth        |  `100` |
| Velocity limit           |    `7` |
| Velocity gain            | `0.01` |
| Velocity integrator gain |    `0` |
| Velocity ramp rate       |    `1` |

Deze waarden zijn **experimenteel bepaald** en moeten niet automatisch als universele instellingen voor andere motoren worden beschouwd.

---

# 1. Motor en encoder configureren

De basisconfiguratie:

```python
odrv0.axis0.requested_state = AXIS_STATE_IDLE

# --- Motor ---
odrv0.axis0.motor.config.pole_pairs = 6
odrv0.axis0.motor.config.motor_type = <MOTOR_TYPE_HIGH_CURRENT of MOTOR_TYPE_GIMBAL>
odrv0.axis0.motor.config.current_lim = <A>
odrv0.axis0.motor.config.calibration_current = <A>

# overige motor-/voedingsinstellingen volgens de gebruikte opstelling

# --- Encoder: hall-sensor ---
odrv0.axis0.encoder.config.mode = ENCODER_MODE_HALL
odrv0.axis0.encoder.config.cpr = 36
odrv0.axis0.encoder.config.use_index = False
odrv0.axis0.encoder.config.bandwidth = 100

# --- Regelaar: snelheid ---
odrv0.axis0.controller.config.control_mode = CONTROL_MODE_VELOCITY_CONTROL
odrv0.axis0.controller.config.input_mode = INPUT_MODE_VEL_RAMP
odrv0.axis0.controller.config.vel_ramp_rate = 1
odrv0.axis0.controller.config.vel_limit = 7
odrv0.axis0.controller.config.vel_gain = 0.01
odrv0.axis0.controller.config.vel_integrator_gain = 0

odrv0.save_configuration()
```

Na `save_configuration()` wordt het bord opnieuw opgestart. Daarna moet opnieuw verbinding worden gemaakt.

---

# Belangrijke ontdekking: encoder mode

Een van de belangrijkste ontdekkingen tijdens dit experiment was dat de encoder correct als hall-sensor moet worden ingesteld:

```python
odrv0.axis0.encoder.config.mode = ENCODER_MODE_HALL
```

Wanneer de encoder op `ENCODER_MODE_INCREMENTAL` blijft staan, ontstaan in closed-loop-control fouten zoals:

```text
CONTROL_DEADLINE_MISSED
```

Interessant genoeg draait de motor met `LOCKIN_SPIN`, dus zonder encoderfeedback, wel soepel.

Dit was een belangrijke aanwijzing dat het probleem waarschijnlijk in de encoderconfiguratie zat en niet in de motor zelf.

---

# 2. Motor en encoder kalibreren

### Motorcalibratie

```python
import time

odrv0.axis0.requested_state = AXIS_STATE_MOTOR_CALIBRATION
time.sleep(10)

print(odrv0.axis0.current_state)
dump_errors(odrv0)
```

Daarna de encoder-offset kalibreren:

```python
odrv0.axis0.requested_state = AXIS_STATE_ENCODER_OFFSET_CALIBRATION
time.sleep(10)

print(odrv0.axis0.current_state)
dump_errors(odrv0)
```

De verwachte toestand na een succesvolle kalibratie is:

```text
1 = IDLE
```

en:

```text
no error
```

---

# 3. Kalibratie controleren

De belangrijkste controles:

```python
odrv0.axis0.motor.is_calibrated
odrv0.axis0.encoder.is_ready
odrv0.axis0.encoder.config.offset
odrv0.axis0.encoder.error
```

Resultaat:

```text
motor.is_calibrated  → True
encoder.is_ready     → True
encoder.error        → 0
```

De encoder-offset was op dit bord:

```text
42
```

Deze waarde is **bord- en motorgebonden** en mag niet zonder meer worden overgenomen.

---

# 4. Kalibratie opslaan

Na een succesvolle kalibratie:

```python
odrv0.axis0.motor.config.pre_calibrated = True
odrv0.axis0.encoder.config.pre_calibrated = True

odrv0.save_configuration()
```

Na het opnieuw opstarten:

```python
odrv0.axis0.encoder.config.mode
odrv0.axis0.encoder.config.cpr
```

Resultaat:

```text
ENCODER_MODE_HALL
36
```

---

# 5. Eerste draaitest

Voor de eerste test heb ik de motor bewust met een lage snelheid gestart:

```python
dump_errors(odrv0, True)

odrv0.axis0.controller.input_vel = 0
odrv0.axis0.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL

odrv0.axis0.controller.input_vel = 3
```

De motor begon daadwerkelijk te draaien.

### Motor stoppen

```python
odrv0.axis0.controller.input_vel = 0
odrv0.axis0.requested_state = AXIS_STATE_IDLE
```

### Veilig starten

Begin altijd met:

```python
input_vel = 0
```

en gebruik de velocity ramp.

Een directe sprong van `0` naar een hoge snelheid zorgt voor een grotere belasting van de regelaar.

---

# 6. Velocity controller testen

Verschillende waarden voor `vel_gain` zijn getest.

| `vel_gain` | Resultaat                                                |
| ---------: | -------------------------------------------------------- |
|     `0.01` | Stabiel                                                  |
|    `0.015` | Draait, maar meer rimpel                                 |
|     `0.02` | Instabiel, `OVERSPEED` binnen ongeveer een halve seconde |

Voor de integrator:

| `vel_integrator_gain` | Resultaat                 |
| --------------------: | ------------------------- |
|                   `0` | Geen restfout gemeten     |
|                `0.15` | Veel te hoog, `OVERSPEED` |

Voorlopig lijkt daarom:

```text
vel_gain = 0.01
vel_integrator_gain = 0
```

de beste configuratie.

---

# 7. Encoder bandwidth testen

Ook de encoder bandwidth is getest.

| Bandwidth | Resultaat                            |
| --------: | ------------------------------------ |
|      `50` | Te traag, lus faalt bij hogere winst |
|     `100` | Beste resultaat                      |
|    `200+` | Niet beter, soms onrustiger          |

Voorlopig gebruik ik:

```python
odrv0.axis0.encoder.config.bandwidth = 100
```

---

# 8. Gedrag van de motor

Tijdens het testen viel het volgende op:

* Onder ongeveer **2 toeren/s** begint de motor nauwelijks echt te draaien.
* Rond **3,5–6 toeren/s** draait de motor het soepelst.
* Vanaf ongeveer **5–6 toeren/s** begint de motor weer te schokken.

Dit betekent dat de optimale snelheid niet simpelweg de laagste of hoogste snelheid is.

---

# 9. Werkelijke snelheid meten

`encoder.vel_estimate` blijkt bij deze hall-sensorconfiguratie behoorlijk te rimpelen.

Met slechts **36 tellingen per omwenteling** is dit geen goede maatstaf voor de korte-termijnsnelheid.

Daarom heb ik de encoder-teller over langere tijd gemeten:

```python
def echte_snelheid(vel, wacht=5, n=5):
    ax = odrv0.axis0

    ax.controller.input_vel = 0
    ax.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
    ax.controller.input_vel = vel

    time.sleep(wacht + abs(vel))

    res = []

    for _ in range(n):
        c0 = ax.encoder.shadow_count
        t0 = time.time()

        time.sleep(1)

        res.append(
            (ax.encoder.shadow_count - c0)
            / 36
            / (time.time() - t0)
        )

    ax.controller.input_vel = 0
    ax.requested_state = AXIS_STATE_IDLE

    return sum(res) / len(res), res
```

Test:

```python
for v in [1, 2, 3, 4, 5, 6]:
    gem, _ = echte_snelheid(v)
    print(v, round(gem, 3))
```

Bij een opdracht van 3 toeren/s werd onder andere gemeten:

```text
[2.996, 2.995, 2.993, 2.997, 3.080,
 2.997, 2.997, 2.997, 2.968, 2.994]
```

Het gemiddelde ligt hiermee dicht bij de gevraagde 3 toeren/s.

Deze meting zegt echter alleen iets over het **gemiddelde toerental**. Het zegt niet dat de motor binnen één omwenteling volledig gelijkmatig draait.

Schokken moeten daarom ook op gehoor en gevoel worden beoordeeld.

---

# 10. Problemen

## `MOTOR_ERROR_CONTROL_DEADLINE_MISSED`

Mogelijke oorzaken:

* Verkeerde `encoder.config.mode`
* Problemen met de hall-bedrading
* Problemen met pull-ups
* Liveplotter die problemen veroorzaakt

Controleer bijvoorbeeld:

```python
odrv0.axis0.encoder.config.mode
```

---

## `CONTROLLER_ERROR_OVERSPEED`

Mogelijke oplossingen:

* `vel_gain` verlagen
* `vel_integrator_gain` verlagen
* `INPUT_MODE_VEL_RAMP` gebruiken
* Encoder-offset opnieuw kalibreren
* Controleren of de motorfasen correct zijn aangesloten

---

## `AXIS_ERROR_CONTROLLER_FAILED`

Dit lijkt een gevolg te zijn van een onderliggende controllerfout.

Controleer:

```python
odrv0.axis0.controller.error
```

---

# 11. Fouten uitlezen

Alle fouten weergeven:

```python
dump_errors(odrv0)
```

Fouten wissen:

```python
dump_errors(odrv0, True)
```

---

# 12. Werkende configuratie bewaren

Een belangrijk resultaat van dit experiment is dat de werkende configuratie moet worden vastgelegd.

De volledige configuratie kan worden bekeken met:

```python
odrv0.axis0.motor.config
odrv0.axis0.encoder.config
odrv0.axis0.controller.config
```

De waarden die afhankelijk zijn van de specifieke motor of het specifieke bord moeten bij het configureren van een nieuw bord opnieuw worden gecontroleerd.

---

# Resultaat

De eerste motor draait succesvol met de MKS XDRIVE MINI.

De huidige experimentele configuratie:

| Parameter                | Waarde |
| ------------------------ | -----: |
| Pole pairs               |    `6` |
| Encoder                  |   Hall |
| CPR                      |   `36` |
| Encoder bandwidth        |  `100` |
| Velocity limit           |    `7` |
| Velocity gain            | `0.01` |
| Velocity integrator gain |    `0` |
| Velocity ramp rate       |    `1` |

Deze configuratie werkt voor deze specifieke motor en moet voorlopig worden beschouwd als een **experimenteel resultaat**, niet als definitieve documentatie.

## Volgende experimenten

* [ ] Groter snelheidsbereik testen
* [ ] Oorzaak van het schokken bij hogere snelheid onderzoeken
* [ ] Hall-sensor signaal verder analyseren
* [ ] Meer motoren testen
* [ ] Configuratie vergelijken tussen verschillende motoren
* [ ] Werkende configuratie omzetten naar officiële documentatie

{% include links.html %}
