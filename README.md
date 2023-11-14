<div align="center">
  <img src="https://user-images.githubusercontent.com/36193643/205978239-f29e0f55-a561-4b90-931c-066045662f9b.png" />
</div>

<h1 align=center>Vue testing guidelines</h1>

U ovom dokumentu opisati će se neke smjernice toga kako testirati aplikaciju na e2e, unit i component način kao i tehnički dio načina testiranja.

## Sadržaj

- [Uvod](#uvod)
- [E2E i component testiranje](#e2e-i-component-testiranje)
  - [Struktura direktorija](#struktura-direktorija)
  * [E2E](#e2e)
    - [Autentifikacija](#autentifikacija)
    - [Smjernice](#smjernice)
    - [Environment](#environment)
    - [Pokretanje testova](#pokretanje-testova)
  * [Component](#component)
    - [Primjer component testova](#primjer-component-testova)
    - [Pokretanje component testova](#pokretanje-component-testova)
- [Unit](#unit)
  - [Primjer unit testova](#primjer-unit-testova)
  - [Pokretanje unit testova](#pokretanje-unit-testova)

## Uvod

Sučelje ovog projekta testira se na tri različita načina:

1. End to end testing (E2E) - simulacija korisnika koji prolazi kroz aplikaciju
2. Component - Izolirane komponente nad kojima se provode razni eksperimenti
3. Unit - Individualne funkcije, klase i moduli

E2E i component testiranje rade se pomoću [Cypressa](https://www.cypress.io/), dok se unit testiranje radi preko [Vitesta](https://vitest.dev/).

I jedno i drugo je odabrano iz razloga što je Vue tim radio na njima i savršeno odgovaraju projektima koji su napravljeni s tim radnim okvirom.

## E2E i component testiranje

E2E i component testiranje zajedno su u jednom poglavlju zbog načina kako Cypress funkcionira - konfiguracijske datoteke koje diktiraju kako se provode testovi nalaze se unutar `cypress` direktorija na vrhu repozitorija te i sami e2e testovi, dok se component testovi nalaze individualno u direktorijima od komponenti.

### Struktura direktorija

Struktura `/cypress` direktorija je kao što slijedi:

- `/tsconfig.json` - TypeScript konfiguracijska datoteka specifična za Cypress

- `/downloads/` - privremene datoteke koje se koriste za prijenos podataka

- `/e2e/` - ovdje se nalaze datoteke u kojima su opisani end to end testovi, ekstenzija je `.cy.ts` npr. `dashboard.cy.ts`

- `/fixtures/` - datoteke koje se koriste za mockanje podataka, konstante

- `/support/` - konfiguracijske datoteke specifične za testiranje

  - `commands.ts` - deklariranje dodatnih naredbi koje se koriste prilikom testiranja, primjerice za autentifikaciju:

  ```ts
  Cypress.Commands.add("login", (username, password) => {
    cy.visit(`${Cypress.env("base_url")}/login`);
    cy.get("input[name='username']").type(username);
    cy.get("input[name='password']").type(`${password}{enter}`);
  });
  ```

  - `component-index.html` - inicijalna točka gdje se virtualna Vue aplikacija mounta kako bi se moglo izvršiti testiranje

  - `components.ts` - konfiguracijska datoteka za component testiranje, generalno se deklariraju dodatne naredbe za pomoć pri ovom tipu testiranja

  - `e2e.ts` - konfiguracijska datoteka za e2e testiranje, generalno se deklariraju dodatne naredbe za pomoć pri ovom tipu testiranja

Van ovog direktorija se također nalaze konfiguracijske i deklaracijske datoteke na vrhu repozitorija zvane `cypress.config.ts` i `cypress.d.ts`.

Unutar konfiguracijske datoteke `cypress.config.ts` definirane su top level postavke za e2e i component testiranje, dok se unutar deklaracijske datoteke `cypress.d.ts` tipiziraju naredbe koje se koriste prilikom testiranja.

### E2E

Kod end to end testiranja nastoji se simulirati točni slijed operacija kojime bi korisnik obavio nešto na aplikaciji poput autentificiranja, upisivanja teksta u polja za unos pa zatim odlazak na neki drugi dio aplikacije gdje još dodatno radi neke stvari koje se mogu testirati npr. uređivanje profila.

Za razliku od unit i component testiranja, ovi testovi se vrše nad pravom aplikacijom i to preko automatiziranih testova.

#### Autentifikacija

Kod skoro svake aplikacije postoji neki oblik autentifikacije i autorizacije sadržaja, a prilikom testiranja svaki put bi u pravilu trebali započeti s cijelim procesom nanovo kako bi imali sve čisto i spremno za samo taj određeni dio.

U tu svrhu napisana je `login` naredba unutar `commands.ts` datoteke i tokom e2e testiranja se poziva prije svakog ciklusa funkcijom `beforeEach`, primjerice:

```ts
describe("Dashboard tests", () => {
  beforeEach(() => {
    cy.login(Cypress.env("username"), Cypress.env("password"));
  });

  it("should be able to access the dashboard screen", () => {
    cy.visit(`${Cypress.env("base_url")}/dashboard`);
    cy.url().should("contain", "/dashboard");
    cy.title().should("eq", "KSF");
  });
});
```

#### Smjernice

Svaki test bi trebao biti usredotočen na neku cjelinu poput jednog ekrana, u ovom slučaju upravljačka ploča što je enkapsulirano `describe` funkcijom čija je svrha samo objediniti nekoliko testova.

Recimo da korisnik želi doći na već spomenuti ekran gdje postoji popis kartica sa slikama grafova i gumb za dodatak nove stavke, pa želi dodati novu karticu pritiskom na gumb - testirali bi to primjerice na ovakav način:

```ts
describe("Dashboard tests", () => {
  beforeEach(() => {
    cy.login(Cypress.env("username"), Cypress.env("password"));
  });

  it("should be able to access the dashboard screen", () => {
    cy.visit(`${Cypress.env("base_url")}/dashboard`);
    cy.url().should("contain", "/dashboard");
    cy.title().should("eq", "KSF");
  });

  it("should be able to add a new item", () => {
    cy.visit(`${Cypress.env("base_url")}/dashboard`);
    cy.get("#chart-row").children().should("have.length", 12);
    cy.get("#open-add-chart-dialog-btn").click();
    cy.get("#create-chart").click();
    cy.get("#chart-row").children().should("have.length", 13);
  });
});
```

Cypress koristi sličan sustav dohvaćanja elemenata kao i jQuery uz pomoć `get` funkcije što je temeljni dio ovakvog načina testiranja - **treba se usredotočiti samo na vizualni dio sučelja, ne dirati interno stanje ni podatke koji su proizvedeni posljedicom**. Kod E2E testiranja trebamo simulirati način kako korisnik koristi aplikaciju i k tome izbjegavati zabadati u to **kako** nešto radi, već **što** radi.

Ono što bi se trebalo provjeravati je: **odgovara li sučelje na pravilan način kako bi trebalo unutar postavljenih uvjeta?** Ovdje smo provjeravali postoji li novi stavka unutar popisa tj. je li se povećala količina child elemenata unutar elementa roditelja.

Kako bi se moglo konkretnije testirati neki dio sučelja, vrlo vjerojatno će se morati dodati neka konkretna naznaka o kojem se elementu radi pomoću `id` ili `name` atributa jer inače je skoro pa nemoguće na normalan način dohvatiti neki element za unos ili gumb. Ponekad je neizbježna situacija gdje će se morati dohvaćati element na čudan način (pomoću nekog jako specifičnog selektora), ali bi se u prvom planu trebalo voditi s konkretnijim nazivljem.

<table>
<tr align="center">
<td> 🟥 </td> <td> 🟩 </td>
</tr>
<tr>
<tr>
<td>

```html
<input class="username-klasa" />
<ul>
  <li>Prva stavka</li>
  <li>Druga stavka</li>
</ul>
```

```ts
cy.get(".username-klasa").type("Nešto");
cy.get("ul:first-child").should("have.text", "Prva stavka");
```

</td>
<td>

```html
<input name="username" />
<ul>
  <li id="prva-stavka">Prva stavka</li>
  <li>Druga stavka</li>
</ul>
```

```ts
cy.get("input[name='username']").type("Nešto");
cy.get("#prva-stavka").should("have.text", "Prva stavka");
```

</td>
</tr>
</table>

Uz sve navedeno opisi testova bi trebali glasiti nalik na: `should do thing` i varijacije te rečenice, ovisno o situaciji.

#### Environment

Ponekad je potrebno da se aplikacija testira unutar raznih okolina poput QA, u produkciji i slično pa kad se testovi pišu nebi trebali hardkodirati vrijednosti koje su specifične za tu okolinu poput informacija o korisniku preko kojeg se obavlja testiranje te aplikacije.

Postoji par načina kojime se prenose te informacije, u ovom slučaju preko naredbe za pokretanje i to pomoću argumenata koji se proizvoljno imenuju, primjerice za QA:

```bash
cypress open --env base_url=https://localhost:8080,username=admin1,password=admin1
```

Unutar samog koda ti argumenti se dohvaćaju preko `Cypress.env()` funkcije:

```ts
Cypress.env("username");
```

#### Pokretanje testova

Testovi se pokreću preko Cypressovog sučelja, u tu svrhu napisano je par naredbi u `package.json` za različite okoline:

```json
{
  "test:e2e": "cypress open --env base_url=https://localhost:8080,username=admin1,password=123",
  "test:e2e:qa": "cypress open --env base_url=192.168.2.1,username=admin1,password=123",
  "test:e2e:uat": "cypress open --env base_url=192.168.2.2,username=admin1,password=123",
  "test:e2e:prod": "cypress open --env base_url=192.168.2.3,username=admin1,password=123",
  "test:e2e:preprod": "cypress open --env base_url=192.168.2.4,username=admin1,password=123"
}
```

### Component

Testiranje komponenti je slično E2E testiranju u smislu da se isto koristi Cypress i trebalo bi se usredotočiti na dvije ključne stavke:

- **Vizualna logika** - reagira li komponenta pravilno na promjene prenesenih podataka (propovi, slotovi)
- **Logika ponašanja** - reagira li komponenta pravilno na korisnični podražaj

Ni pod razno se nebi trebalo dirati interno stanje komponente ili njegove metode, ako je potrebno testirati baš specifičnu metodu vezanu uz komponentu to se radi preko unit testova izolirano.

Svaki test bi se trebao pisati unutar direktorija gdje se nalazi sama komponenta, npr. ako postoji komponenta `slider.vue` trebala bi postojati i datoteka `slider.cy.ts` na istoj razini ili unutar posebnog foldera zvanog `tests`.

#### Primjer component testova

```ts
import { hexToRgb } from "@/shared/helpers/misc";
import { capitalize } from "@/shared/helpers/string";
import DatePicker from "../CustomDatePicker.vue";
import { createNativeLocaleFormatter } from "../helpers";

const currentDate = new Date();

// 1
const defaultProps = {
  modelValue: currentDate.toISOString().substring(0, 10),
  max: "2023-12-12",
  min: "2020-01-24",
  firstDayOfWeek: 1,
  locale: "en-US"
};

// 2
describe("DatePicker", () => {
  // 3
  it("should mount", () => {
    cy.mount(DatePicker, {
      props: defaultProps
    });
  });

  // 4
  it("should have the default appearance", () => {
    const currentYear = currentDate.getFullYear().toString();
    cy.mount(DatePicker, {
      props: defaultProps
    });
    cy.get(".date-picker-title").should(
      "have.css",
      "background-color",
      hexToRgb("2e79bd")
    );
    cy.get(".date-picker-title__year").should("have.text", currentYear);
  });

  // 5
  it("should react to model value changes", () => {
    cy.mount(DatePicker, {
      props: defaultProps
    });
    const nextDay = new Date(currentDate.getTime() + 86400000)
      .toISOString()
      .substring(0, 10);
    cy.vue().then((wrapper) => {
      wrapper.setProps({
        // Current date + 1 day
        modelValue: nextDay
      });
    });
  });

  // 6
  it("should react to locale value changes", () => {
    const locale = "hr-HR";
    cy.mount(DatePicker, {
      props: defaultProps
    });
    cy.vue().then((wrapper) => {
      wrapper.setProps({
        locale
      });
      // Formatiranje - mjesec i godina
      const formatter = createNativeLocaleFormatter(
        locale,
        { month: "long", year: "numeric", timeZone: "UTC" },
        { length: 7 }
      );
      const expectedText = capitalize(
        formatter!(currentDate.toISOString().substring(0, 10))
      );
      cy.get(".date-picker-header__value").should("have.text", expectedText);
    });
  });

  // 7
  it("should be able to scroll through months", () => {
    cy.mount(DatePicker, {
      props: defaultProps
    });
    cy.get("#prev-month").click();
    cy.get("#next-month").click();
  });
});
```

Opisano po točkama:

1. Prije samog testiranja ponekad je potrebno deklarirati neke statičke podatke koji će se naknadno koristiti, mogu se deklarirati i tokom pojedinačnih testova
2. Svaki test kao i kod E2E treba opisati, generalno kod testiranja komponenti je to sami naziv komponente
3. Inicijalni test koji bi trebalo provesti je: Hoće li se komponenta uopće renderati? Ovo radimo `mount` funkcijom pa zatim dodatno definiramo neke podatke koji su specifični toj komponenti
4. Jedan od testova bi trebao sadržavati vizualni opis stanja komponente kad se rendera, generalno samo provjere usklađuje li se stil i izgled onome kako bi trebalo biti
5. Iako je spomenuto da nebi trebali dirati interno stanje komponente, u nekim testovima je potrebno samo promijeniti vrijednost na nešto drugo kako bi se moglo vidjeti kako će komponenta reagirati i hoće li biti reaktivna
6. Isto kao i prošla točka, ali se mijenja drugi prop i gleda reaktivnost u ovisnosti na to - sve promjene internog stanja rade se pomoću custom naredbe `cy.vue()`
7. Jednostavno testiranje klikanjem po komponenti

#### Pokretanje component testova

Testovi za komponente mogu se pokrenuti ili preko interaktivnog sučelja od E2E testova ili preko komandne linije:

```json
{
  "test:component": "cypress run --component"
}
```

## Unit

Najjednostavnije od svih oblika testiranja, unit testiranje se bavi izoliranim funkcijama, klasama i modulima koji bi u pravilu trebali funkcionirati Single Responsibility principom (SRP) tj. trebali bi obavljati ono što trebaju obavljati i ništa više od tog.

U tu svrhu koristi se Vitest jer je jednostavan, brz i pravio ga je isti tim kao i Vue tim.

Unit testovi bi trebali biti kategorizirani ovisno o svojoj svrsi npr. ako testiramo neke funkcije koje se koriste za pripomoć u radu aplikacije zvane `helpers.ts` onda bi stvorili folder na istoj razini zvan `tests` s datotekom za testiranje `helpers.test.ts`.

### Primjer unit testova

U ovom primjeru se testira hoće li se tekst pravilno formatirati tj. hoće li dobiti veliko prvo slovo hoće li se pravilno stvoriti akronim:

```ts
import { describe, expect, test } from "vitest";

import { acronym, capitalize } from "../string";

describe("string tests", () => {
  test("should produce an acronym", () => {
    expect(acronym("Ministarstvo Unutarnjih Poslova")).toBe("MUP");
  });

  test("text should be capitalized", () => {
    expect(capitalize("branko")).toBe("Branko");
  });
});
```

Kao i prije testovi su grupirani `describe` funkcijom pa zatim je svaki test funkcije odvojen u svoj `test` blok istoimenom funkcijom pa zatim se koristi `expect` funkcija od Vitesta.

### Pokretanje unit testova

Ovi testovi mogu se pokrenuti ili preko interaktivnog sučelja ili preko komandne linije:

```json
{
  "test:unit": "vitest run",
  "test:unit:ui": "vitest --ui"
}
```
