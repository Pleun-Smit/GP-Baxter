---
title: "Het eerste gewricht beweegt"
published: true
permalink: /bewegend-gewricht.html
summary: "In deze post vertel ik hoe de rechterschouder (S0) van Baxter voor het eerst beweegt via ros2_control, en hoe ik de overbrenging van het gewricht heb gemeten."
tags: [news]
---


# Het eerste gewricht beweegt

Een motor die los op tafel draait is één ding, maar een gewricht van Baxter dat echt beweegt is iets heel anders. Op **10/11/2026** was het zover: de rechterschouder (**S0**) beweegt, aangestuurd via ROS 2.

## Van testtool naar ros2_control

In de vorige posts stuurde ik de motoren aan met een eigen testtool en later met het [dashboard](can-dashboard.html). Voor de echte robot moet alles via **ros2_control** lopen, zodat ROS 2 de arm kan aansturen zoals elke andere robotarm.

Daarvoor heb ik de CAN-code omgebouwd tot een hardware-interface. In `baxter_arm_bringup/urdf/baxter_arm.ros2_control.xacro` staat per joint hoe de motor aan het gewricht gekoppeld is:

* de **overbrenging** (hoeveel motoromwentelingen per omwenteling van het gewricht)
* de **draairichting**
* de **limieten** in software
* de **beginstand** (home)

## Video van het bewegende gewricht

<iframe width="463" height="823" src="https://www.youtube.com/embed/jDIuxBRL4OQ" title="Gewricht S0 beweegt!" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> **Video:** de rechterschouder (S0) van Baxter beweegt voor het eerst via ros2_control.

## De overbrenging meten

Ook hier was er geen documentatie, dus ik moest zelf uitzoeken hoeveel de motor moet draaien voor één omwenteling van het gewricht. Dat heb ik zo gemeten:

* de motor een vast aantal omwentelingen laten draaien en de encoder laten tellen: **94 motoromwentelingen**;
* de hoek van de bovenarm meten met een kompas-app, meerdere keren: **79–82°**;
* daaruit de overbrenging berekenen.

Dat geeft een overbrenging van ongeveer **423 : 1** (±2%).

De metingen waren goed herhaalbaar, maar een kompas-app is niet ideaal naast staal en motoren. Die kunnen het kompas steeds op dezelfde manier verstoren. Voordat ik precieze posities nodig heb, controleer ik de hoek daarom nog een keer met een winkelhaak of een gyroscoop-app.

## Wat betekent 423 : 1?

Zo'n grote overbrenging heeft voor- en nadelen:

* **Langzaam:** 0,1 rad/s op het gewricht is al **6,7 omwentelingen per seconde** op de motor. Met de snelheidslimiet van 15 omwentelingen per seconde haalt het gewricht maximaal ongeveer **±0,22 rad/s** (zo'n 13° per seconde).
* **Lage snelheden zijn lastig:** de motor regelt slecht onder 3–4 omwentelingen per seconde. Heel langzaam bewegen van het gewricht gaat dus nog niet soepel.
* **Nauwkeurig:** de hall-sensoren geven maar 36 tellingen per omwenteling, maar door de overbrenging is dat op het gewricht ongeveer **0,02° per telling**.

## Limieten en beginstand

* De limiet in software staat op **±1,60 rad** (origineel was dat ±1,70 rad).
* De beginstand is **0,7854 rad** (45°): dan wijst de bovenarm recht naar voren.

Dat laatste lijkt vreemd, maar in het originele model van Baxter is de rechterarm 45° gedraaid gemonteerd. Bij S0 = 0 wijst de arm dus schuin naar voren-rechts.

## Nog een les over ID's

In een [eerdere post](can-motor.html) leerde ik al dat een verkeerd node-ID stil faalt. Bij meerdere controllers op de bus kwam daar een nieuw probleem bij: sommige borden reageerden niet op commando's, terwijl ze wel gewoon op de CAN-bus zichtbaar waren.

De oorzaak: de MKS XDRIVE MINI heeft **twee verschillende ID's**, en ik had alleen de eerste aangepast. Daardoor reageerden meerdere borden op berichten die niet voor hen bedoeld waren. Nadat ik beide ID's goed had ingesteld, werkte het wel.

## Wat nu?

De volgende stappen:

* de hoek van S0 controleren met een winkelhaak of gyroscoop-app
* de overbrenging meten voor de andere gewrichten van de arm
* latency en jitter meten over CAN, om te laten zien dat CAN mijn eisen (> 100 Hz, < 5 ms latency, < 1 ms jitter) wel haalt
* de regeling bij lage snelheid soepeler krijgen

{% include links.html %}
