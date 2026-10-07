---
title: "Baxter reboot"
keywords: Baxter, Rethink Robotics, robotica, B'Robotics, robot rehabilitation, 
tags: [getting_started]
sidebar: mydoc_sidebar
permalink: index.html
---

<div class="home">

<div class="page-header">
    <h1>Baxter Reboot</h1>
    <p class="lead">
        Een oude Rethink Robotics Baxter weer tot leven brengen.
    </p>
</div>

<img src="{{ '/images/baxter.jpeg' | relative_url }}"
     alt="Rethink Robotics Baxter robot"
     style="max-width: 100%;">

<hr />

<hr />

<h2>Van stofhappen naar demo's</h2>

<p>
    Dit project draait om een oude Rethink Robotics Baxter
    robot die na jaren trouwe dienst buiten gebruik is geraakt.
    De harde schijf van de robot is defect geraakt en daarmee is ook
    de originele softwareomgeving verloren gegaan.
</p>

<p>
    Mijn stagebedrijf <strong>B'Robotics</strong> wil deze Baxter graag
    weer kunnen gebruiken voor demonstraties. Mijn opdracht is daarom
    om de armen van de robot opnieuw aan de praat te krijgen en een
    nieuwe softwarebasis op te bouwen.
</p>

<p>
    Het doel is niet simpelweg om de oude software opnieuw na te bouwen.
    Ik wil een systeem maken dat robuust, modulair en
    toekomstbestendig is, zodat de robot ook in de toekomst
    verder ontwikkeld en uitgebreid kan worden.
</p>

<h2>De uitdaging</h2>

<p>
    Zonder de originele harde schijf ontbreekt de
    oorspronkelijke software en configuratie. Daardoor moet alles van
    de werking opnieuw worden uitgezocht.
</p>

<p>
    Dat betekent experimenteren met de hardware, motorcontrollers,
    encoders, communicatie en software. Soms werkt iets meteen,
    soms kost het dagen om uit te zoeken waarom iets niet werkt.
</p>

<p>
    Juist dat proces vormt een belangrijk onderdeel van dit project.
</p>

<h2>Mijn doel</h2>

<p>
    Uiteindelijk moet Baxter weer zelfstandig kunnen functioneren
    tijdens demonstraties. Daarvoor werk ik aan een nieuwe software-
    architectuur waarin verschillende onderdelen van de robot zoveel
    mogelijk onafhankelijk van elkaar kunnen functioneren.
</p>

<ul>
    <li>De armen en motoren weer betrouwbaar laten bewegen</li>
    <li>Een robuuste besturing ontwikkelen</li>
    <li>Hardware en software modulair opbouwen</li>
    <li>De afhankelijkheid van verouderde software verminderen</li>
    <li>De code begrijpelijk en onderhoudbaar houden</li>
    <li>Een basis creëren waarop later nieuwe functies kunnen worden gebouwd</li>
</ul>

<h2>Van experiment naar werkende robot</h2>

<p>
    Ik documenteer het project op drie verschillende manieren.
</p>

<div class="row">

<div class="col-md-4">
    <h3>Lab Notes</h3>
    <p>
        Mijn technische werknotities. Hier staan experimenten,
        metingen, configuraties, problemen en dingen die ik tijdens
        het ontwikkelen ontdek.
    </p>
    <p>
        <a href="{{ '/lab_notes_home.html' | relative_url }}">
            Bekijk de Lab Notes →
        </a>
    </p>
</div>

<div class="col-md-4">
    <h3>News</h3>
    <p>
        De belangrijkste mijlpalen van het project. Hier schrijf ik
        over nieuwe resultaten en momenten waarop er daadwerkelijk
        iets vooruitgaat.
    </p>
    <p>
        <a href="{{ '/news.html' | relative_url }}">
            Bekijk het laatste nieuws →
        </a>
    </p>
</div>

<div class="col-md-4">
    <h3>Documentation</h3>
    <p>
        De uiteindelijke technische documentatie. Hier komen de
        onderdelen die voldoende getest en stabiel zijn om door
        anderen gebruikt of nagebouwd te worden.
    </p>
    <p>
        <a href="{{ '/mydoc_introduction.html' | relative_url }}">
            Bekijk de documentatie →
        </a>
    </p>
</div>
```

</div>

<hr />

<h2>De eerste stap is gezet</h2>

<p>
    Een van de eerste doelen was om één van de Baxter-motoren weer
    daadwerkelijk te laten draaien. Met een nieuwe motorcontroller,
    de <strong>MKS XDRIVE MINI</strong>, en de bestaande hall-sensoren
    is dat inmiddels gelukt.
</p>

<p>
    Dat is nog maar één motor, maar het bewijst dat de hardware opnieuw
    aangestuurd kan worden. Vanaf hier begint het echte werk:
    meerdere motoren betrouwbaar laten samenwerken en uiteindelijk
    de armen van Baxter weer laten bewegen.
</p>


<hr />

<h2>Over dit project</h2>

<table>
    <tr>
        <th>Robot</th>
        <td>Rethink Robotics Baxter</td>
    </tr>
    <tr>
        <th>Stagebedrijf</th>
        <td>B'Robotics</td>
    </tr>
    <tr>
        <th>Doel</th>
        <td>Baxter weer geschikt maken voor demonstraties</td>
    </tr>
    <tr>
        <th>Focus</th>
        <td>Robuuste, modulaire en toekomstbestendige software</td>
    </tr>
    <tr>
        <th>Status</th>
        <td>In ontwikkeling</td>
    </tr>
</table>

<hr />

</div>


{% include links.html %}
