---
layout: post
title:  "Een willekeurige blik onder de motorkap"
author: sander.hautvast
categories: [testautomatisering]
image: assets/images/motorkap/nathan-dumlao-ycXpgCeERQc-unsplash.jpg
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Het genereren van (pseudo) willekeurige waardes kán heel eenvoudig zijn.

_disclaimer: ik ben geen expert op dit gebied._
Deze post begon gewoon vanuit nieuwsgierigheid. En lezend en googelend ontdekte ik een paar feiten die ik nog niet kende. 

De javadoc van `java.util.Random` bevat een verwijzing naar Knuth's, _The Art of Computer Programming, Volume 2, Third edition: Seminumerical Algorithms, Section 3.2.1._ Als je daar gaat kijken staat er deze formule:

$$ X_{n+1} = (a X_n + c) \mod m $$

waarin
* $$ X_{n+1} $$ : de volgende waarde
* $$ a $$ : de multiplier
* $$ X_n $$ : de huidige waarde (begint met de _seed_)
* $$ c $$ : de increment
* $$ m $$ : de modulus

Het algoritme is dus een simpele formule om een reeks getallen te genereren waarbij een waarde afhankelijk is van de waarde ervóór en een paar vaste parameters.

Je begint met een eerste waarde, de _seed_. Dat is ofwel een zelf gekozen getal ofwel een waarde die de jdk genereert met behulp van `System.nanoTime()`. Zolang je zelf een waarde kiest voor de seed en altijd dezelfde, genereer je altijd dezelfde reeks _pseudo_ random getallen.

> Er zijn, behalve de systeemtijd ook andere bronnen van willekerigheid, zoals muisbewegingen, netwerkverkeer etc.

De andere waardes, multiplier, increment en modulus zijn niet helemaal willekeurig en kun je hard coderen.

Als je in de code gaat kijken, zie je dat de javadoc niet helemaal klopt en dat de implementatie (voor het standaard algoritme), geen modulo gebruikt, maar een _bitmask_, dat wil zeggen een _bitwise-and_ met een waarde die binair een reeks enen is.

De reden daarvoor is dat modulo veel duurder is dan een _and_ operatie. En de feitelijke uitwerking is vergelijkbaar: hak het minst willekeurige deel van het getal af. Bij java is de mask zodanig dat de hoogste 48 bits overblijven. De mask is namelijk `(1 << 48) - 1`, oftewel 48 enen in binair.
 
De onderstaande _kotlin_ code laat zien dat de werking van java Random heel makkelijk na te maken is.
```java
import java.util.*

var seed = 1398255702L
val mask = (1L shl 48) - 1
val multiplier = 0x5DEECE66DL
val increment = 0xBL

fun main(){
    val javaRandom = Random(seed)

    seed = seed xor multiplier and mask // "initial scramble"

    for (i in 0..10) {
        val nextJavaInt = javaRandom.nextInt()
        val myNextInt = nextInt()

        val paddedNextJavaInt = nextJavaInt.toString().padStart(11)
        val paddedNextInt = myNextInt.toString().padStart(11)

        println("java: $paddedNextJavaInt \t mine:  $paddedNextInt")
        assert(nextJavaInt == myNextInt)
    }
}

fun nextInt(): Int {
    seed = (seed * multiplier + increment) and mask
    return (seed shr 16).toInt()
}
```
##### Run
```
java:  -866824032 	 mine:   -866824032
java:  -112010878 	 mine:   -112010878
java:   907797781 	 mine:    907797781
...etc
```

Het algoritme zelf staat op regel 26 en 27. 
De seed is willekeurig gekozen. In de java code wordt er eerst een "initial scramble" gedaan, zie regel 11. De reden daarvoor is me eerlijk gezegd onduidelijk. Ook de keuze voor de magic numbers voor increment en multiplier weet ik niet. Knuth geeft er wel specifieke regels voor, maar die gelden voor de modulo implementatie. Niet elke combinatie van parameters is even goed voor de entropie.  De seed zelf moet ook zoveel mogelijk bits bevatten, anders is de reeks makkelijk te raden (bijvoorbeeld als je maar 8 bits hebt en dus 256 mogelijke waardes in plaats van $$ 2^{63}-1 $$ voor long)

