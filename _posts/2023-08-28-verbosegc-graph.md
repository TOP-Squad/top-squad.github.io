---
layout: post
title:  "Memoryleaks op de commandline"
author: sander.hautvast
categories: [java, performance]
image: assets/images/jop/jop.gif
beforetoc: ""
featured: true
hidden: false
lang: nl
---
Het vaststellen van memoryleaks kán heel eenvoudig zijn. Deze [video](https://youtu.be/JoQN4xoXY5Y) van Jack Shirazi laat dat heel mooi
zien. Je hebt er wel een tooltje voor nodig om de cijfers te visualiseren. Hij gebruikt er [GCViewer](https://github.com/chewiebug/GCViewer)
voor. Dit draait dan op je eigen desk/laptop. Het kan alleen soms vervelend zijn om de logs vanaf de serveromgeving over te zetten. Ook kun je
dan niet _tailen_, zodat je dan alleen achteraf naar de resultaten kunt kijken.

Gelukkig bestaat er ook een manier om op de commandline userinterfaces te maken. _Java_ is daar alleen in het algemeen niet heel geschikt voor.
Om die reden, en omdat het een leuke uitdaging was, wilde ik kijken of het in _Rust_ kan en dat bleek zo te zijn. Het belangrijkste was
dat ik een library moest vinden die de schermafhandeling doet en die vond ik in [textplots](https://github.com/loony-bean/textplots-rs). 
In een paar uur had ik iets in elkaar gesleuteld dat de logs (jdk20) parst met regular expressions en twee lijnen toont, een groene vóór garbage 
collection en een rode voor erna, de belangrijkste. 

Het resultaat is te vinden op [github](https://github.com/shautvast/jop). Na `cargo build --release` is de executable 2,5 Mb groot. De enige
parameter is de filenaam waar de logs in staan. De source bevat ook een java main, die je als voorbeeld zou kunnen gebruiken:
```bash
java -verbose:gc -cp src Filler >outfile
```

en dan:
```shell
jop outfile
```
of
```shell
cargo run outfile
```

Als jouw logs er anders uitzien, andere jdk versie, zou je de regex aan kunnen passen. Idealiter komt er een tweede argument
voor de java versie, zodat je verschillende regex'es kunt gebruiken.

That's it!

Wat nog mooi zou zijn is als het ook een unix pipe aan zou kunnen en wat ik nog niet getest heb is of het zou werken in bijvoorbeeld
openshift, als je terminal in de browser draait. Ik vermoed dat het wel moet kunnen, aangezien iets als VIM er ook in werkt 
(zie [https://developers.redhat.com/blog/2021/01/19/use-vim-in-a-production-red-hat-openshift-container-in-6-easy-steps]())

<div style="text-align: right">∞</div>