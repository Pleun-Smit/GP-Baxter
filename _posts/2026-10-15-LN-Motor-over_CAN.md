---
title: "Motor aansturen via CAN - MKS XDRIVE MINI"
published: true
category: lab-notes
permalink: /can-motor-ln.html
summary: "Een Baxter-motor aansturen over een CAN-bus met een MKS XDRIVE MINI, SocketCAN, C++ en ROS 2: opzet, commando's en foutopsporing."
tags: [lab-notes]
---

# Motor aansturen via CAN - MKS XDRIVE MINI

**Datum:** 15/10/2026
**Hardware:** MKS XDRIVE MINI (2 stuks), USB-naar-CAN-adapter
**Motor:** Baxter-motor (hall-sensoren)
**Software:** `odrivetool 0.5.1`, Ubuntu, ROS 2, C++17 (SocketCAN), `colcon`

> Dit experiment bouwt voort op [Eerste draaiende motor - MKS XDRIVE MINI](spinning-motor-ln.html). De motor draait daar al via `odrivetool` over USB. Hier gaat het erom dezelfde motor over **CAN** aan te sturen.

## Doel

Het doel was om:

1. de MKS XDRIVE MINI-borden via een CAN-bus te laten communiceren met mijn laptop;
2. testberichten te sturen om de verbinding te controleren;
3. de motor via CAN te laten draaien (snelheid) en naar een positie of een aantal toeren te laten bewegen;
4. dit in **C++** te doen, zodat het later in **ROS 2** (`ros2_control`) kan worden gebruikt.

---

## Achtergrond: welk protocol praat het bord?

De MKS XDRIVE MINI is een ODrive 3.6-kloon en draait ODrive-firmware. Daarom spreekt het bord het **CANSimple**-protocol van ODrive. Er hoeft dus geen eigen protocol te worden bedacht, alleen de juiste frames moeten worden verstuurd.

Een CAN-frame bestaat hier uit:

```text
CAN-ID (11 bit) = (node_id << 5) | command_id
Data            = little-endian, maximaal 8 bytes
```

Voorbeeld voor node `1`: `node_id << 5` = `0x20`. Het commando `Set_Axis_State` (`0x07`) krijgt dan ID `0x027`.

> De command-ID's in dit document gelden voor de **0.5.x**-firmware. Heeft jouw bord een andere versie, controleer ze dan in de ODrive-documentatie van die versie.

---

## Hardware

Gebruikt:

* 1 of 2 MKS XDRIVE MINI-borden
* Baxter-motor, aangesloten zoals in de vorige lab-note
* Motorvoeding
* USB-naar-CAN-adapter
* Laptop met Ubuntu en ROS 2 (in mijn geval een VM; dan moet de USB-adapter aan de VM worden doorgegeven)

### CAN-bedrading

* Verbind **CAN-H met CAN-H** en **CAN-L met CAN-L** tussen adapter en bord(en).
* Zet **120 Ω terminatie** aan beide uiteinden van de bus. Meet je met een multimeter tussen CAN-H en CAN-L, dan hoor je ongeveer **60 Ω** te zien als beide uiteinden zijn afgesloten.
* Alle apparaten op de bus moeten **dezelfde bitrate** gebruiken. Ik gebruik **250000 bit/s**.

---

# 1. Het bord voor CAN configureren (`odrivetool`)

Verbind het bord via USB en start `odrivetool`. De motor-, encoder- en regelaarsinstellingen uit de vorige lab-note blijven hetzelfde:

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

Voor CAN komen daar de volgende instellingen bij. De namen zijn voor `odrivetool 0.5.1`. Gebruik tab-completion (`odrv0.can.<TAB>`) om te controleren of ze in jouw versie zo heten:

```python
# Elk bord / elke as heeft een uniek node-ID op de bus
odrv0.axis0.config.can_node_id = 1

# Bitrate van de bus
odrv0.can.set_baud_rate(250000)

# Hoe vaak het bord een heartbeat stuurt (ms)
odrv0.axis0.config.can_heartbeat_rate_ms = 100

odrv0.save_configuration()
```

