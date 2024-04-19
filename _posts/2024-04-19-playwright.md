---
layout: post
title: "Playwright!"
author: sander.hautvast
categories: [test, typescript]
image: assets/images/playwright/playwright-logo.png
beforetoc: "seeds"
featured: true
hidden: false
lang: nl
---
Ik ben gewoon blij met [playwright](https://playwright.dev/)!

Ik ben geen tester, maar heb altijd veel getest. Waar heb ik eerder mee gewerkt?
* fitnesse
* cucumber
* robotframework

Meestal was dat in de context van backend testing, alleen met robot (python) heb ik ook frontend testing gedaan. Ik neem aan dat je dit herkent: je tijd gaat zitten in het zoeken naar het juiste html element, je zit te wachten tot het weer gedraaid heeft.

**De feedback-cycle:**
1. aanpassing
2. start script
3. concludeer dat het nog niet werkt
4. debug in de browser developer console
5. terug naar 1

##### Nadelen:
1. je moet switchen tussen drie tools (editor, browser, commandline)
2. het duurt lang, zeker bij langere flows met afhankelijkheden (in zowel UI als testscript)
3. screenshots helpen, maar je kunt niet de test stilzetten en op dat moment het scherm inspecteren
4. shadow dom...

Het fijne en nieuwe van playwright is dat het voor al deze issues een oplossing biedt. Het is namelijk niet alleen een testrunner, die een rapportje genereert.
Met
`npx playwright test --ui`
start je een development omgeving, inclusief runner, browser (de drie bekende types) en debug tooling!

![playwwright ui](/assets/images/playwright/playwright.png)

En als je dan in de playwright.config.ts configureert dat je lokale frontend wordt gestart:
```typescript
webServer: {
    command: 'npm run start',
    url: 'http://127.0.0.1:3000',
    reuseExistingServer: !process.env.CI,
  },
```

En als je ook `watch` aanzet op je tests, dan zorgt elke wijziging voor een refresh van je testresultaat. Dat is eindelijk een comfortabele werkomgeving voor frontend testers.


##### locators
Playwright kent het begrip [locators](https://playwright.dev/docs/locators). 

Een hele simpele is deze:
`getByRole("button")`
Zie de rode pijlen in de afbeelding. De bovenste twee wijzen naar de elementen die playwright geselecteerd heeft met deze locator (onderste rode pijl).

In robot gebruikte ik altijd xpath. Dat is ook een mogelijkheid binnen playwright, maar heeft als nadeel dat het tegen de muren van de shadowroot op botst. `getByRole` kan daar doorheen. 

De button role geeft als zoekresultaat drie knoppen. Je moet op 1 element uitkomen om een actie of verwachting op uit te voeren.
De onderstaande voorbeeld test die playwright heeft gegenereerd, geeft een indruk hoe je dat kunt doen:

```typescript
import { test, expect } from '@playwright/test';

test('has title', async ({ page }) => {
  await page.goto('https://playwright.dev/');

  // Expect a title "to contain" a substring.
  await expect(page).toHaveTitle(/Playwright/);
});

test('get started link', async ({ page }) => {
  await page.goto('https://playwright.dev/');

  // Click the get started link.
  await page.getByRole('link', { name: 'Get started' }).click();

  // Expects page to have a heading with the name of Installation.
  await expect(page.getByRole('heading', { name: 'Installation' })).toBeVisible();
});
```

Je werkt met deze source in je favoriete editor. Er is een plugin voor visual studio code. Intellij werkte voor mij ook prima.

##### Timeline
Kijk nog eens naar de groene pijl in het plaatje. Er staat een timeline van het browser scherm door de test heen. Veel makkelijker dan handmatig screenshots genereren, om dan vast te stellen dat op dat moment het scherm nog niet gerenderd (en dus wit) is. Kan trouwens wel in playwright, maar nadat ik met de UI leerde omgaan, heb ik daar geen gebruik meer van gemaakt.

##### Wat is er gebeurd
![playwwright ui](/assets/images/playwright/playwright2.png)
Je kunt altijd precies nagaan wat de test gedaan heeft. Op sleutelpunten heeft het een scherm (nu geen screenshot, maar een inspecteerbare DOM) vastgehouden. Hier heb ik `locator.click` geselecteerd en playwright geeft met de rode stip aan waar hij op geklikt heeft. 

Hier geen volledige tutorial, want dat hebben ze zelf al gedaan. 

Takeaway van deze blog is dat playwright een orde van grootte beter is dan vergelijkbare tools. 

De enige aanmerking die je wat mij betreft kunt maken is dat hij van microsoft komt. Deze gigant lukt het dus om met een (nu) gratis tool markaandeel te krijgen. Iets wat voor startups niet mogelijk is. Daar heb ik wel bedenkingen bij. Maar ja, het is eindelijk eens leuk om te testen!

<div style="text-align: right">∞</div>