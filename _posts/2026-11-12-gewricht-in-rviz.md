---
title: "Het gewricht beweegt, en nu ook in RViz"
published: true
permalink: /gewricht-rviz.html
summary: "In deze post laat ik zien hoe de echte beweging van Baxters rechterschouder (S0) nu ook in RViz te zien is, en waarom dat zo handig is."
tags: [news]
---


# Het gewricht beweegt, en nu ook in RViz

In de [vorige post](bewegend-gewricht.html) kreeg ik de rechterschouder (**S0**) van Baxter voor het eerst in beweging via ros2_control. De volgende stap: de beweging ook op mijn scherm zien. Op **12/11/2026** was het zover: als de echte arm beweegt, beweegt het 3D-model van Baxter in **RViz** mee.

## Video: de echte arm en RViz naast elkaar

<!-- Vervang VIDEO_ID door het ID van je YouTube-video -->
<iframe width="1344" height="756" src="https://www.youtube.com/embed/VIDEO_ID" title="Baxter S0 beweegt, ook in RViz" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> **Video:** de rechterschouder van Baxter beweegt, en het model in RViz beweegt mee.

## Van motor naar model

Wat je in RViz ziet, is geen simulatie maar de echte stand van de arm. De informatie gaat zo van de motor naar het scherm:

1. De controller (MKS XDRIVE MINI) meet met de hall-sensoren hoeveel de motor gedraaid heeft en stuurt dat via **CAN** naar de pc.
2. Mijn **hardware-interface** in ros2_control rekent de motoromwentelingen om naar een gewrichtshoek. Daarvoor gebruikt hij de overbrenging (**423 : 1**), de draairichting en de beginstand.
3. De **joint_state_broadcaster** publiceert die hoek op het topic `/joint_states`.
4. De **robot_state_publisher** berekent met het model van Baxter (URDF) waar elk onderdeel van de arm staat.
5. **RViz** tekent het model in die stand.

## Waarom dit zo handig is

RViz is voor mij vooral een controle of mijn metingen kloppen. Elke fout in de instellingen zie je meteen terug:

* **Verkeerde overbrenging:** het model draait verder of minder ver dan de echte arm.
* **Verkeerde draairichting:** het model draait de andere kant op.
* **Verkeerde beginstand:** het model staat steeds een vaste hoek scheef ten opzichte van de echte arm. Bij Baxter is dat extra belangrijk, omdat de rechterarm in het model **45° gedraaid** gemonteerd is.

Dat het model nu netjes met de echte arm meebeweegt, laat zien dat de overbrenging, richting en beginstand van S0 goed zijn ingesteld.

Het is ook handig voor de veiligheid. Voordat ik een gewricht laat bewegen, zie ik in RViz al hoe de arm staat en hoe ver hij nog van zijn limieten af is.

## Wat nu?

De volgende stappen:

* de hoek in RViz vergelijken met een meting met een winkelhaak of gyroscoop-app
* de andere gewrichten van de arm op dezelfde manier aansluiten, zodat de hele arm in RViz meebeweegt
* bewegingen vanuit ROS 2 aansturen en eerst in RViz controleren voordat de echte arm ze uitvoert

{% include links.html %}