Na `save_configuration()` herstart het bord. Maak daarna opnieuw verbinding.

**Belangrijk:**

* Heb je meer borden, geef elk een **ander node-ID**. Twee apparaten met hetzelfde ID verstoren elkaar.
* Een bord heeft twee assen. Elke as heeft een eigen node-ID (`axis0` en `axis1`).
* Zorg dat de motor al succesvol is gekalibreerd en dat `pre_calibrated = True` is opgeslagen (zie de vorige lab-note). Zo kan de motor direct in closed-loop worden gezet.

---

# 2. CAN-interface op Linux activeren

Voor een **candleLight / gs_usb**-adapter (verschijnt direct als `can0`):

```bash
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 250000
ip link show can0        # moet "UP" tonen
```

Voor een **seriële slcan**-adapter:

```bash
sudo slcand -o -c -s5 /dev/ttyUSB0 can0     # -s5 = 250 kbit/s
sudo ip link set can0 up
```

> Draait Linux in een VM, geef de USB-adapter dan eerst door aan de VM. Zonder dat verschijnt `can0` niet.

---

# 3. Snelle test zonder eigen code (`can-utils`)

Met `can-utils` kun je de verbinding controleren voordat er C++ aan te pas komt:

```bash
sudo apt install can-utils
candump can0
```

Het bord stuurt periodiek een **heartbeat** (command `0x01`). Bij node `1` zie je frames met ID `021`. De eerste 4 bytes zijn de `axis_error`, byte 4 is de `axis_state` (1 = idle, 8 = closed loop).

Je kunt de motor ook met losse frames aansturen. Voorbeeld voor node `1` (let op: de float `1.0` is little-endian `00 00 80 3F`):

```bash
# Closed-loop control aanzetten (Set_Axis_State = 8)
cansend can0 027#08000000

# Control mode = velocity (2), input mode = passthrough (1)
cansend can0 02B#0200000001000000

# Snelheid 1.0 toeren/s (float 1.0 + torque feed-forward 0)
cansend can0 02D#0000803F00000000

# Stoppen en terug naar idle
cansend can0 02D#0000000000000000
cansend can0 027#01000000
```

Beweegt de motor hiermee, dan werkt de hele keten (adapter, bus, node-ID, bitrate, firmware) en is de rest alleen nog software.

---

# 4. De C++-code bouwen met `colcon`

Het pakket `baxter_can` bevat:

| Bestand | Doel |
| ------- | ---- |
| `include/baxter_can/can_interface.hpp` | `CanFrame`, `ICanInterface`, `SocketCanInterface` (SocketCAN-wrapper) |
| `include/baxter_can/odrive_motor.hpp` | `MotorController`: codeert en decodeert CANSimple-berichten |
| `src/can_test.cpp` | Testprogramma voor op de command line |
| `src/motor_node.cpp` | Minimale ROS 2-node (snelheidscommando in, `joint_states` uit) |

Dit past op het klassendiagram: `ICanInterface` en `SocketCanInterface` zijn ongewijzigd overgenomen, en `MotorController` doet de `encodeCommand`/`decodeStatus` van het bord volgens CANSimple.

Bouwen:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
tar xzf /pad/naar/baxter_can.tar.gz

