---
layout: post
title: "Lekker spelen met webassembly"
author: sander.hautvast
categories: [rust-lang]
image: assets/images/wasm/torbjorn-helgesen-SKlrApjIBrw-unsplash.jpg
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Het is me eindelijk gelukt iets 'zinvols' met webassembly te doen. Dat is op zich best lastig. De belofte dat het javascript zou gaan vervangen lijkt mij in ieder geval niet haalbaar. Ik heb eens geprobeerd met [https://yew.rs/](https://yew.rs/) iets te maken. Hierbij wordt rust vertaald naar wasm. Het programmeermodel van yew is afgeleid van react, met een virtual DOM en zo, dus het zou mogelijk moeten zijn om er iets niet-triviaals mee te maken, maar _rust_ is zo vervelender dan javascript en eenvoudige acties worden zo lastig dat het gewoon niet meer leuk is:

```javascript
const canvas2dContext = document.getElementById("canvas").getContext("2d");
```

dezelfde actie met rust:

```rust
if let Some(canvas) = document().get_element_by_id("canvas").and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()){
    let ctx: CanvasRenderingContext2d = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()
        .unwrap();
}
```

Ik geloof niet dat dit mainstream gaat worden...

Meer kans maakt [leptos](https://leptos.dev/). Dit is een 'modern' web framework in de zin dat het geen virtual DOM bevat en dat er _signals_ in zitten. Precies de bewegingen die je overal in de webwereld ziet gebeuren. Signals kun je zien als een event dat zorgt dat de view geupdate wordt. Vroeger (vorige week nog) deed je dat met _property/data binding_ (angular, react etc). Deze frameworks bepalen steeds opnieuw de verschillen in de laatste versie van de virtual DOM met de vorige en doen zo nodig een update van de echte. 

Signals werken direct. Je registreert er één en het framework weet waar hij gebruikt wordt (wordt build time bepaald). Elke update van een waarde zorgt voor een update van de view. Klinkt als een goed idee.

Leptos bevat behoorlijke tutorials en een fraaie set standaard componenten. Ze hebben, denk ik tijd besteed aan het verbergen van de lelijke code van hierboven. Goed werk. Toch vraag ik me af of de webwereld zit te wachten op `rust`.

Natuurlijk kun je ook andere talen compileren naar webassembly. Maar mijn indruk is dat het hobby werk blijft. Er is tot nog toe geen FAANG (facebook, apple etc) op gesprongen om er een hype van te maken, en een volwassen framework. De JS wereld is ook te groot om er snel een deuk in te kunnen slaan.

En waarom zou je het doen? De doorsnee web apps in javascript zijn snel genoeg. Als er performance problemen zijn, liggen die meestal op de server.

De echte kracht van webassembly ligt veel meer in taken die je normaal gesproken _buiten_ de browser doet. Zoals een [database](/database-browser). Afgelopen winter deed ik een security training en daarbij was er een leeromgeving met intellij, maar dan dus ín de browser. Grote voordeel is: geen installatie. Iedereen krijgt exact hetzelfde voorgeschoteld. Een soort Citrix.

>Wat is WebAssembly eigenlijk?

Lange tijd voelde het allemaal een beetje magisch. Dat heb ik wel vaker als ik het niet helemaal snap. Maar webassembly is uiteindelijk een hoop bytes, die zoals java bytecode door een runtime engine worden uitgevoerd. Dat kan er zo simpel uitzien als hieronder:

```html
<script>
    const data = [0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, 0x01, 0x06, 0x01, 0x60, 0x01, 0x7f, 0x01, 0x7f,
        0x03, 0x02, 0x01, 0x00, 0x07, 0x05, 0x01, 0x01, 0x66, 0x00, 0x00, 0x0a, 0x0d, 0x01, 0x0b, 0x01,
        0x7f, 0x7f, 0x20, 0x00, 0x41, 0xef, 0x00, 0x6c, 0x0f, 0x0b];
    WebAssembly.instantiate(Uint8Array.from(data)).then(wasm => console.log(wasm.instance.exports.f(9)));
</script>
```
* `data` bevat de gecompileerde bytes van een eenvoudige functie
* `wasm` is de handle naar de webassembly runtime
* `f` is de naam van de geëxporteerde functie

Een groot verschil tussen de JRE en de WASM runtime is dat de laatste geen garbage collection heeft. Het geheugen is gewoon een lange rij bytes. Precies dit maakt het mogelijk dat je alles, dat wil zeggen `C`, `Go`, `Rust`, `Python`, `Java`, `...` kunt compileren naar wasm bytes. Er is ook een headless runtime en die heet [https://wasmer.io/](https://wasmer.io/). Dat is dus in potentie de universele runtime. Is dat dan even snel als native? Niet tenzij je een JIT compiler gaat gebruiken. Voor zover ik weet gebeurt dat (nog) niet. 


##### _Terug naar mijn projectje:_

In zekere zin is dit het product van jaren. Het begon ergens in 2016, geloof ik, in java met behulp van [http://www.jhlabs.com/ip/filters/](http://www.jhlabs.com/ip/filters/). 

Het idee was om een flood-fill algoritme te maken voor alle 'vlakken' in een afbeelding

![pic](/assets/images/wasm/suaad_maasi-cutout.png)

Later heb ik de java vervangen door rust. De snelheid werd wel groter, maar het was geen wereld van verschil. Ik kreeg het idee om niet een statische kleur te gebruiken voor het opvullen, maar stukjes uit andere afbeeldingen. Ik ontdekte een boek, [Klaer Lightende Spiegel der Verfkonst](https://www.thisiscolossal.com/2014/05/color-book/). Een handgeschilderd zeventiendeëeuws werk met kleurenstalen, precies zoals je nu [color palettes](https://colors.muz.li/) hebt met bij elkaar passende kleuren.

Je krijgt dan dit: ![pic2](/assets/images/wasm/unsplash.png)

Ik heb ooit een poging gedaan dit in een webassembly applicatie om te bouwen met `yew`, maar dat liep hopeloos vast. Na mijn ervaring, dit jaar, met intellij in de browser kreeg ik de ingeving om de image processing in wasm te doen, maar de user interface met (vanilla) javascript. Dat werkte beter.

Het leuke is dat allerlei rust crates (libraries) zonder meer bruikbaar zijn in een web context. [image](https://crates.io/crates/image) is een veel gebruikte rust libary voor image processing. De eerste fase is de reductie van _noise_ in de afbeelding. De kleurvlakken worden daardoor groter en daarmee het artistieke effect. De _median_ filter van image doet precies dat. [photon](https://silvia-odwyer.github.io/photon/guide/using-photon-web/) doet de vertaling van html `img` naar rust formaat en terug naar een `canvas` element.

De applicatie draait volledig client side, wat deployment heel makkelijk maakt en dus kan ik hem gewoon op github pages hosten:
[https://shautvast.github.io/spiegel-demo/](https://shautvast.github.io/spiegel-demo/)

De sourcecode:
[https://github.com/shautvast/spiegel-web](https://github.com/shautvast/spiegel-web)

_De performance is niet bijzonder goed..._

Native, zeker in release-mode is het aanmerkelijk sneller. Dat maakt dat ik denk dat je alle claims ten aanzien van webassembly, _'near native speed'_ met een korrel zout moet nemen. Ik heb nog wel een idee om het algoritme te versnellen. Maar het valt me een klein beetje tegen...

En dat is de reden dat `photoshop` nog niet binnen je browser draait. En dat is ook de reden dat webassembly IMHO niet de wereld gaat veroveren, behalve voor zeer specifieke toepassingen. Maar je weet het nooit!
<div style="text-align: right">∞</div>