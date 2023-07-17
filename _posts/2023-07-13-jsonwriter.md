---
layout: post
title:  "Scherp je geest, met JSON"
author: sander.hautvast
categories: [json, unicode, reflection, performance]
image: assets/images/bruno-oliveira-7SFNYdcHI-s-unsplash.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
_Werkwijze_: Plan een dag vrij, zonder collega's of gezinsleden om je lastig te  vallen. Start de dag met een koude douche, een rondje door het park en dan pas je eerste koffie. Leg je telefoon weg en sluit al je browsertabs. Zet je favoriete teringherrie op (of niet) en trek je hackershoodie aan (of niet). Alles is bedoeld voor maximale concentratie...

Ik ben altijd gefascineerd geweest door JSON writers (serializers, marshallers, generators, mappers etc.). Parsers zijn ook interessant maar dat is een vak op zich. Misschien kom ik nog eens toe aan het lezen van [craftinginterpreters](http://www.craftinginterpreters.com/). 

Het mooie van JSON writers is:
- het is niet heel makkelijk, maar zeker niet onmogelijk voor de gemiddelde developer (denk ik?)
- de hoeveelheid code is te overzien
- afgebakende requirements
- referentie implementaties (om te vergelijken of spieken, mocht je dat willen)
- er zijn diverse design beslissingen die je moet nemen
- je kunt het in verschillende talen proberen (ik hier `java`)
- performance uitdagingen
- je leert iets wat bruikbaar zal blijken in je dagelijkse werk

Dit gaat niet allemaal lukken in een dag, maar dat is ook niet het doel. Het is een oefening om in de _zone_ te komen en je geest te scherpen. 


**De requirements**

Libraries als [jackson](https://github.com/FasterXML/jackson) zitten vol toeters en bellen. Annotaties, configuratie, extensies etc. Zou ik niet gelijk doen. 

Het is heel simpel: maak dit werkend: 
```java
public static void Json.write(OutputStream out, Object value){
    ...
}
```

Voor het gemak in deze blog doe ik het zo:

```java
public static String Json.write(Object value){
    ...
}
```


Allebei kan. Maar omdat een json writer doorgaans in een http server zit, zou ik het zo maken dat die daar het meest voor geschikt is, met een outputstream die bytes schrijft.

`Object` is dus een willekeurig java object. Voor de primitives moet je de methode overloaden, met dank aan het java typesystem...

Arrays, List, Map, Enum, beans, Records etc. Standaard objecten: `Long`, `String`, alle soorten datums, `BigInteger`, noem maar op, moeten standaard ondersteund worden. Uiteraard moet het resultaat correcte json zijn. Eventueel zou je de output in unittests kunnen vergelijken met die van een bestaande library. 

Vrije keuze oefening: lees [rfc4627](https://www.ietf.org/rfc/rfc4627.txt)

**De basics**

Begin simpel (en test-driven):
```java
assertEquals("null", Json.write(null));
```
Ik zou nu niet nadenken over een design en gewoon het simpelst mogelijke doen voor deze eerste userstory. Gaandeweg laten ontstaan en refactoren on-the-go met als vangrail de unittests.

```java
assertEquals("42", Json.write(42));
```
Dus alle acht primitives of alleen hun wrappers, maar autoboxing geeft iets extra overhead. En vervolgens:

```java
assertEquals("\"Zaphod Beeblebrox\"", Json.write("Zaphod Beeblebrox"));
```

Nog steeds makkelijk. Als het een String is, quotes eromheen.
Maar hou je vast, want we gaan de diepte in met speciale ascii karakters.

```java
assertEquals("\"\\\tArthur Dent\"", Json.write("\tArthur Dent"));
```

_true story:_

Ik was ze vergeten en ik was _over the moon_ van de resultaten: 10% sneller dan Jackson!!

Maar daarna: 15% trager...Meh

Het escapen van speciale karakters vereist namelijk dat je een String niet _in batch_ kunt overzetten, maar elk karakter moet inspecteren. 

> Hoe ga je dit doen? 

Loop door alle elementen van de String. Is het een speciaal karakter, verander het dan in de escape-variant. 
Dat kan met if/switch, met een mapping van inputchar naar outputchars, of met dezelfde mapping op basis van de integer ascii waarde. 

Het gebruik van een `Map` is in mijn ervaring wat je het meest ziet in dit soort situaties in enterprise code. Het is wel de meest trage. Beide, wat ouderwetsere manieren zijn sneller en ontlopen elkaar niet veel, maar een array(List) die je met de ascii waarde als index benadert, kost minder code.

```java
0: [0],
1: [1],
..
9: ["\\\t"], // 9 is de ascii waarde van [TAB]
..
``` 

_Vervelend_: We moeten een StringBuilder vullen met alle karakters of hun escape. Die laatste zijn altijd langer (2 karakters), dus _in-place_ vervangen kan niet (tenzij je ervoor kiest om de volgende karakters 1 char op te schuiven). Meh.

(Maar vergeet deze opmerking, als je toch de OutputStream variant gebruikt).

De rfc verplicht je alleen tot het escapen van bepaalde ascii waarden onder de 96, dus dat beperkt de grootte van de array die je moet maken. 

Andere unicode karakters, mag je escapen, maar dat móet niet. "😮‍💨" is even correcte JSON als "\uD83D\uDE2E‍\uD83D\uDCA8".

**Lijsten**

Lijsten in java komen in arrays of Lists, eventueel Iterators.
```java
assertArrayEquals("[1,1,2,3,5,8,13,21]", Json.write(new int[]{1, 1, 2, 3, 5, 8, 13, 21}));
```

Dit is interessanter dan je misschien denkt. Want als het goed is hebben we `write(int/Integer)` al geimplementeerd en roepen we deze aan voor elk element in de lijst. Sterker nog: wat doen we met `int[][]` en `int[][][]` ... ∞ ? 

> ... to recurse is divine

Waarom? Omdat recursie in staat is om te gaan met structuren die theorie oneindig zijn.

```java
import java.lang.reflect.Array;

public static String write(Object value){
    var builder = new StringBuilder();
    ...
    else if (value.getClass().isArray()){
        builder.append("[");
        StringJoiner joiner = new StringJoiner(",");
        for (int i = 0; i < Array.getLength(value); i++) {
            Object arrayElement = Array.get(value, i);
            joiner.add(Json.write(arrayElement)); // recursie
        }
        builder.append(joiner.toString());
        builder.append("]");
    }
    return builder.toString();
}
```
> In de praktijk is het aantal dimensies beperkt tot 255...😏

De java StringJoiner zit trouwens slim in elkaar. Hij berekent bijvoorbeeld de grootte van de output String aan de hand van het aantal elementen, de eventuele pre- en postfixes en de delimiter. Hij lost het eeuwenoude probleem op dat er na het laatste element geen komma komt.

Deze code werkt voor elke array, dankzij `java.lang.reflect.Array`... Uiteindelijk moet dat eruit want: veel te traag. Hoewel, is dat zo? Reflective method invocations zijn traag vanwege de runtime checks. Geldt dat hier ook?

 Benchmark | Mode  | Cnt  |  Score    | Error  |  Units
  -----------|-------|-----:|----------:|-------:|---------
   non reflective | avgt | 25 | 83,318   |± 0,081  |  ns/op  
   reflective    | avgt | 25 | 7442,266 |± 59,568 |  ns/op  

Ja. (volgens mijn [jmh](https://github.com/openjdk/jmh) benchmark)

Net geen 100x zo traag! Terwijl `Array.get` _native_ is. Deze [openjdk bug](https://bugs.openjdk.org/browse/JDK-8051447) bevestigt de meting: 100x trager. En het lijkt erop dat het niemand interesseert. De bug staat al bijna tien jaar open! 

De reden voor de slechte performance is enerzijds de JNI overhead, maar volgens de bugmelding is de C-code ook heel slecht.

We kunnen drie dingen doen:

1. niks. de slechte performance accepteren. 
2. code maken die de array inspecteert met heel veel `instanceof`. (_goed te doen, maar lelijke code_, zie de bugmelding)
3. dynamische bytecodegeneratie voor elke soort array met ASM (_keimoeilijk_)

Genoeg voor nu. Volgende keer verder met Collections en Maps. En we gaan de dynamische dieptes in met runtime inspectie van javabeans en records.