source /opt/ros/<distro>/setup.bash        # bijv. humble of jazzy
cd ~/ros2_ws
colcon build --packages-select baxter_can
source install/setup.bash
```

Ontbreekt `colcon`: `sudo apt install python3-colcon-common-extensions`.
Bouwfout over `rclcpp` of `ament_cmake`: je bent vergeten `/opt/ros/<distro>/setup.bash` te sourcen **voor** het bouwen.

In elke nieuwe terminal opnieuw:

```bash
source /opt/ros/<distro>/setup.bash
source ~/ros2_ws/install/setup.bash
```

---

# 5. Verbinding testen met `scan`

```bash
ros2 run baxter_can can_test can0 scan
```

Resultaat bij mij:

```text
found node 1 (state 1, error 0x0)
found node 16 (state 1, error 0x0)
```

Dit betekent dat er twee nodes op de bus zitten (state `1` = idle, geen fouten).

### Belangrijkste les: gebruik het juiste node-ID

Mijn eerste pogingen mislukten omdat ik commando's naar **node 3** stuurde, terwijl die niet op de bus bestond. Het resultaat:

```text
$ ros2 run baxter_can can_test can0 est 3
no reply
```

Er kwam geen foutmelding, het bord negeerde de berichten simpelweg. **Gebruik altijd een node-ID dat `scan` heeft gevonden.** Weet je niet welke as bij welk ID hoort, vraag het dan op in `odrivetool`:

```python
odrv0.axis0.config.can_node_id
odrv0.axis1.config.can_node_id
```

---

# 6. De motor laten draaien via CAN

In de commando's hieronder vervang je `<node>` door jouw node-ID (bij mij kwamen `1` en `16` uit de scan).

### 6.1 Kalibreren (alleen nodig als dat nog niet is opgeslagen)

```bash
ros2 run baxter_can can_test can0 state <node> 4     # motorcalibratie
ros2 run baxter_can can_test can0 state <node> 7     # encoder-offsetcalibratie
ros2 run baxter_can can_test can0 listen             # wacht tot state=1, error=0x0
```

`state <node> 3` doet de volledige kalibratiereeks in één keer. Wacht daarna ongeveer 10 seconden en controleer in `listen` of de heartbeat weer `state=1` en `error=0x0` toont. Staat er een andere foutcode, dan is de kalibratie mislukt. Gebruik dan `dump_errors(odrv0)` in `odrivetool` om de oorzaak te zien.

### 6.2 Snelheid

Begin **laag**, net als in de vorige lab-note:

```bash
ros2 run baxter_can can_test can0 vel <node> 1.0    # 1 toer/s
ros2 run baxter_can can_test can0 vel <node> 3.0    # 3 toeren/s
```

Dit commando doet drie dingen achter elkaar: `Set_Controller_Modes` (velocity, passthrough), `Set_Axis_State` (closed loop) en `Set_Input_Vel`.

**Let op:** de CAN-tool zet de input mode op `PASSTHROUGH`. De `INPUT_MODE_VEL_RAMP` uit de `odrivetool`-configuratie wordt daarmee dus **niet** gebruikt en de motor krijgt een directe sprong. Begin daarom bij lage snelheden, of voeg een eigen ramp toe in de software.

Houd rekening met de bevindingen uit de vorige lab-note: onder ongeveer 2 toeren/s draait de motor nauwelijks soepel, tussen 3,5 en 6 toeren/s het best, en daarboven begint het te schokken. De `vel_limit` van `7` blijft de bovengrens.

### 6.3 Uitlezen

```bash
ros2 run baxter_can can_test can0 est <node>
```

Voorbeeld van de uitvoer:

```text
pos=12.3456 turns vel=2.9970 turns/s
```

Het commando stuurt een **RTR-frame** (`Get_Encoder_Estimates`) en wacht op het antwoord. Net als in de vorige lab-note zal `vel` bij 36 tellingen per omwenteling flink rimpelen. Gebruik voor een betrouwbare snelheid het gemiddelde over langere tijd.

### 6.4 Stoppen

```bash
ros2 run baxter_can can_test can0 stop <node>       # snelheid 0 + idle
ros2 run baxter_can can_test can0 clear <node>      # fouten wissen
ros2 run baxter_can can_test can0 estop <node>      # noodstop
```

Na een `estop` moet de as eerst worden gewist (`clear`) en opnieuw worden ingeschakeld.

---

# 7. Naar een positie of aantal toeren bewegen

Voor positieregeling gebruikt de code een **trapezoidaal traject** (`INPUT_MODE_TRAP_TRAJ`). Het bord plant zelf de versnelling, de constante snelheid en de vertraging, op basis van de snelheids- en versnellingslimieten die worden meegestuurd. Dat is veel zachter dan een directe positiesprong.

```bash
# 2 volle toeren vooruit, vanaf de huidige positie
ros2 run baxter_can can_test can0 turn <node> 2.0

# een halve toer terug, max 0,5 toeren/s, versnelling 1 toer/s²
ros2 run baxter_can can_test can0 turn <node> -0.5 0.5 1.0

