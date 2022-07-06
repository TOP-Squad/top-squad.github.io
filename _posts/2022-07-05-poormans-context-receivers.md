---
layout: post
title:  "Poor man's Context receivers in Java"
author: Sander
categories: [ Java, Discussion ]
tags: [red, yellow]
image: assets/images/contextmatters.jpg
description: "Is dit een bruikbaar pattern?"
featured: false
hidden: false
rating: 4.5
---
Toen ik las dat het concept 'Context Receiver' werd geintroduceerd in [Kotlin](https://blog.jetbrains.com/kotlin/2022/02/kotlin-1-6-20-m1-released/), was mijn eerste gedachte: 'Dat ken ik nog van Pascal!' Ik heb sinds de jaren negentig geen Pascal meer gezien (waar blijft de tijd?), maar het bleek te kloppen:

 

```pascal
type
  // Declare a customer record
  TCustomer = Record
    firstName : string[20];
    ...
  end;
var
  John : TCustomer;


begin
  With John do
  begin
    firstName := 'John';
    ...
  end;
```

https://smartpascal.github.io/help/assets/with.htm

 

De Kotlin syntax is in de basis het zelfde:

```kotlin
with(loggingContext) {
        startBusinessOperation()
        ...
}
```

Ik geloof dat onder water pascal meer op javascript lijkt in het opzoeken van variabelen in verschillende contexten, dan kotlin of java, maar dit concept en het keyword zijn dezelfde.

Maar daar gaat deze post niet over. In java bestaan er geen context receivers, dus einde verhaal zou je denken.

Totdat ik toevallig hier tegenaan liep:

```java
new ArrayList<Integer>() {
   add(1);
   add(2);
}};
```

https://stackoverflow.com/questions/1958636/what-is-double-brace-initialization-in-java

 

`Double curly brace initialization` (waarom heb ik dit nooit eerder gezien?) is een combinatie van twee enkelvoudige curly braces ...doh!... Accolades in goed Nederlands!

1. één voor een anonymous inner class

2. één voor een initializer block

 

En afgezien van het geringe 'side-effect' dat je een subclass instantieert in plaats van het type zelf, zijn er voor zover ik kan bedenken geen nadelen voor deze werkwijze. Het zorgt er natuurlijk wel voor dat je het niet op `final` classes kan toepassen.

 

Dus, neem deze code (realistisch voorbeeld uit een spring `Configuration` class):

 

```java
@Bean
public ThreadPoolTaskScheduler threadPoolTaskScheduler() {
    ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();
    threadPoolTaskScheduler.setPoolSize(4);
    threadPoolTaskScheduler.setThreadNamePrefix("AdmissionsAPI-");
    return threadPoolTaskScheduler;
}
```

 

Dat kún je dus veranderen in dit:

```java
@Bean
public ThreadPoolTaskScheduler threadPoolTaskScheduler() {
    return new ThreadPoolTaskScheduler() {
        setPoolSize(4);
        setThreadNamePrefix("AdmissionsAPI-");
    }};
}
```
 

En dat ziet er min of meer uit als de kotlin en pascal voorbeelden hierboven. Bijkomend voordeel is dat je de mutable lokale variabele `threadPoolTaskScheduler` kwijtraakt. Hoe minder lokale variabelen, hoe minder _moving targets_ is één van mijn persoonlijke stokpaardjes (_Used wisely_).

 
Een verschil is natuurlijk wel dat je deze 'truuk' alleen direct na instantiatie kunt toepassen.

 
Ooeps! iemand noemt het een [anti-pattern](https://blog.jooq.org/dont-be-clever-the-double-curly-braces-anti-pattern/).

Argumenten:
1. Het is leesbaarder maar dat is niet belangrijk.
2. Je creëert, zoals ik al aangaf een inner subclass, wat extra overhead is voor de classloader en de garbage collector
3. De inner subclass bevat een referentie naar het object waar je hem geïnstantieerd hebt. Dat geeft weer extra overhead en leidt tot memory leaks.

 

Mijn tegenargumenten:

1. Je schrijft code voor je collega's, niet voor computers. Leesbaarheid is altijd belangrijk. _Delete as much code as you can_ (https://matt-rickard.com/reflections-on-10-000-hours-of-programming/)

2. Klopt natuurlijk. Maar weegt dit op tegen #1? Dit lijkt me een gevalletje _premature optimization_

3. Memory leaks? Really? Ik heb ze gezien, van dichtbij. Daar zaten nooit inner classes bij. Oja `Hashmap.Entry`, natuurlijk, want leaks zitten altijd in een `Collection`. Met andere woorden, de JDK zit vol met inner classes (wat te denken van jdk8 lambdas...). De auteur maakt op geen enkele manier duidelijk hoe zou leiden tot problemen. Onnodige bangmakerij.

4. Oké, in de context van EJB's (waar de auteur vandaan lijkt te komen, we hebben het over 2014), heb ik wel eens nare problemen gehad met subclasses. Maar die kwamen door JEE zelf (als gevolg van AOP), niet door curly braces.

 

Conclusie: Er sterven geen katjes door double curly braces.

	
	
