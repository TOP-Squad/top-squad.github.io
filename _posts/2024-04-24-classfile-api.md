---
layout: post
title: "Class-File API en de gehallucineerde blogs"
author: sander.hautvast
categories: [java, jdk-22]
image: assets/images/classfile/writer.jpg
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Op een middag deze de week liep ik weer eens tegen de nieuwe classfile api (jdk22) aan. Vorig jaar had ik al wel de [presentatie van Brian Goetz](https://www.youtube.com/watch?v=pcg-E_qyMOI) hierover gezien. Hij is de _java language architect_ van Oracle. Zijn verhaal is boeiend omdat hij weergeeft wat de rol van intuïtie is als je iets nieuws aan het ontwikkelen bent. 

Goetz:
>"And I looked at the API and I'm like: 'I don't like it. I'm not happy.'
And so, people quite reasonably said, okay, what would make you happy?
Well I don't know. But I don't like it"

Hij kon het niet precies onder woorden brengen en hij wist ook nog niet wat hij wilde. Hij wist alleen dat hij niet tevreden was. Herkenbaar denk ik. Als het gaat om een rest API, of een stuk code. In het begin werkt het wel, maar ziet het er dan nog lelijk of verwarrend uit. 

En daarna vertelt hij hoe ze tot een betere, consistente API kwamen. 

Die is er nu _preview_ in java 22. 

Ik heb eerder [geschreven](/jsonwriter2) over het genereren van java code met ASM. Deze API is de vervanger daarvoor, dus het lag voor de hand hier ook eens naar te kijken.

Op zoek naar bruikbare blogs, liep ik tegen een paar voorbeelden aan, die naast juiste informatie, ook code geven, die gewoon niet werkt.

De eerste op [medium](https://medium.com/@benweidig/looking-at-java-22-class-file-api-a4cb241ff785) begint heel degelijk met een inleiding over het waarom van de API en wat code, die als ik het uitprobeer prima werkt.
Maar zodra we bij _Generating a Class File_ aankomen, staat daar dit:

```java
ClassBuilder classBuilder = ...;
classBuilder.withMethod("fooBar",
...                        
```

Ik geef maar even een heel klein stukje weer. Het suggereert dat de ClassBuilder het startpunt is van de API. Ik ging het uitproberen en ik ontdekte waarom de instantiatie van de ClassBuilder ontbreekt. En ik ontdekte ook dat de code 1-op-1 is overgenomen van de oorspronkelijke tekst van [JEP-457](https://openjdk.org/jeps/457). En sindsdien is de API dus een paar keer over de kop gegaan.

De `ClassBuilder` kun je namelijk niet zelf instantieren en het hoeft ook niet.

Ook [deze blog](https://jameshamilton.eu/programming/jep-457-hello-world) klopt niet. Hij komt meer in de richting, maar het compileert niet.

```java
Classfile.of().buildTo(Path.of("HelloWorld.class")
```

Het is `ClassFile` en niet `Classfile`. En de argumenten in de aanroep naar `buildTo` kloppen niet.

Kortom, afhankelijk van hoe je het bekijkt, deze bloggers waren te haastig met hun post, of te lui voor een update, of om zelf te kijken wat de code moet zijn. 

Dus hierbij wat code voor een _HelloWorld_, die ik heb aangepast zodat hij wél werkt. Op dit moment althans (want de api is nog in preview op dit moment in JDK-22).



```java
byte[] bytes = 
ClassFile.of()
  .build(helloWorld, clb -> clb.withFlags(ClassFile.ACC_PUBLIC)
  .withMethod("main", voidStringArray, ClassFile.ACC_PUBLIC + ClassFile.ACC_STATIC,
    mb -> mb.withCode(cb ->
      cb.getstatic(system, "out", printStream)
        .ldc("Hello World")
        .invokevirtual(printStream, "println", voidString)
        .return_())));
```
_(dit is een fragment, onderaan staat de hele class)_ 

Als je dit zo bekijkt, is het inderdaad precies wat er moet gebeuren. Geen ingewikkelde boilerplate, een prima API.

Het verschil met ASM is met name dat alle bytecode ops methodes zijn op de `CodeBuilder` (cb), in plaats van constanten die je als parameter meegeeft. En geen visitor pattern meer.

`return_` met underscore is wel lelijk, maar ja het is een java keyword, dus je hebt weinig keus. (misschien met backticks in kotlin 🤔)

##### use case?
Voor ons, applicatie developers is de Class-File API niet heel bruikbaar. Desalniettemin kun je er deels _java reflection_ mee vervangen. Dus als je toch reflection wil gebruiken, heb je hiermee een beter performend alternatief, dat (in tegenstelling tot ASM) altijd in sync is met de bytecode in de jvm. Ook is het nog steeds (zoals met ASM) mogelijk om bestaande classes te analyseren en te instrumenteren. Dus het maakt het leven makkelijker voor toolbouwers.

Nog een paar links:
* [javadoc](https://docs.oracle.com/en/java/javase/22/docs/api/java.base/java/lang/classfile/package-summary.html)
* [inside java](https://www.youtube.com/watch?v=bQ2Rwpyj_Ks)

happy coding!
<div style="text-align: right">∞</div>
---

_HelloWorldBuilder_
```java
import java.lang.classfile.ClassFile;
import java.lang.constant.ClassDesc;
import java.lang.constant.MethodTypeDesc;

import static java.lang.classfile.ClassFile.ACC_PUBLIC;
import static java.lang.classfile.ClassFile.ACC_STATIC;
import static java.lang.constant.ConstantDescs.*;

@SuppressWarnings("preview")
public class HelloWorldBuilder {
    public static void main(String[] args) throws Exception {
        var helloWorld = ClassDesc.of("HelloWorld");
        var voidStringArray = MethodTypeDesc.of(CD_void, CD_String.arrayType());
        var system = ClassDesc.of("java.lang.System");
        var printStream = ClassDesc.of("java.io.PrintStream");
        var voidString = MethodTypeDesc.of(CD_void, CD_String);

        byte[] bytes = ClassFile.of().build(helloWorld,
                clb -> clb.withFlags(ACC_PUBLIC)
                        .withMethod(INIT_NAME, MTD_void,
                                ACC_PUBLIC,
                                mb -> mb.withCode(
                                        cb -> cb.aload(0)
                                                .invokespecial(CD_Object,
                                                        INIT_NAME, MTD_void)
                                                .return_()))
                        .withMethod("main", voidStringArray, ACC_PUBLIC + ACC_STATIC,
                                mb -> mb.withCode(
                                        cb -> cb.getstatic(system, "out", printStream)
                                                .ldc("Hello World")
                                                .invokevirtual(printStream, "println", voidString)
                                                .return_())));

        var classloader = new ByteClassLoader();
        classloader.addClass("HelloWorld", bytes);
        var helloworld = classloader.loadClass("HelloWorld");
        var main = helloworld.getMethod("main", String[].class);
        main.invoke(null, new Object[]{new String[0]});

    }
}
```

_ByteClassLoader_ (Je kunt ook de bytes wegschrijven naar een class file en dan op het classpath zetten)
```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class ByteClassLoader extends ClassLoader {

    private final ConcurrentMap<String, Class<?>> classes = new ConcurrentHashMap<>();
    public final static ByteClassLoader INSTANCE = new ByteClassLoader();

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class<?> instance = classes.get(name);
        if (instance == null) {
            throw new ClassNotFoundException(name);
        }
        return instance;
    }

    public void addClass(String name, byte[] bytecode) {
        Class<?> classDef = defineClass(null, bytecode, 0, bytecode.length);
        classes.put(name, classDef);
    }
}
```