---
title: "Een motor aansturen over CAN"
published: true
permalink: /can-motor.html
summary: "In deze post vertel ik hoe ik de Baxter-motor via een CAN-bus heb laten draaien met C++ en een USB-naar-CAN-adapter."
tags: [news]
---


# Een motor aansturen over CAN

Een motor die draait via `odrivetool` is mooi, maar uiteindelijk moet Baxter niet aan een USB-kabel en een Python-console hangen. De volgende stap was daarom om dezelfde motor aan te sturen over een **CAN-bus**.

Op **15/10/2026** was het zover: de motor draait nu via CAN, aangestuurd vanuit C++ op mijn laptop.

## Van USB naar CAN

De **MKS XDRIVE MINI** blijkt een ODrive 3.6-kloon te zijn en praat daarom het **CANSimple**-protocol van ODrive. Dat scheelde veel werk: er hoefde geen eigen protocol te worden bedacht, alleen de juiste berichten moesten worden verstuurd.

Het ID van elk bericht bestaat uit het **node-ID** van de as en een **commandonummer**. Zo kun je meerdere assen en borden op dezelfde bus aansturen, zolang elk een uniek ID heeft.

De opzet:

* **USB-naar-CAN-adapter** op de laptop, als `can0` onder Linux
* **250 kbit/s** op de bus
* **C++ met SocketCAN**, verpakt als ROS 2-pakket (`colcon build`)
* Een kleine testtool om de verbinding te controleren en de motor aan te sturen

## Eerst de verbinding testen

Met een `scan` luistert de tool naar de heartbeats van de borden. Bij mij kwamen daar twee nodes uit (ID `1` en `16`). Daarna kon ik de motor met een paar commando's laten draaien:

```bash
ros2 run baxter_can can_test can0 scan
ros2 run baxter_can can_test can0 vel 1 1.0
ros2 run baxter_can can_test can0 est 1
ros2 run baxter_can can_test can0 stop 1
```

Met andere woorden: scannen, een toer per seconde draaien, positie en snelheid uitlezen en weer stoppen.

## Video van twee draaiende motoren over CAN

<iframe width="1344" height="756" src="https://www.youtube.com/embed/QMBG14gXi5w" title="Twee motoren aansturen over CAN" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Een les over node-ID's

Mijn eerste poging werkte niet. De motor deed niets en de tool meldde alleen `no reply`. De oorzaak: ik stuurde commando's naar node `3`, terwijl de borden op `1` en `16` zaten. Het bord negeert berichten voor een ander ID gewoon, zonder foutmelding.

Gebruik dus altijd eerst `scan` om te zien welke nodes er echt zijn.

## Naar een positie

Naast snelheid heb ik ook commando's toegevoegd om een bepaald aantal toeren te draaien of naar een positie te gaan. Het bord plant dan zelf een zacht traject met een maximale snelheid en versnelling. Die onderdelen moeten nog uitgebreider getest worden, want met slechts 36 tellingen per omwenteling van de hall-sensoren blijft positieregeling grof.

## Wat nu?

De volgende stappen:

* de positiecommando's verder testen en afstellen
* meerdere motoren tegelijk op één bus aansturen
* de code ombouwen naar een `ros2_control`-hardware-interface voor Baxter

De volledige stappen, commando's en foutopsporing staan in de [**Lab notes: motor aansturen via CAN**]({% link _posts/2026-10-15-LN-Motor-over_CAN.md %}).

{% include links.html %}