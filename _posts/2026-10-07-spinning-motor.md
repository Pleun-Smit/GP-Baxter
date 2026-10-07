---
title: "Eerste stapjes: een draaiende motor"
published: true
permalink: /spinning-motor.html
summary: "In deze post vertel ik hoe ik een van de motoren van Baxter voor het eerst aan het draaien heb gekregen."
tags: [news]
---


# Eerste stapjes: een draaiende motor

Op **05/10/2026** kwamen de nieuwe **MKS XDRIVE MINI** motorcontrollerborden binnen. Tijd om te kijken of ik eindelijk een van de motoren van Baxter aan het draaien kon krijgen.

Dat bleek iets meer uitzoekwerk te zijn dan verwacht.

## Van losse draden naar een draaiende motor

Na het aansluiten en solderen van de motor begon het echte experimenteren. Er is namelijk maar weinig documentatie beschikbaar over de gebruikte motoren. Zaken zoals het aantal *pole pairs*, de encoderinstellingen en de juiste instellingen voor de hall-sensoren moest ik daarom zelf achterhalen.

Na verschillende tests kwam ik uit op onder andere:

* **6 pole pairs**
* **36 CPR**
* **Hall-sensor als encoder**
* **vel_gain: 0.01**

Met deze instellingen draait de motor nu daadwerkelijk in closed-loop control.

## De eerste draaiende motor

Video van de eerste succesvolle test:

<iframe width="462" height="822" src="https://www.youtube.com/embed/nb4o4B2ZByU" title="1 keer draaiende baxter motor" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> **Video:** eerste succesvolle test van een Baxter-motor met de MKS XDRIVE MINI.

## Een kleine maar belangrijke stap

Het klinkt misschien als een klein resultaat: *een motor die draait*.

Maar voor dit project is het een belangrijke eerste stap. De combinatie van de Baxter-motor, hall-sensoren en de MKS XDRIVE MINI werkt daadwerkelijk samen.

De motor draait het soepelst rond **3,5–6 toeren per seconde**. Bij hogere snelheden begint hij weer wat meer te schokken, dus er is nog genoeg om verder te onderzoeken.

De technische details, configuratie en experimenten heb ik bijgehouden in de [**Lab notes: spinning motor**]({% link _posts/2026-10-07-LN-spinning-motor.md %}).

## Wat nu?

De volgende stap is om de motor verder te testen en de regeling over een groter snelheidsbereik stabiel te krijgen. Daarna wordt het interessant om meerdere motoren van Baxter tegelijk aan te sturen.

{% include links.html %}
