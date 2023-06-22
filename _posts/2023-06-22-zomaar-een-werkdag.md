---
layout: post
title:  "Zomaar een werkdag"
author: sander.hautvast
categories: [craftsmanship]
image: assets/images/clem-onojeghuo-hdW4rZPHe2g-unsplash.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
9.00 De teksten en plaatjes, die de customer journey expert had opgestuurd in de frontend verwerkt. Na een flink aantal refinements zijn we deze week begonnen met bouwen. Een wijziging in de flow voor gezamenlijke rekeningen. Behalve de frontend ook drie nieuwe endpoints en de ontsluiting ervan in de gateway.


9.45 standup, korte discussie of we de backend eerst moeten mocken in de gateway, om sneller iets te hebben wat demobaar is. Ik maak aardige voortgang met mijn stukje frontend, maar de backend loopt vertraging op, omdat de developers eerst nog een API live moeten zetten. Onze _azure devops_ pipeline is best wel _flaky_. Kan aan van alles liggen. Deze week loopt _skopeo copy_ vaak mis (openshift). Geen idee waarom. We zijn devops, maar alle infra voorzieningen zijn als centrale voorziening buiten onze controle. Lijkt eigenlijk best wel op hoe het vroeger ging, 'toen je nog beheerders had'. 

O ja, er stonden laatst vreemde meldingen in de productielog (_kibana_). Mijn mededeling geeft wat fronsende gezichten. Klanten die zichzelf lijken op te zoeken als mederekeninghouder?? Het staat er toch echt! Snel wat voorbeelden zoeken en opsturen.

10.30 Na de standup even gesynchroniseerd met m'n directe collega die ook een stukje frontend doet. Alle SVG's moeten ook als JS dependency worden opgenomen in de code. Klinkt apart, maar werkt prima.

12.00 Iets wat tussendoor komt. Laatst was er een vulnerability ontdekt in _snappy-java_. Dat is een library die in tomcat voor http-compressie wordt gebruikt. Ik was uit interesse gaan kijken op github en daar zag ik toevallig dat iemand klaagde dat het in _GraalVM_ niet werkt. Omdat snappy de classloader gebruikt om native libraries te gebruiken, en GraalVM AOT compilatie doet is alles met reflectie en resources (en JNI) niet zonder meer mogelijk. Ik had de documentatie opgezocht en op github gepost wat ik dacht dat de oplossing was (een cmdline optie op _native_image_). Het antwoord was helaas: 'werkt niet' en nu jeuken mijn vingers voor het vinden van de oplossing. Dus in de middagpauze GraalVM installeren en een mini applicatietje maken (2 regels code). Werkt lekker, maar alleen met die cmdline opties!

Morgen worden alle _signers_ in de certificaten vernieuwd en moeten alle applicaties gedeployed zijn met de nieuwe truststores. De wijziging hadden we gecombineerd met een aanpassing in de service discovery (of eigenlijk gaan we dat nu pas gebruiken voor deze integratie). Die liep ook weer achter. Ik dacht dat ik het twee weken geleden voor elkaar had, maar ondanks dat de test goed leek, was het niet zo (Auw!). We hebben een dependency op een library X die een soort gezamenlijk bezit is, lees vaak breaking updates. Ik had gehoopt deze dependency eruit te kunnen gooien, omdat er een vage fout was opgetreden, maar helaas... M'n collega lukte het gelukkig om de fout te verhelpen, maar vervolgens, vandaag dus, werkt de deployment niet meer. Eén dag voor de deadline geeft dat best wat stress. 

13.00 We deployen vanaf master om te kijken of het aan de applicatie ligt of de pipeline. Nieuwe vulnerabilities die we alleen kunnen _whitelisten_. Skopeo copy loopt weer fout, herstarten. Nu lukt de deployment wel. Mijn vermoeden is dat library X de oorzaak is. Als ik develop lokaal bouw, zie ik allerlei _spring_ dependency updates binnenkomen. Klinkt eng, want dat kan conflicteren met het framework. Ik kan dat niet onderbouwen want er staan nergens fouten in de logging. Het feit dat de deployment nu wel lukt, ondersteunt mijn vermoeden, maar mijn collega denkt dat er iets anders aan de hand is, iets vaags in de pipeline, maar dat zou betekenen dat de fout random optreedt. Ik weet niet wat je liever hebt.

Uiteindelijk besluiten we de truststore update te scheiden van die in de service discovery. Collega maakt een nieuwe branch op master, met de truststore wijzigingen (minimaal), PR maken en weer deployen. Gaat goed, nou ja een toch nog wat herstarts hier en daar. En altijd wachten op de product owner, om de pipeline approvals te geven. Deployen naar productie kan best een dagje (wachten) kosten.

14.00 Nu kan ik weer verder met de frontend. Het is best lastig om een button te laten zorgen voor een alternatieve flow, maar uiteindelijk blijkt dat ik net de verkeerde component gebruik, én de volgorde van de event afhandeling is anders dan ik dacht. Staart me al een uur aan vanaf de console.log...

15.00 meeting over de frontend. De UX designer kan er niet bij zijn. Ik laat zien wat ik tot nog toe gemaakt heb. De CJE stelt vragen en heeft wat kleine opmerkingen. We stemmen af over de backend integratie. 

Intussen de productie deployment vanaf de zijlijn in de gaten houden. Lijkt OK. De tekstuele wijzigingen van vanmorgen nog even afronden.

16.30 Ik wil de frontend wijzigingen op de testomgeving zetten. Hè bah,  weer vergeten eerst _lint_ te draaien vóór het pushen en het starten van de pipeline. Die faalt nu met een stuk of veertig fouten... Sommige zijn optisch: CamelCase in plaats van snake_case. Kennelijk moeten alle imports de .js extensie bevatten. Andere, circulaire dependencies?? Blijkt dat ik een constante heb toegevoegd in een util.js. Andere plek geeft dezelfde fout. Ik heb nu geen zin om dit góed op te lossen, en zet het wel op de plek waar het gebruikt wordt (2 stuks). Dat lossen we later wel op. Nu eerst testen. O nee, nu eerst eten!