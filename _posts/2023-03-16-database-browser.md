---
layout: post
title:  "Big data in je browser"
author: sander.hautvast
categories: [database, webassembly, sqlite, java]
image: assets/images/national-cancer-institute-pCqzMe04s8g-unsplash.jpg
beforetoc: ""
featured: true
hidden: false
lang: nl
---
[SQLite](https://sqlite.org/index.html) kom je in de java wereld niet direct tegen. Er is een JDBC driver voor, dus technisch kan het, maar er is een belangrijk nadeel: je kunt er geen connectie mee openen. SQLite draait namelijk _embedded_. Het opent geen poorten zoals reguliere databases en meerdere gebruikers tegelijk is al gelijk een uitdaging. Aan de andere kant is het de meest gebruikte database ter wereld (!). 

Maar dan dus op mobiele telefoons, op IOT devices, of als onderdeel van een desktop applicatie. De runtime is zo klein dat je hem makkelijk mee kan bundelen (en de licentie staat dat ook toe). In plaats van een een gewoon bestand, open je een SQLite bestand en je leest hem met standaard SQL.

Ik zag afgelopen zomer een [presentatie](https://www.youtube.com/watch?v=lP_qdhAHFlg) van Steve Sanderson ([github](https://github.com/SteveSandersonMS), [blog](http://blog.stevensanderson.com/)) developer voor .Net en bedenker van [blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor). Een feature van blazor die hij liet zien is SQLite in de browser. Dit maakt het mogelijk om vele duizenden (100.000) rijen in de browser te tonen (dat performt traditioneel helemaal niet) en razendsnel te scrollen en te filteren. 

![plaatje](/assets/images/steve-sanderson-presentation-blazor.png)

_Ik ging kijken of er iets vergelijkbaars is voor de java wereld. Nee. Dat wil zeggen, nog niet..._

Enter [SQLighter](https://gitlab.com/sander-hautvast/sqlighter)...

Wat is SQLighter? Het is een java library die met je met een eenvoudige api in staat stelt een bestand te genereren in SQLite formaat. 

_Is dat handig?_

JSON is de de facto standaard voor backend api's. Het wordt native ondersteund door javascript en de (enterprise) java wereld maakt al jaren gebruik van jackon (of iets vergelijkbaars) om java objecten te serialiseren naar een leesbaar tekstformaat dat naar de browser gaat. Waarom zou je dat vervangen?

_Hoe draai je eigenlijk SQLite in de browser?_

Dat is in principe heel eenvoudig met [sql.js](https://github.com/sql-js/sql.js/). Met een paar commando's heb je een javascript scriptje waarin je SQL kunt praten tegen een database in die via webassembly (WASM) in de browser draait. Sinds kort is er ook een WASM build van SQLite zelf. Deze werkt onder meer met de opkomende standaard [OPFS](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API) voor het werken met bestanden in de browser.

__De combinatie van beide technologieën is >subtoptimaal<__
1. je krijgt data van de server in JSON formaat.
2. je start een lege database.
3. je gebruikt javascript om met SQL inserts een database te vullen.
4. je gebruikt SQL _selects_ om de data te tonen.

Als SQLite de data rechtstreeks kan lezen, heb je stap 1 en 3 in één keer gedaan.
Dus:
1. je krijgt data van de server in SQLite formaat.
2. je start de database met het bestand (dat gaat net zo snel).
3. je gebruikt SQL _selects_ om de data te tonen.


Handig? Jazeker! je hebt nu geen server roundtrips meer nodig om door je data te zoeken en je resultaat is er vrijwel direct. Dat is handig als je veel data hebt waar gebruikers zelf in kunnen (en mogen) zoeken. Dus niet voor het openen van je nieuwe bankrekening, maar wel als je de bankmedewerker via een browser toegang wil geven tot de gegevens van rekeninghouders. 

Hieronder zie je hoe je dat kunt doen met SQLighter. Doe de setup met `DatabaseBuilder`, maak nieuwe records met `new LtRecord(...)`, voeg er waardes (`LtValue`) aan toe en 'insert' ze met `addRecord`. Maak tot slot de de database met DatabaseBuilder.build() en schrijf deze met `write` weg naar een bestand of OutputStream. Er is ook een handige utility (`ResulSet2SQLite`) om een record in een `java.sql.ResultSet` om te zetten in een LtRecord.

De performance is vergelijkbaar met die van Jackson, dus je kunt dit rustig aan je rest-api toevoegen.

Volledig voorbeeld met dummy waardes (uit de demo):
```java
public Database getAllCustomersAsSQLite() {
        DatabaseBuilder databaseBuilder = new DatabaseBuilder();
        databaseBuilder.addSchema("customers",
                "create table customers (name varchar(100), email varchar(100), streetname varchar(100), housenumber integer, city varchar(100), country varchar(100))");

        final RandomStuffGenerator generator = new RandomStuffGenerator();
        long rowid = 1;

        for (int i = 0; i < 100_000; i++) {
            LtRecord record = new LtRecord(rowid++);
            record.addValues();

            String firstName = generator.generateFirstName();
            String lastName = generator.generateLastName();
            record.addValues(LtValue.of(firstName + " " + lastName),
                    LtValue.of(firstName + "." + lastName + "@icemail.com"),
                    LtValue.of(generator.generateStreetName()),
                    LtValue.of(generator.generateSomeNumber()),
                    LtValue.of(generator.generateSomeCityInIceland()),
                    LtValue.of(generator.generateIceland()));

            databaseBuilder.addRecord(record);
        }
        return databaseBuilder.build();
    }
```


Demo lokaal starten?

```bash
git clone https://gitlab.com/sander-hautvast/sqlighter.git
cd sqlighter/demo
bash start_api.sh
bash start_ui.sh
```

Of als je geen bash/zsh hebt:
```
cd sqlighter/demo/api
mvn -f api/pom.xml -DskipTests clean spring-boot:run
cd ../ui
npm run dev
```

Ga naar https://localhost:5173/ (Let op: gebruik Chrome en https!)

![plaatje](/assets/images/sqlighter-screenshot.png)

De demo UI is gebouwd met [Lit](https://lit.dev/). Dit javascript/typescript framework is wat ik ook gebruik bij mijn huidige klant. In vergelijking met bijvoorbeeld _angular_ valt op hoe lichtgewicht het is. 

Er is een ook een branch waarin de UI gebaseerd is op SQL.js. Deze is getest in _Firefox_.

Het grote nadeel ervan is dat zowel SQL.js als de SQLite WASM build geen ESM modules zijn, zodat je 10 jaar terug bent in de tijd qua web development. Er is echter een wrapper ([https://github.com/overtone-app/sqlite-wasm-esm/](https://github.com/overtone-app/sqlite-wasm-esm/)) die maakt dat je SQLite/WASM kunt gebruiken als `import` in een _[vite](https://vitejs.dev/)_ project. Andere build systemen, webpack, parcel etc, zou ook moeten kunnen, maar dat werkt niet _out of the box_. Vite en Parcel zijn nieuwe buildsystemen, die lekker werken vanwege hun snelheid en kleine hoeveelheid setup die je nodig hebt om aan de slag te kunnen.

<div style="text-align: right">∞</div>