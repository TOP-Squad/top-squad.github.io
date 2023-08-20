---
layout: post
title:  "Multidimensionele arrays bestaan niet!"
author: sander.hautvast
categories: [java, performance]
image: assets/images/arrays/nik-fDaUCTp28dA-unsplash.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
Ik had niet gedacht een artikel te zullen schrijven over arrays in java. Ze zijn bijzonder saai, en wanneer heb je ze eigenlijk nodig? 

De enige (actieve) herinnering aan arrays in een _echte_ codebase die ik heb, ging over een situatie dat je eigenlijk multiple return parameters zou willen hebben (zoals in python), of (door de taal ondersteunde) tuples (zoals in rust). Als ik nu in een review een methode met `Object[]` als return parameter zou zien, zou dat tot kamervragen leiden.
>Niet heel erg opzienbarend

In een [vorige blog](/jsonwriter/) had ik al geschreven over de slechte performance van _java.lang.reflect.Array_. In een poging daar een beter alternatief voor te maken stuitte ik op een paar onverwachte eigenaardigheden.

Arrays zijn sinds java 1.0 niet veranderd. Dat is logisch gezien het feit dat ze een standaard onderdeel zijn van de meeste talen en het feit dat java altijd backwards compatible is gebleven. En, tja, java uit die tijd, is, soms _apart_ of beter gezegd _C-achtig_.

__3 manieren om een multidimensionele arrays te instantieren__

1. `int[][] array = new int[3][2];`     // ok..
2. `int[] array[] = new int[2][2];`     // doet denken aan C pointer notaties
3. `int[][] array = new int[5][];`      // WTF?

>Wat _betekent_ optie #3?

En hoe kan dit werken?
```java
String[][][] array = new String[1][1][1];
array[0] = new String[2][1];
```