##### Wat is willekeurigheid?

1. Onvoorspelbaarheid / entropie: bijvoorbeeld een dobbelsteen
2. Uniformiteit: de dobbelsteen is niet verzwaard, zodat elk cijfer even grote kans krijgt.

Klinkt aannemelijk toch?
Maar is dit een goede manier om een dobbelsteen te simuleren?

```java
double d = Math.random() * 6 + 1;
```

Volgens mij heeft iedereen wel eens zulke code gebruikt. Maar deze dobbelsteen is verzwaard! Waarom? Omdat de uitkomst van Math.random() $$ 2^{48} $$ mogelijke uitkomsten heeft, en het is onmogelijk om die eerlijk over 6 mogelijkheden te verdelen, want $$ 2^{48} $$ is niet deelbaar door 6! Zie deze uitgebreide [post](https://stackoverflow.com/questions/70717929/how-to-get-sufficient-entropy-for-shuffling-cards-in-java) van rzwitserloot (van Lombok) op stackoverflow.

`Math.random` gebruikt ook `java.util.Random`. Maar gebruik hem zelf niet en neem voor je dobbelsteen `Random.nextInt`! Van een double een int maken is in principe ook trager, want Math.random maakt eerst een double van een int (twee om precies te zijn).

##### Wanneer is java.util.Random niet meer voldoende? 
1. simulaties 
2. cryptografie, genereren van keys

Vanaf java 17 bevat de jdk een nieuwe API (`RandomGenerator`) met veel meer algoritmes met meer bits dan 48 (tot 1152). Als je maar vaak genoeg een pseudo random waarde genereert, gaat er periodiciteit optreden. Stel dat je elke nanoseconde java.util.Random.nextInt aanroept, dan heb je na een uur of 78 alle mogelijke waardes van $$ 2^{48} $$ gehad en zou je weer opnieuw beginnen. Met 1152 bits stel je dat uit tot na het einde van het universum. 

Bijvoorbeeld:
```java
RandomGenerator g = RandomGenerator.of("L64X128MixRandom");
```
Wel jammer dat dit niet samenwerkt met Collections.shuffle(List, Random). Deze methode voor het 'schudden' van een lijst (speelkaarten) heeft een instantie van Random nodig en RandomGenerator is dat niet. Er is wel een [voorstel](https://bugs.openjdk.org/browse/JDK-8294694) om dat aan te passen.
De standaard Random is vaak wel voldoende, en uniform. Dus je kunt je kaarten er rustig mee schudden en anders is er nog `SecureRandom`, speciaal voor cryptografische toepassingen.

Zowel Random als RandomGenerator hebben een aantal bijzondere methodes, zoals `ints` en `doubles`, die een _oneindige_ Stream geven. Lijkt ineens op een python generator.
Als als je maar 6 random doubles nodig hebt:
```java
g.doubles(6).forEach(System.out::print);
```

##### threads
Standaard Random heeft mutable state in de vorm van het laatst uitgegeven nummer, dat input is voor de volgende. Gelukkig is er daarvoor een `AtomicLong` gebruikt, die
1. _atomic_ wordt gemuteerd (voor standaard long is dat niet gegarandeerd)
2. zorgt voor de nodige memory barriers voor het bijwerken van alle cpu-caches
3. race condities voorkomt

Je kunt hem dus rustig in al je threads gebruiken, met maar één caveat: je verliest de herhaalbaarheid van de willekeurige reeks. Als je dat nodig hebt, moet je aan elke thread een eigen Random toekennen. Speciaal voor dit geval is er de ThreadLocalRandom, die dat automatisch doet en voor meer entropie heb je ook nog de SplittableRandom. Die laatste is met name bedoeld voor parallel streams.


<div style="text-align: right">∞</div>