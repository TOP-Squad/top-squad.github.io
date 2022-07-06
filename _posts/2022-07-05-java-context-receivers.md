---
layout: post
title:  "Context Receivers in Java?"
author: sander.hautvast
categories: [ Java, Discussion ]
image: assets/images/joao-henrique-3EKQxtOEXHw-unsplash.jpg
beforetoc: "Double Curly Brace Initialization. Een obscuur taal element van java, dat je meer zou moeten gebruiken"
featured: true
hidden: false
rating: 4.5
---
Toen ik las dat het concept 'Context Receiver' werd geintroduceerd in [Kotlin](https://blog.jetbrains.com/kotlin/2022/02/kotlin-1-6-20-m1-released/), was mijn eerste gedachte: 'Dat ken ik nog van Pascal!' 

Ik heb sinds de jaren negentig geen Pascal meer gezien (waar blijft de tijd?), maar het bleek te kloppen:

 
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
[https://smartpascal.github.io/help/assets/with.htm](https://smartpascal.github.io/help/assets/with.htm)


De Kotlin syntax is in de basis het zelfde:

```kotlin
with(loggingContext) {
        startBusinessOperation()
        ...
}
```
[https://blog.jetbrains.com/kotlin/2022/02/kotlin-1-6-20-m1-released/](https://blog.jetbrains.com/kotlin/2022/02/kotlin-1-6-20-m1-released/)

Ik geloof dat onder water pascal meer op _javascript_ lijkt in het opzoeken van variabelen in verschillende contexten, dan _kotlin_ of _java_, maar dit concept en het keyword zijn dezelfde. Je creëert een context waarbinnen een object instantie (tijdelijk) de nieuwe `this` wordt. 

Maar daar gaat deze post niet over. In java bestaan er geen context receivers, dus einde verhaal zou je denken.

Totdat ik toevallig hier tegenaan liep:

```java
new ArrayList<Integer>() { {
   add(1);
   add(2);
} };
```
[https://stackoverflow.com/questions/1958636/what-is-double-brace-initialization-in-java](https://stackoverflow.com/questions/1958636/what-is-double-brace-initialization-in-java)

 
`Double curly brace initialization` (waarom heb ik dit nooit eerder gezien?) is een combinatie van twee enkelvoudige curly braces ...doh!... accolades in goed Nederlands!

1. één voor een anonymous inner class
2. één voor een initializer block

Ik zie beide vrij weinig in gangbare codebases. En de combinatie is helemaal zeldzaam. _Waarom?_

Afgezien van het geringe 'side-effect' dat je een subclass instantieert in plaats van het type zelf, zijn er voor zover ik kan bedenken geen nadelen voor deze werkwijze. Dit zorgt er natuurlijk wel voor dat je het niet op `final` classes kan toepassen.
 
Dus, neem deze code (realistisch voorbeeld uit een Spring `Configuration` class):


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
    return new ThreadPoolTaskScheduler() { {
        setPoolSize(4);
        setThreadNamePrefix("AdmissionsAPI-");
    } };
}
```
 

En dat ziet er min of meer uit als de kotlin en pascal voorbeelden hierboven. Het is minder tekst om te verwerken in je hoofd en je hebt de  _mutable_ lokale variabele `threadPoolTaskScheduler` niet meer nodig. Hoe minder lokale variabelen, hoe minder _moving targets_ is één van mijn persoonlijke stokpaardjes (_Used wisely_).

Een verschil is natuurlijk wel dat je deze 'truuk' alleen direct na instantiatie kunt toepassen.

 
**Maar, Ojee! iemand noemt het een [anti-pattern](https://blog.jooq.org/dont-be-clever-the-double-curly-braces-anti-pattern/).**

Wat zijn de argumenten van deze auteur?
1. Het is minder leesbaarder.

2. Je creëert, zoals ik al aangaf een inner subclass, wat extra overhead is voor de classloader en de garbage collector

3. De inner subclass bevat een referentie naar het object waar je hem geïnstantieerd hebt. Dat geeft weer extra overhead en leidt tot memory leaks.

 
Mijn tegenargumenten:
1. Het is even wennen, maar als je het patroon herkent, is het juist leesbaarder. [_Delete as much code as you can_](https://matt-rickard.com/reflections-on-10-000-hours-of-programming/)

2. Klopt in theorie. Maar weegt dit op tegen #1? Dit lijkt me een gevalletje _premature optimization_

3. Memory leaks? Really? Ik heb ze gezien, van dichtbij. Daar zaten nooit inner classes bij. Oja `Hashmap.Entry`, natuurlijk, want leaks zitten altijd in een `Collection`. Met andere woorden, de JDK zit vol met inner classes (wat te denken van jdk8 lambdas...). De auteur maakt op geen enkele manier duidelijk hoe die zouden kunnen leiden tot problemen (en dit is ook niet zo). Onnodige bangmakerij! Zolang je je variabelen declareert in de context van een method (bijvoorbeeld voor een http request), is er zowieso nauwelijks kans op dit soort leaks, want aan het einde wordt deze variabele opgeruimd (_eligible for garbage collection_).

**Conclusie:**

Je kunt in java iets doen wat ergens wel op context receivers lijkt. De belangrijkste reden om dat te doen is om de code leesbaarder te maken. De nadelen zijn verwaarloosbaar. Je moet wel even je collega's inlichten, want ik geef toe, het ziet er op het eerste gezicht niet java-achtig uit!
	
**Edit:**

Je hebt natuurlijk wel allerlei alternatieven. Bijvoorbeeld `List.of()` als je een `Collection` wil instantieren met waardes. `@PostConstruct` is sowieso gangbaarder en meestal beter dan een `initializer block` (want alle _wiring_ is al gebeurd). En het gebruik van `method chaining` (zoals in builders) vermindert ook herhaling van de instantie waar je mee werkt. 