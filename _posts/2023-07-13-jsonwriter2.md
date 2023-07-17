---
layout: post
title:  "Scherp je geest, met JSON"
author: sander.hautvast
categories: [json, ASM]
image: assets/images/neom-Oj8w6hWC0dU-unsplash.jpg
beforetoc: ""
featured: false
hidden: true
lang: nl
---
**Deel 2 Dynamische code**

_Werkwijze_: Plan een tweede dag vrij, of pak die dag dat je nog net niet helemaal hersteld bent van een griep. Laat je niet afleiden door Insta of youtube shorts.

Het toevoegen van `List`, `Set` en `Map` aan de code, zou niet al teveel moeite moeten kosten. Eén ding. De vorige keer deden we de inspectie met `getClass`. Dat is nu niet handig. `instanceof Collection` is slimmer, omdat je met één enkele check alle mogelijke implementatie classes afdekt (ArrayList, LinkedList, HashSet etc.) 

Voor Lists en Sets is de code qua structuur gelijk aan die voor arrays. Ik zou weer kiezen voor recursie vanwege de mogelijkheid van lijsten van lijsten enzovoorts. Een Map, `object` in JSON terminologie is net anders, maar niet heel veel.

> It's turtles all the way down

Een kleine gotcha met met Maps is dat de key (_name_) in JSON altijd een string is. Zet er dus (dubbele) quotes omheen, ook al is het in java bijvoorbeeld een Integer.

*Mooie opwarmer voor het echte werk...*

We komen nu toe aan javabeans, of hun moderne variant, records. Eventueel stel je dit nog uit met enums, maar uiteindelijk moet het toch gebeuren. We moeten een beslissing nemen. De uitdaging is dat we een JSON object moeten maken door runtime het java type te inspecteren. Een class is een aantal mappings van type naar waarde (afgezien van methods). 
Grofweg zijn er de volgende manieren om een class te inspecteren
1. runtime reflectie met `java.lang.reflect` 
2. runtime reflectie met `java.lang.invokeMethodHandle`
3. runtime reflectie met een bytecode engineering library
4. buildtime reflectie zoals Lombok


(1). makkelijkste optie, geen uitdaging, slechte performance

(2) is niet veel moeilijker dan (1). Je gebruikt reflectie om de getters te vinden en maakt daar een MethodHandle voor. Performt beter, want geen runtime checks.

(4) heb ik nog niet geprobeerd. Het moet mogelijk zijn. Je moet dan wel altijd een maven/gradle plugin gebruiken. De voordelen zijn minimale overhead en zonder meer bruikbaar in GraalVM. Eventueel gebruik je nu wel reflectie, want dat wordt nu alleen buildtime gebruikt.

Bij 2, 3 en 4 maak je per te serialiseren type een Writer en slaat die op in een cache, zodat je hem maar één keer hoeft te maken. 

Er zijn diverse bytecode libraries. In het gebruik is [javassist](https://www.javassist.org/) het eenvoudigst, omdat het een javacompiler bevat. Je genereert dus javacode die je laat compileren. Het belangrijkste nadeel is een ernstige warning van de JVM in jdk9 of hoger en de vraag of op termijn de code nog wel kan draaien. Andere libraries hebben hier vreemd genoeg geen last van. Dat zijn met name [bytebuddy](https://bytebuddy.net/#/) en [ASM](https://asm.ow2.io/). 

Bytebuddy heb ik geprobeerd en ik vond de _fluent_ API bijzonder onintuïtief. ASM is aan de ene kant lastig, omdat je echt op bytecode level programmeert, maar aan de andere kant hebben we een superhandig hulpmiddel in de vorm van `javap`.

Voorbeeld:

```java
protected void json(StringBuilder b, Object o) {
        Bean1 value = (Bean1)o;
        b.append("{");
        b.append("data1");
        b.append(":");
        Mapper.json(b, value.getData1());
        b.append("}");
    }
```

*javap output* voor bovenstaande methode:

```bytecode
 0: aload_2
 1: checkcast     #7     // class nl/sanderhautvast/json/ser/nested/Bean1
 4: astore_3
 5: aload_1
 6: ldc           #9     // String {
 8: invokevirtual #11    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang StringBuilder;
11: pop
12: aload_1
13: ldc           #17    // String data1
15: invokevirtual #11    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
18: pop
19: aload_1
20: ldc           #19    // String :
22: invokevirtual #11    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
25: pop
26: aload_1
27: aload_3
28: invokevirtual #21    // Method nl/sanderhautvast/json/ser/nested/Bean1.getData1:()Ljava/util/UUID;
31: invokestatic  #25    // Method nl/sanderhautvast/json/ser/Mapper.json:(Ljava/lang/StringBuilder;Ljava/lang/Object;)V
34: aload_1
35: ldc           #31    // String }
37: invokevirtual #11    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
40: pop
41: return
```

Het voert wat ver om alles uit te leggen, maar:
* `aload_2.` Alle lokale variabelen, te beginnen met `this`, vervolgens de methode argumenten en de rest hebben geen naam in bytecode, alleen een index. Index *2* slaat op de parameter *Object o*
* `astore_3` pakt een waarde van de stack en slaat deze op in de nieuwe lokale variabele met index 3.
* `ldc` laadt een constante op de stack
* `invokevirtual` voor reguliere, niet static method invocations. De signature van de method is een argument voor de operation. Feitelijk staat hier een referentie naar de `constantpool` van de class. ASM abstraheert dat gelukkig weg. Deze operation popt twee items van de stack.
* `pop`. De return parameter van append wordt niet gebruikt. Deze operation gooit hem voor je van de stack.

En tot slot hieronder een klein voorbeeldje van hoe je dat weer toepast binnen ASM.

```java
add(new VarInsnNode(ALOAD, 1));
add(new LdcInsnNode("{"));
add(new MethodInsnNode(INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;"));
add(new InsnNode(POP))
```

Je ziet dat deze code 1-op-1 overeenkomt met de bytecode van javap. Het is wel goed je verdiepen in de weergave van bijvoorbeeld de method descriptor. Kijk bijvoorbeeld eens in de [java spec](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3.2). 

De [user manual](https://asm.ow2.io/asm4-guide.pdf) van ASM bevat ook de informatie over alle benodigde boilerplate code. 

Het debuggen van fouten is lastiger omdat je niet een referentie krijgt naar waar het precies fout gaat in de gegenereerde code. Handig is om die tijdens het bouwen weg te schrijven naar een bestand en die dan weer proberen te lezen met javap.

Tot slot. Een record is vrijwel hetzelfde als een gewone class. Het enige waar je rekening mee moet houden is dat in tegenstelling tot beans, de getters en setters niet beginnen met _get_ dan wel _set_. De naam van de methode is gelijk aan die van de property.

Happy programming!