# naar absolute positie 3,0 toeren
ros2 run baxter_can can_test can0 pos <node> 3.0
```

De optionele argumenten na het doel zijn de maximale snelheid (toeren/s, standaard `1.0`) en de versnelling (toeren/s², standaard `2.0`). Het programma wacht tot de motor binnen 0,01 toer van het doel is en toont de actuele positie.

Wat er gebeurt:

1. `Set_Controller_Modes`: positie-regeling + trapezoidaal traject
2. `Set_Traj_Vel_Limit` (`0x11`) en `Set_Traj_Accel_Limits` (`0x12`)
3. `Set_Axis_State`: closed loop
4. `Set_Input_Pos` (`0x0C`) met het doel
5. Positie periodiek uitlezen tot het doel is bereikt

Eigenschappen en aandachtspunten:

* Posities zijn in **motortoeren**. 0,25 toer = 90°. Zit er een tandwielkast achter, deel dan door de overbrengingsverhouding.
* **Absolute positie = positie sinds het opstarten.** Een motor met alleen hall-sensoren heeft geen indexpuls, dus na het uit- en aanzetten is de referentie weg.
* De gevraagde snelheid moet **lager zijn dan `vel_limit`** (hier `7`), anders ontstaat een fout of loopt de motor vast.
* Door de lage resolutie (36 tellingen per omwenteling) is de positieregeling grof. Gaat de motor slingeren, verlaag dan `pos_gain` en `vel_gain` in `odrivetool` en verhoog ze daarna stap voor stap.
* **Dit gedeelte heb ik minder uitgebreid getest dan de snelheidsregeling.** Begin met kleine bewegingen (bijv. `turn <node> 0.25`).

---

# 8. Overzicht van de commando's

Alle commando's: `ros2 run baxter_can can_test <can-interface> <commando> ...`

| Commando | Wat het doet |
| -------- | ------------ |
| `listen` | Toont alle frames op de bus en decodeert heartbeats (state en error) |
| `scan` | Luistert 2 s en toont welke node-ID's een heartbeat sturen |
| `state <node> <n>` | Vraagt as-state `n` aan: 1 idle, 3 volledige kalibratie, 4 motorcalibratie, 7 encoder-offsetcalibratie, 8 closed loop |
| `vel <node> <toeren/s>` | Velocity mode, closed loop en snelheidssetpoint |
| `pos <node> <toeren> [vel] [acc]` | Absolute positie met trapezoidaal traject |
| `turn <node> <delta> [vel] [acc]` | Zelfde als `pos`, maar relatief ten opzichte van de huidige positie |
| `est <node>` | Vraagt encoderwaarden op: positie (toeren) en snelheid (toeren/s) |
| `clear <node>` | Wist fouten op de as |
| `stop <node>` | Zet snelheid op 0 en de as in idle |
| `estop <node>` | Noodstop |

---

# 9. Wat er over de bus gaat (CANSimple)

| Cmd-ID | Naam | Data |
| -----: | ---- | ---- |
| `0x01` | Heartbeat (bord naar host) | `u32 axis_error`, as-state in byte 4 |
| `0x02` | Estop | geen |
| `0x07` | Set_Axis_State | `u32 state` |
| `0x09` | Get_Encoder_Estimates | aanvragen met RTR-frame; antwoord `float pos`, `float vel` (toeren, toeren/s) |
| `0x0B` | Set_Controller_Modes | `i32 control_mode` (1 koppel, 2 snelheid, 3 positie), `i32 input_mode` (1 passthrough, 5 trapezoidaal traject) |
| `0x0C` | Set_Input_Pos | `float pos`, `i16 vel_ff`, `i16 torque_ff` |
| `0x0D` | Set_Input_Vel | `float vel`, `float torque_ff` |
| `0x0F` | Set_Limits | `float vel_limit`, `float current_limit` |
| `0x11` | Set_Traj_Vel_Limit | `float` |
| `0x12` | Set_Traj_Accel_Limits | `float accel`, `float decel` |
| `0x17` | Get_Vbus_Voltage | aanvragen met RTR-frame; antwoord `float` |
| `0x18` | Clear_Errors | geen |

Voorbeeld: `vel 1 1.0` stuurt (voor node 1) `Set_Controller_Modes` (ID `0x02B`), `Set_Axis_State` = 8 (ID `0x027`) en `Set_Input_Vel` = 1.0 (ID `0x02D`).

---

# 10. ROS 2-node

```bash
ros2 run baxter_can motor_node --ros-args -p node_id:=<node> -p can_interface:=can0
```

In een tweede terminal:

```bash
ros2 topic pub /joint_cmd_vel std_msgs/msg/Float64 "{data: 1.0}" -r 10
ros2 topic echo /joint_states
```

De node:

* zet de as bij het starten in closed loop (velocity mode);
* stuurt `/joint_cmd_vel` (toeren/s) door naar de motor;
* publiceert `/joint_states` (positie en snelheid in radialen) op basis van `Get_Encoder_Estimates`;
* zet de snelheid op 0 als er 0,5 s geen commando binnenkomt (eerste versie van een watchdog/`SafetyMonitor`).

Dit is het uitgangspunt voor een `ros2_control` `SystemInterface` (`BaxterCanHardware`).

---

# 11. Problemen

| Probleem | Mogelijke oorzaak en oplossing |
| -------- | ------------------------------ |
| `no reply` of de motor reageert niet | Verkeerd node-ID. Voer `scan` uit en gebruik een gevonden ID |
| `scan` vindt niets | Controleer CAN-H/CAN-L, terminatie, bitrate aan beide kanten en `ip link show can0` (moet `UP` zijn). Bij een VM: USB-adapter doorgegeven? |
| `can0` bestaat niet | Adapter niet herkend (VM-doorgifte, driver) of een slcan-adapter waarvoor `slcand` nog niet is gestart |
| Kalibratiefout | `dump_errors(odrv0)`; controleer encoder-mode (`ENCODER_MODE_HALL`), CPR `36`, pole pairs `6`, calibratiestroom |
| `CONTROLLER_ERROR_OVERSPEED` na een `vel`-commando | De CAN-tool gebruikt passthrough, dus geen ramp: begin lager of verlaag `vel_gain` |
| Motor schokt of slingert in positiemodus | Verlaag `pos_gain` en `vel_gain`; de hall-resolutie is laag |
| Positie wordt niet bereikt (timeout) | `vel_limit` lager dan gevraagde snelheid, of de as staat in een foutstatus: bekijk `listen` en `dump_errors` |
| Foutstatus blijft hangen | `clear <node>` of in `odrivetool` `dump_errors(odrv0, True)` |

---

# Resultaat

De Baxter-motor kan nu via CAN worden aangestuurd vanaf mijn laptop, in C++ en met een opzet die direct in ROS 2 past.

| Onderdeel | Status |
| --------- | ------ |
| CAN-bus (adapter, bitrate 250000, node-ID's) | Werkt, `scan` vindt de borden |
| Snelheid sturen via CAN | Werkt, motor draait |
| Encoderwaarden uitlezen via CAN | Geïmplementeerd |
| Positie / aantal toeren via CAN | Geïmplementeerd, nog verder te testen |
| ROS 2-node | Minimale versie, nog te testen met meerdere motoren |

De gebruikte waarden (node-ID's, bitrate, motorparameters) zijn **experimenteel bepaald** voor deze opstelling en moeten bij een ander bord of een andere motor opnieuw worden gecontroleerd.

## Volgende experimenten

* [ ] Positie- en `turn`-commando's uitgebreid testen
* [ ] `pos_gain` en `vel_gain` afstellen voor soepele positieregeling met hall-sensoren
* [ ] Eigen snelheidsramp in de software toevoegen
* [ ] Meerdere motoren tegelijk op één CAN-bus aansturen
* [ ] ROS 2-node uitbreiden met `/joint_cmd_pos` en een action voor "ga N toeren"
* [ ] Omzetten naar een `ros2_control` `SystemInterface` (`BaxterCanHardware`)
* [ ] Veiligheidsfuncties (`SafetyMonitor`): heartbeat- en foutbewaking

{% include links.html %}