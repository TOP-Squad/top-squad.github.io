---
layout: post
title: "First Contact"
author: sander.hautvast
categories: [java, jdk-22]
image: assets/images/javaffi/arrival.png
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Nog meer nieuwe java (22) features, maar nu één die je _waarschijnlijk_ niet gaat gebruiken. Toch leuk om over te schrijven want ik kon nergens voorbeelden vinden, voor wat ik wilde en ik had het eigenlijk al opgegeven, maar toen lukte het tóch, met een klein beetje hulp van ... JetBrains AI.

![introduce yourself](/assets/images/javaffi/who-are-you.png)

![introduce yourself](/assets/images/javaffi/jetbrains-is-powered-by-chatgpt.png)

Waar gaan we het over hebben? JEP-454: Foreign Function and Memory API. De nieuwe oplossing voor de oude JNI, die niemand leuk vond, 
waarschijnlijk vanwege `UnsatisfiedLinkException` 🤬

De java FFI API is bedoeld om het _contact_ tussen de heap in de JVM en de native wereld en off-heap memory te standaardiseren en eenvoudiger 
te maken. Geen `sun.misc.Unsafe` meer en geen native `ByteBuffer`s. De API zat trouwens al in eerdere releases van de JDK, maar is aangepast 
in JDK-22. Het is nog steeds een _preview_ feature en het kan dus nog allemaal anders worden.

