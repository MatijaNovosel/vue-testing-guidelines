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
    - [Uvod](#uvod)
    - [Autentifikacija](#autentifikacija)
    - [Smjernice](#smjernice)
    - [Environment](#environment)
    - [Pokretanje testova](#pokretanje-testova)
  * [Component](#component)
    - [Uvod](#uvod)

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

#### Uvod

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

Testiranje komponenti je slično E2E testiranju u smislu da se isto koristi Cypress.

WIP
