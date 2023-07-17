---
layout: post
title:  "Mag ik mijn monoliet terug?"
author: sander.hautvast
categories: [craftsmanship]
image: assets/images/harry-grout-ivfmiQ5FbHw-unsplash.jpg
beforetoc: ""
featured: false
hidden: true
lang: nl
---
Dit jaar is applicatie X bij webwinkel Y ongeveer 20 jaar oud. Toen ik er begon in 2002 was het project nog net niet begonnen en ongeveer twee jaar later, na de nodige _ups en downs_, een _pas op de plaats_  en een _heroriëntatie_ ging het live. Ik werk er niet meer, maar heb er tien jaar van mijn leven besteed en voor zover ik weet werkt het nog steeds, ondanks pogingen om ervan af te komen.

Dit is niet het verhaal van die applicatie.

Natuurlijk was het een monoliet. Iets anders kenden we niet. Dat wil zeggen er was een backend en een frontend voor het callcenter (dat er toen nog was), allebei gedeployed in een bekend merk 'webcontainer'. Omdat internet wel iets leek te worden werd de website ook aangesloten. Deze was al eerder gebouwd en in plaats van met het mainframe werd het op applicatie T aangesloten. Bedrijf X voerde daarvoor de nodige wijzigingen door en iedereen was gelukkig.

Ik verliet bedrijf W om bij deze X te gaan werken, om vijf jaar later weer bij W terug te keren. Wat ik bij terugkomst zag was dat de applicatie architectuur in wezen nog dezelfde was, met meer toeters, ik bedoel functionaliteit en meer bellen, daarmee bedoel ik koppelingen met andere applicaties. Het voelde als een huis waar in de loop van de tijd een uitbouw achter bij was gebouwd, en daarna nog één ervoor, vijf dakkappellen, een extra schoorsteen en een hele donkere kelder. 

Een nieuwe versie live zetten was een project op zich. Dat gebeurde ongeveer twee keer per jaar en naarmate de datum dichterbij kwam werd iedereen steeds zenuwachtiger. Alle tests op de diverse frontends waren handmatig, of het was een uur handwerk om de juiste _fitnesse suite_ te vinden die wél werkte. Er was een minutieus plan waar iedereen, ontwikkelaars, testers, mainframe beheerders, projectleiders, teamleiders, een Change Manager, zeg maar heel IT bij betrokken was. Dan was het nachtwerk, met _points of no return_, _groene lichten_ en _rollback beslissingen_. Ik kan me nog herinneren dat iedereen in mineur was omdat iets niet leek te werken, wat zorgde voor een _abort_ en dus geen nieuwe versie de volgende morgen, maar dat achteraf bleek dat het wél had gewerkt. Jammer dus!

Ik weet eerlijk gezegd niet of ik hier echt naar terug wil...

Het is hier fantástisch! CI/CD, microservices, cloud, _infrastructure as code_, virtualisatie, pipelines. Tik alle buzzwords maar af. Agile. Spotify-model. Tribes, squads, chapters. Grote organisatie, internationaal, veel teams, zowel in Nederland als diverse andere landen. 
We kunnen elk moment live. Het duurt wel een half uur om een nieuwe versie op Test en/of Acceptatie te krijgen. Static code checks, security scans. Dat betekent dat je opnieuw moet beginnen met de hele deployment (weer een half uur dus) nadat je de onveilige dependency op een whitelist hebt gezet, zodat het tóch mag (als er nog geen nieuwe versie is).

Ach, de deployment is toch weer gefaald omdat iets in AWS het niet meer doet. Opnieuw proberen, of de slack geschiedenis doorspitten om te kijken of andere mensen het ook hebben gemeld. Want je zou eens iets dubbel melden en de _gurus_ boos maken. Je bent zo een dag verder. 
Ach er is een handige feature om automatisch de nieuwe truststores te downloaden. Inbouwen maar. Ja we moeten toch elke maand deployen, want anders gaan de systemen mails sturen en komt je API op een rode lijst. 

 0: aload_0
         1: invokespecial #10                 // Method java/lang/Object."<init>":()V
         4: return
