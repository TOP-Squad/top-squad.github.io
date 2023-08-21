---
layout: post
title:  "Een memoryleak op heterdaad betrappen"
author: sander.hautvast
categories: []
image: assets/images/daan-mooij-91LGCVN5SAI-unsplash.jpg
beforetoc: ""
featured: false
hidden: true
lang: nl
---
Performance is niet moeilijk. Het is of de database of garbage collection. Ok, het vinden en het oplossen van het probleem kan nog steeds lastig zijn. 
Als het gaat om memoryleaks, zijn er twee vragen om te beantwoorden:
1. Hebben we een memoryleak?
2. Waar zit het lek?

Er is al jaren allerlei tooling om beide vragen te beantwoorden. Voor de eerste vraag, bestaat dat enerzijds uit verbose GC logging en anderszijds uit de visualisatie ervan. Eén blik op het plaatje volstaat meestal om te zien waar je aan toe bent. De lijn in de grafiek gaat in een hobbelende lijn omhoog. Meestal niet in een spike, want dat duidt vaker op plotselinge overbelasting, en moet je gewoon de max heap vergroten. Maar het kan wel.

Als het gaat om java performance zijn er weinig mensen zo gespecialiseerd als Jack Shirazi, al sinds jaren de drijvende kracht achter [javaperformancetuning.com](http://www.javaperformancetuning.com/) (en ja, nog altijd )
