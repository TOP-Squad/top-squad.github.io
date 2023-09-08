---
layout: post
title:  "Test is Productie"
author: sander.hautvast
categories: [testautomatisering]
image: assets/images/test_is_prod/nico-smit-HjFUevA2g1k-unsplash.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
##### Het is 2023. Waarom gebeurt dit nog steeds in testcode? 
* slechte code kwaliteit
* geen scheiding van testdata en techniek (gluecode)
* slecht/geen design
* breekbaar ('brittle') -> false positives/negatives
* de tests zijn niet in lijn met de productiecode
* geen duidelijk inzicht in succes/falen
* slechte coverage (alleen de happy flows) en/of geen enkel inzicht in de coverage

##### Het kan zo mooi zijn:
* elke story bevat een test-taak (= aanpassing aan de testcode)
* de test-code is onderhoudbaar (en dat gebeurt dus ook)
* veel testcases
* foutloze uitvoering geeft vertrouwen om live te gaan
* duidelijke rapportage
* een goede basis: cucumber of cypress bijvoorbeeld

Ik wil zo ver gaan dat dit meer waard is dan een hele verzameling unittests, maar daar mag je anders over denken. Ik heb zelf teveel unittests gezien die de structuur van de code testen in plaats van de uitvoering. Voegt weinig toe en levert bij aanpassingen meestal meer werk op dan het aanpassen van de productiecode zelf. Heb je veel mocks? Go figure!

##### code kwaliteit
Denk aan alles wat je in de 'echte' code niet meer mag:
* lange bestanden
* onjuist commentaar
* uitgecommentarieerde code
* onduidelijke naamgeving
* 'lelijke code': achterhaalde constructies, code smells
* duplicatie
* geen logging
* de code bevat secrets

>Test code moet net zo onderhoudbaar zijn als productiecode, anders verliest het gaandeweg zijn waarde. Reserveer tijd voor onderhoud. Refactor de testcode.

##### scheiding van testdata en techniek (gluecode)
Meestal heb je dan niet gebruik gemaakt van een framework zoals cucumber. Het is bijvoorbeeld een enkel script dat alles doet. Het gevolg: 
* iemand wordt de de-facto tester, want alleen hij/zij snapt de code
* het is vervelend om nieuwe testgevallen toe te voegen, dus doen we dat maar niet

>Testen is een team-verantwoordelijkheid. Ook als je een tester in je team hebt. Het scheiden van de data draagt enorm bij aan de kwaliteit. Vergemakkelijkt het testwerk.

Copy-paste van een regel data (met een kleine aanpassing)? Moet kunnen. Goede naamgeving van je testgevallen? Doen.

##### design
Een testsuite (al dan niet met een framework) bevat net zoveel design als een reguliere applicatie. Denk aan de project-structuur. Is techniek gescheiden van functionaliteit? Is de functionaliteit netjes ingedeeld?
Maar ook: een python script, geschreven door een (ervaren) java developer. Die denkt in classes, overerving. Maar de generieke parents bevatten allerlei details die alleen relevant zijn voor een enkele child-class.

>Het is allemaal software en ga er dus net zo kritisch mee om.

##### brittleness, breekbaarheid
Deze term wordt wel eens gebruikt als het gaat om de hoeveelheid aanpassingen in de testcode ten opzichte van de productiecode. Maar breekbaarheid komt over de hele linie voor:
* Flaky tests, die af en toe een 503 opleveren in plaats van een 404 en niemand snapt waarom, maar als je de pipeline opnieuw draait, gaat het meestal wel goed... 
* Of je bent afhankelijk van externe services, onderhouden door andere teams voor het genereren van testdata. 
* Of je acceptatie systeem wordt elke maand ververst met productiedata. En dat dan één systeem niet bijgewerkt is.
* Of je test tegen een hele pyramide van backends, die allemaal up en in sync moeten zijn.
* Of je test tegen mockservers die net iets anders doen dan de echte.

Al deze fenomenen maken dat wijzigingen in de productiecode vertraging oplopen door de test. Het testwerk wordt bepalend voor de team velocity. Testen komt onder druk te staan, omdat nieuwe features live moeten. Uiteindelijk ga je dit merken in de vorm van productie incidenten.

>Dit kan een heel breed en moeilijk te veranderen aspect zijn. Er is vaak niet één oplossing. De context speelt een belangrijke rol. Een microservice architectuur heeft ook nadelen. Praat met andere teams. Zorg voor draagvlak.

##### 'de tests kloppen niet'
Meegemaakt: er komt een nieuwe tester, die niet overweg kan met de bestaande suite. Hij is zo eigenwijs om een nieuwe te beginnen. De tests zijn allemaal goed, maar de oude doen het niet meer, terwijl ze toch nog steeds een deel van de bestaande code afdekken en de nieuwe niet!

Of je zit in een strak scrum-regime en een sprint van twee weken is niet genoeg om de feature te bouwen en te testen. Het team besluit om eerst een sprint te bouwen en de volgende te testen. Dat levert frictie op. Context-switches bij de teamleden. Ruis.

>Stelling: Testen is heilig, maar het proces (agile, etc) niet. Kijk wat werkt. _Inspect and adapt_!

Voorwaarde: team autonomie. Niet een SAFE proces dat afdwingt dat alle teams in het zelfde ritme dansen. 

##### succes en falen
De testscripts zijn wel gedraaid, maar niemand heeft goed naar de resultaten gekeken. Bijvoorbeeld omdat die niet centraal opvraagbaar zijn. Omdat ze een op een andere machine staan. Omdat...

>Bouw test-evidence in je proces in. Zodat je per release weet of de tests gedraaid zijn en wat de resultaten waren.

##### coverage
Je moet het kunnen meten om het te weten. In een moderne IDE is het zo makkelijk als gesneden brood. En natuurlijk kan het ook op je testomgeving. 

>Testen moet gelijk op gaan met bouwen. Heb je een nieuwe feature? Schrijf een test. Vind je een bug? Schrijf een test. En maak het inzichtelijk.

Sonar is een bekende tool voor code kwaliteit, waaronder de testcoverage. Hoewel dit product al jaren bestaat, heb ik zelden of nooit gezien dat het goed wordt ingezet en meerwaarde biedt. En de testcode zelf wordt natuurlijk niet geanalyseerd...

#### kortom
Behandel je tests als productie!

<div style="text-align: right">∞</div>