Er zijn (weinig) voorbeelden te vinden, zoals [hier](https://www.infoq.com/news/2023/10/foreign-function-and-memory-api/), 
of [hier](https://bazlur.ca/2023/10/16/accessing-native-c-functions-from-java-using-openjdks-jep-454-foreign-function-memory-api/) 
voor hoe het nu moet, maar:
1. ze gebruiken standaard (stdlib) functies (in plaats van eigen (_rust_) code)
2. ze gebruiken alleen primitieve types (ints bijvoorbeeld).

Ik wilde het iets ambitieuzer aanpakken:
1. simpele rust functie maken die een String teruggeeft
2. deze aanroepen in java en het resultaat tonen

>Een HelloWorld dus.

Toch was het nog een lange weg. 

Het maken van de functie in rust is simpel genoeg, maar om hem zo te maken dat het ook samenwerkt met de java wereld,
leek teveel voor me gevraagd. Het projectje lag alweer twee weken stof te vangen op de harde schijf, toen ik spontaan besloot een JetBrains AI abonnement
te nemen. 

En dan ook maar eens kijken waar díe mee komt: 

![introduce yourself](/assets/images/javaffi/rust-functie.png)

>Fascinerend....

Ik had `21` getypt in plaats van `22`, maar AI zelf loopt ook achter:

![introduce yourself](/assets/images/javaffi/latest-release.png)

Really?

>Dan lopen we nog helemaal niet achter in mijn opdracht.

Ok. Dan maar eens naar de code kijken. Dat ziet er beter uit:

`#[no_mangle]`
Klopt dat had ik ook, omdat zonder dat attribute de functie in de binaire code de naam van de hele signature krijgt (dus inclusief de return parameter).

`extern "C"`
Nodig voor de FFI vanuit rust.

`-> *mut c_char`
Aha, ik had een rust CString gebruikt. Misschien dat een raw mutable pointer beter werkt. Maar waarom mutable?

Helaas werkt de code niet. Het resultaat van CString::new heeft nog een `unwrap()` nodig.

`std::mem::forget(s)`
Zorgt ervoor dat rust vergeet de string op te ruimen. Heel goed, A-Ietje!

Dus deze functie geeft een ouderwetse C pointer terug naar een String. Zou zo moeten werken, vind ik, maar ik kreeg het niet aan de praat..

```java
public static void main(String[] args) throws Throwable {
    try (var arena = Arena.ofConfined()) {
        var linker = Linker.nativeLinker();
        var rustlib = SymbolLookup.libraryLookup("/Users/Shautvast/dev/javaffi_rust/target/debug/libjavaffi_rust.dylib", arena);

        var helloWorld = rustlib.find("get_hello_world").orElseThrow();
        var methodHandle = linker.downcallHandle(helloWorld, FunctionDescriptor.of(
                ValueLayout.ADDRESS
        ));
        MemorySegment output = (MemorySegment)methodHandle.invoke();
        System.out.println(output.getString(0));
    }
}
```

Geeft:
```
Exception in thread "main" java.lang.IndexOutOfBoundsException: Out of bound access on segment MemorySegment{ address: 0x6000039f61d0, byteSize: 0 }; new offset = 0; new length = 1
	at java.base/jdk.internal.foreign.AbstractMemorySegmentImpl.outOfBoundException(AbstractMemorySegmentImpl.java:434)
	...
	at Main.main(Main.java:36)
```

>Ok, maar wat is dit voor code?

**Memory Arena's** zijn ofwel een interessante nieuwe hype in `zig` en `rust`, ofwel iets wat we voor de verandering in java al lang hadden in de vorm van _generational_
Garbage Collection. Het zijn namelijk gebieden in het geheugen die in één keer ge(de)allokeerd worden, in plaats van per object. In dit geval is het 
het stukje geheugen dat we samen met de rust-code willen gebruiken.

**Linker** het superhandige dingetje dat toegang geeft tot de native functies, samen met `libraryLookup(...)`
De dylib is de library die `Cargo` net gebouwd had.
Al met al veel minder gedoe dus in vergelijking met JNI.

```rust
var methodHandle = linker.downcallHandle(helloWorld, FunctionDescriptor.of(
                ValueLayout.ADDRESS
        ));
```
Hier beschrijven we de functie signature. Er staat: een functie zonder parameters, met 1 return parameter: een ADDRESS (oftewel pointer). 
Dit komt overeen met de _get_hello_world_ functie hierboven.

**Wacht eens even...**

Het zou mogelijk moeten zijn een _return argument_ te gebruiken, en die te vullen:

```rust
use std::ffi::{c_char, CString};
use std::ptr::copy;

#[no_mangle]
pub extern "C" fn hello_world(raw_string: *mut c_char) {
    unsafe {
        let string = "Hello, world! 🤓";
        let new_raw_string = CString::new(string).unwrap().into_raw();
        
        raw_string.copy_from(new_raw_string, string.len());
        // dit doet het zelfde
        copy(new_raw_string, raw_string, string.len());
    }
}
```

Een iets andere signature. In plaats van een waarde terug te geven, muteren we wat er binnenkomt. Heel old-school en bovendien `unsafe`!

```java
void main() throws Throwable {
    try (var arena = Arena.ofConfined()) {
        var linker = Linker.nativeLinker();
        var rustlib = SymbolLookup.libraryLookup("/Users/Shautvast/dev/javaffi_rust/target/debug/libjavaffi_rust.dylib", arena);

        var helloWorld = rustlib.find("hello_world").orElseThrow();
        MemorySegment buf = arena.allocateFrom("..................", StandardCharsets.UTF_8);
        var methodHandle = linker.downcallHandle(helloWorld, FunctionDescriptor.ofVoid(
                ValueLayout.ADDRESS
        ));

        methodHandle.invoke(buf);

        System.out.println(buf.getString(0));


    }
}
```

![hoera](/assets/images/javaffi/hoera.png)

Kleven wel nadelen aan. Het geallokeerde geheugen moet groot genoeg zijn en beide kanten van de muur moeten precies weten hoeveel dat is, anders 
gebeuren er nare dingen. 

En natuurlijk zijn strings ook nog maar primitief. De java API is geavanceerd genoeg om complexe types ('classes'/ structs) te definieren als 
memory layout. Zover ben ik nu niet gegaan. Maar je zou in theorie ook JSON met elkaar kunnen uitwisselen met deze techniek, of een binair formaat. 
Dit vergemakkelijkt de uitwisseling. Het wordt, ondanks de performance overhead, [daadwerkelijk](https://news.ycombinator.com/item?id=40706467) gebruikt.

<div style="text-align: right">∞</div>