Ik kon het antwoord niet vinden na een paar keer googelen, maar ik stuitte wel (weer) op [Jol](https://github.com/openjdk/jol). Nadat ik de jol-cli had geinstalleerd (werkt het best op linux), zag ik het volgende:
![jol](/assets/images/arrays/screenshot-jol.png)

[Jol](https://github.com/openjdk/jol) is een tool om de internals van java objecten te bestuderen. De manier om dit te doen voor een 2-dimensionele String array te doen is:
```bash
java -jar jol-cli.jar internals "[[Ljava.lang.String;"
```

Wat een rare schrijfwijze! `[[L` en `;` zijn onderdeel van de interne weergave van java objecten, maar dan met `'/'` in plaats van `'.'`. 
Niet toevallig is dit wel precies de manier om `Class.forName` te gebruiken voor arrays (want dat is wat jol doet). Ik wist het niet, dus ik neem aan jullie ook niet.

>_En wat is de conclusie?_

In een multidimensionele array is er maar één array-lengte en dat is de 'buitenste' 
![jol](/assets/images/arrays/screenshot-jol-length.png)

Dus `String[1][1]` is eigenlijk `String[1][]`. Elke element van de buitenste array is een 1-dimensionele array, van willekeurige lengte! Weg zijn je runtime bounds checks (C-achtig dus)! En dus kan bovenstaande code, die een array van lengte 2 toekent aan het eerste element van `array[1][1][1]` gewoon werken.

>There are no true multi-dimensional arrays in Java, just arrays of arrays. This is why int[][] is a subclass of Object[]. If you need a large multi-dimensional int[] in Java, it is a bit more efficient to allocate a large int[] and calculate the offset yourself. However, make sure to, if possible, navigate the int[] in such a way that 64 bytes at a time can be read. That is a lot more efficient than jumping around. **_[Heinz Kabutz](https://javaspecialists.teachable.com/courses/249332/lectures/3886639)_**

De achtergrond voor dat laatste is caching en _prefetching_. Gewoon RAM geheugen is traaag. Het is dankzij de L1/2/3 caches dat processoren nog een beetje presteren, als ze iets uit geheugen willen lezen. Dat gaat het best, als je in batches gelijk meer data op kan halen. De volgende read kun je het dan uit de cache halen. Dit voordeel wordt extra groot door prefetching, als de cpu probeert te voorspellen wat je _next ~~move~~ read_ zal zijn. Dit zorgt dat het uitmaakt, hoe je door een n-dimensionele array loopt (n>1).

>__Hoe groot is eigenlijk het voordeel van CPU caching en prefetching?__

Over deze vraag ben ik gestruikeld en in een konijnenhol gevallen van microbenchmarking. Uiteindelijk heb ik de metingen op mijn apple M2 allemaal opnieuw gedaan op een standaard Amazon linux AMI, vooral omdat die beter te generaliseren zijn.

Je vriend, de JIT compiler wordt je vijand als je gaat benchmarken. In eerste instantie had ik een performance verbetering van 1342%, maar dat had er waarschijnlijk mee te maken dat er code weggeoptimaliseerd werd. 

Dit is de uiteindelijke code:
```java
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public long classicArrayGetTDLR() {
    long t = 0;
    for (int r = 0; r < ROWS; r++) {
        for (int c = 0; c < COLS; c++) {
            t += intArray[r][c];
        }
    }
    return t;
}
```
TDLR staat voor Top Down (buitenste loop), Left Right.

   Benchmark    | Mode  | Cnt  |  Score    |   Error    |  Units
----------------|-------|-----:|----------:|-----------:|---------
classic2DArrayGetLRTD | avgt |   5 | 4184284.298 |± 7651435.011 | ns/op
classic2DArrayGetTDLR | avgt |   5 |  389369.258 |±    4064.665 | ns/op

_Amazon Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz_

Wow, 10x zo snel! Dat is precies wat [simondev](https://www.youtube.com/watch?v=247cXLkYt2M) zegt. En wat ik niet kon reproduceren op mijn M2. Daar is het ongeveer een factor 2.


>**Caveat:**
Zolang je niet weet wat de performance verhouding is tussen de array lookup en de sommatie, weet je ook niet wat de percentuele performance verbetering is van prefetching tijdens het itereren.

Een ander aspect van het citaat van Kabutz is de suggestie dat je met een 1-dimensionele array meerdimensionele kunt simuleren en wellicht met betere performance. Een eerste implementatie die ik had gemaakt voor arrays met elk mogelijk aantal dimensies was niet sneller als je alleen naar TDLR kijkt.

   Benchmark    | Mode  | Cnt  |  Score    |   Error    |  Units
----------------|-------|-----:|----------:|-----------:|---------
seqMultArrayGetLRTD | avgt |   5 | 1399817.940 | ±  271516.298 | ns/op
seqMultArrayGetTDLR | avgt |   5 |  392543.679 | ±    3671.543 | ns/op

>

>Ok, en een specialistische voor 2 dimensies?

Zoals dit?
```java
public int get(int row, int col) {
    return row * this.cols + col;
}

public void set(int row, int col, int val) {
    data[row * this.cols + col] = val;
}
```

   Benchmark     | Mode  | Cnt  |  Score     |   Error    |  Units
-----------------|-------|-----:|-----------:|-----------:|---------
seq2DArrayGetLRTD   | avgt |   5 |  533659.920 |±   87889.355 | ns/op
seq2DArrayGetTDLR   | avgt |   5 |  400005.690 |±    7020.963 | ns/op

>

>Dit geeft dus alleen een verbetering voor de niet-efficiente manier van itereren: LRTD. 

**Conclusie**

Zelf je indexen berekenen heeft niet zoveel zin. (tenzij je op mijn M2 werkt, daar was het circa 25% sneller)

>**En _write_ acties dan?**

   Benchmark     | Mode  | Cnt  |    Score       |     Error     |  Units
-----------------|-------|-----:|---------------:|--------------:|---------
classic2DArraySetLRTD | avgt | 5 | 4212263.046 |± 267087.769 | ns/op
classic2DArraySetTDLR | avgt | 5 | 1032451.067 |±  35040.403 | ns/op
seq2DArraySetLRTD   | avgt | 5 | 2569007.766 |±  45255.561 | ns/op
seq2DArraySetTDLR   | avgt | 5 |  721699.703 |±  22605.344 | ns/op

>

3 tot 4x zo snel, afhankelijk dan de implementatie. En hier heb je wel voordeel van zelf je index berekenen in een 2 dimensionele array. Ik weet niet waarom. Iemand? 

##### _[compiler blackholes](https://shipilev.net/jvm/anatomy-quarks/27-compiler-blackholes/)_

Hoe voorkom je dat de JIT compiler je dode benchmark code wegcompileert?

1. neem een return parameter op (zie voorbeeld)
2. gebruik _compiler blackholes_ (vanaf java 17)

JMH zorgt dat de return parameters in een zwart gat verdwijnen. Dit voorkomt ongewenste optimalisatie, maar heeft zelf wel een eigen performance impact en als response tijden klein zijn, gaat die overheersen. _

_Compiler blackholes_ zorgen ervoor dat je benchmark code niet als dode code wordt opgeruimd, maar dat de aanroep naar de JMH blackhole wel weg-geoptimaliseerd wordt, zodat je een zo zuiver mogelijke meting krijgt.

`java -Djmh.blackhole.mode=COMPILER -jar benchmark.jar`

>**Saai??**

Om terug te keren naar het begin van dit artikel. Zijn arrays saai? Ze worden niet veel gebruikt, dat wil zeggen, niet door jou! Maar in de JVM zijn ze essentieel voor String, HashMap en ArrayList, oftewel de drie meest voorkomende classes in elke JVM instantie. 

Dus als je door een String, of een ArrayList zoekt, maakt het uit, hoe je dat precies doet. En het verschil is groot genoeg om je er bewust van te zijn. En het is trouwens ook precies de reden dat je _vrijwel_ nooit een `LinkedList` moet gebruiken, want dan krijg je _gegarandeerd nooit_ het voordeel van prefetching. Zie deze [video](https://www.youtube.com/watch?v=YQs6IC-vgmo&t=0s) van Bjarne Stroustrup over waarom het algoritmische voordeel van LinkedList achterhaald is. Een poll op LinkedIn liet zien dat 99% van de _java_-mensheid er nog in gelooft. 

>Zelf doen?

```bash
git clone https://github.com/shautvast/arraybench
mvn clean package
java -Djmh.blackhole.mode=COMPILER -jar target/benchmark.jar
```

<div style="text-align: right">∞</div>