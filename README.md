# TBA3 Prototypische Rückmeldung im Fach Mathematik Klassenstufe 8

[![Lizenz: MIT](https://img.shields.io/badge/Lizenz-MIT-yellow.svg)](LICENSE)

Eine Webanwendung zur **Aufbereitung von VerA-8-Ergebnisdaten für Schüler:innen**, die die eigenen Testergebnisse
schrittweise, altersgerecht und motivierend zurückmeldet.
Das Projekt ist eine reine **Vue 3 + Vite Single-Page-App** (kein eigenes Backend). Die Daten kommen über generierte
API-Clients aus der **TBA3-Schnittstelle**, die Texte über eine zur Laufzeit geladene Konfigurationsdatei.

* [Technische Dokumentation](#technische-dokumentation)
* [Inhaltliche Dokumentation](#inhaltliche-dokumentation)

---

## Technische Dokumentation

### Entwicklungsumgebung

#### Voraussetzungen

- [Node.js](https://nodejs.org/) ≥ 20.19 oder ≥ 22.12 (s. `engines` in `package.json`)
- [pnpm](https://pnpm.io/) (via `corepack enable pnpm`)
- Optional: [Docker](https://www.docker.com/) ≥ 24 für die containerisierte Entwicklung
- Optional: Java 21 – nur nötig, wenn die API-Clients neu generiert werden (s. „TBA3-Schnittstelle")

#### Einrichtung

Drei Konfigurationsdateien werden zur Laufzeit erwartet und sind **nicht im Repository** enthalten
(s. `.gitignore`, Abschnitt „Konfiguration"):

```bash
# Laufzeit-Konfiguration der API-Endpunkte aus Vorlage kopieren
cp public/config.example.js public/config.js

# Optional: Umgebungsvariablen/Build-Fallback kopieren
cp .env.example .env

# Redaktionelle Texte, Schwellenwerte und Übungslinks
cp src/assets/competence_guidingideas_texts.json public/config.json
```

Danach die Abhängigkeiten installieren:

```sh
pnpm install
```

#### Starten

```sh
# Compile und Hot-Reload für Entwicklung
pnpm run dev
```

- Frontend → [http://localhost:3000](http://localhost:3000)

Alternativ mit Docker – der Container installiert beim Start die Abhängigkeiten und hält sich offen,
`pnpm run dev` wird darin ausgeführt:

```sh
docker compose up -d
docker compose exec node pnpm run dev
```

> Hinweis: `docker-compose.yml` steht in `.gitignore`. Die im Repository liegende Datei dient als Vorlage und kann lokal
> angepasst werden, ohne dass die Änderungen versioniert werden.

#### Skripte

| Skript                          | Beschreibung                                                        |
|---------------------------------|---------------------------------------------------------------------|
| `pnpm run dev`                  | Dev-Server mit Hot Reload (Port 3000, inkl. API-Proxys)             |
| `pnpm run build`                | Type-Check und Production-Build (`vue-tsc` + `vite build`)          |
| `pnpm run build-only`           | Nur Build, ohne Type-Check                                          |
| `pnpm run type-check`           | Type-Check via `vue-tsc --build`                                    |
| `pnpm run preview`              | Vorschau des Production-Builds (Port 4173, inkl. API-Proxys)        |
| `pnpm run test:unit`            | Unit-Tests mit [Vitest](https://vitest.dev/)                        |
| `pnpm run lint`                 | Lint mit [oxlint](https://oxc.rs/) und [ESLint](https://eslint.org/), jeweils mit `--fix` |
| `pnpm run format`               | Formatierung von `src/` mit [Prettier](https://prettier.io/)        |
| `pnpm run generate:api-resources` | API-Client aus der TBA3-Spezifikation neu generieren               |
| `pnpm run generate:api-new`     | API-Client für die inio-Reportdaten neu generieren                  |
| `pnpm run generate:api-auth`    | API-Client für die inio-Authentifizierung neu generieren            |

#### Code-Konventionen

- Prettier: keine Semikolons, einfache Anführungszeichen, Einrückung 4, Zeilenlänge 140 (`.prettierrc.json`)
- Linting: oxlint (Kategorie `correctness` als Fehler) und ESLint mit Vue-/TypeScript-/Vitest-Regeln
- Empfohlene VS-Code-Erweiterungen liegen in `.vscode/extensions.json` (Volar, Vitest Explorer, ESLint, oxc, Prettier)

---

### Konfiguration

Die App ist so gebaut, dass Endpunkte und Texte **ohne Rebuild** ausgetauscht werden können. Beide Dateien liegen in
`public/` und sind bewusst nicht versioniert.

#### `public/config.js` – Laufzeit-Endpunkte

Wird in `index.html` als klassisches Script eingebunden und setzt `window.appConfig`. Ausgelesen wird sie in
`src/queries/utils.ts`:

```js
window.appConfig = {
    api: {
        baseUrl: '/api-proxy',        // TBA3-Referenz-API (@tba3/api-resources)
        inioApiUrl: '/api-inio',      // inio-Reportdaten (@tba3/api-new)
        inioAuthApiUrl: '/api-auth',  // inio-Authentifizierung (@tba3/api-auth)
        xApiKeySchool: 'TEST',        // Header X-API-KEY-SCHOOL
    },
    defaultPageSize: 20,
}
```

Fehlt ein Wert, wird ein leerer `basePath` verwendet, d.h. es wird gegen den eigenen Origin (und damit im Dev-Betrieb
gegen die Vite-Proxys) angefragt. Der API-Key fällt zusätzlich auf `VITE_X_API_KEY_SCHOOL` und schließlich auf `TEST`
zurück.

#### Ersteinrichtung:
Kopiere für die lokale Entwicklung die Vorlage `public/config.example.js` nach `public/config.js` und passe die Werte bei Bedarf an.

##### Optional: `.env` als Build-Fallback
Standardmäßig wird die Konfiguration zur Laufzeit über `public/config.js` geladen (kein Rebuild erforderlich). 
Falls du den API-Key stattdessen zur Build-Zeit fest in das Bundle einbrennen möchtest, kannst du eine `.env`-Datei auf Basis von `.env.example` anlegen:

```env
VITE_X_API_KEY_SCHOOL=TEST
```

Sicherheitshinweis zum API-Key:
Da es sich um eine reine Client-Anwendung (Single Page Application) handelt, ist der xApiKeySchool für Endnutzer im Browser jederzeit einsehbar. Trage hier niemals geheime Server-Keys oder Admin-Credentials ein, sondern ausschließlich dafür vorgesehene Public-/Schul-API-Keys.

#### `public/config.json` – Inhalte & Schwellenwerte

Diese Datei steuert alle **redaktionellen Texte, Rückmeldungs-Logiken und Schwellenwerte**. Sie wird beim Start der Anwendung geladen (`fetch('/config.json')`), sodass Texte und Bewertungsgrenzen **ohne Rebuild der Anwendung** angepasst werden können.

##### Ersteinrichtung:
Kopiere die im Repository enthaltene Vorlage `src/assets/competence_guidingideas_texts.json` nach `public/config.json` oder nutze direkt die `public/config.json`

##### Struktur der Datei:

| Schlüssel                | Inhalt                                                                 |
|--------------------------|------------------------------------------------------------------------|
| `competence_texts`       | Texte und Beschreibungen je Kompetenz `K1`–`K6`                        |
| `guiding_ideas_texts`    | Texte und Beschreibungen je Leitidee `L1`–`L5`                          |
| `start`                  | Titel und Info-Text des Einstiegs (auch im Info-Modal der Fußzeile)     |
| `overallResult`          | Rückmeldetexte je Gesamtergebnis-Stufe (`K1A`, `K1B`, `K2`–`K5`)        |
| `specialCases`, `specialCasesAdvices` | Texte und Hinweise für auffällige Bearbeitungsmuster       |
| `areas`, `cutOffs`       | Bereichsgrenzen und Schwellenwerte für die Einordnung der Ergebnisse    |
| `testInfo`               | Metadaten zum Testheft (Booklet, Fach, Klassenstufe)                    |
| `exerciseLinks`          | Übungs-/Vertiefungslinks für die Abschlussseite                         |

---

### Technologien & Bibliotheken

- [Vue 3](https://vuejs.org/) – JavaScript-Frontend-Framework, Composition API mit `<script setup>`
- [Vite](https://vitejs.dev/) – Dev- und Build-Tool
- [TypeScript](https://www.typescriptlang.org/) – Typisierung, Type-Check via `vue-tsc`
- [Vue Router](https://router.vuejs.org/) – Routing der Schritt-Seiten
- [Pinia](https://pinia.vuejs.org/) – State Management (Modal-Store)
- [TanStack Query](https://tanstack.com/query/) – Data Fetching und Caching
- [Vue I18n](https://vue-i18n.intlify.dev/) – Oberflächentexte (aktuell nur `de`)
- [Apache ECharts](https://echarts.apache.org/) via [vue-echarts](https://github.com/ecomfe/vue-echarts) – Datenvisualisierung
- [dotLottie für Vue](https://developers.lottiefiles.com/docs/dotlottie-player/) – Animationen
- [Day.js](https://day.js.org/) – Zeit-/Dauerberechnungen (z.B. Bearbeitungsdauer)
- [vite-svg-loader](https://github.com/jpkleemans/vite-svg-loader) – SVGs als Vue-Komponenten (`?component`)
- [OpenAPI Generator](https://openapi-generator.tech/) – Erzeugung der API-Clients (`typescript-fetch`)
- [Vitest](https://vitest.dev/) – Unit-Tests, [oxlint](https://oxc.rs/) und [ESLint](https://eslint.org/) – Linting

---

### Projektstruktur

Das Repository ist ein **pnpm-Workspace**: die App liegt im Wurzelverzeichnis, die generierten API-Clients in
`packages/`.

```
.
├── src/                # Vue-3-App
├── packages/           # Generierte API-Clients (pnpm-Workspace)
│   ├── api-resources/  # @tba3/api-resources – TBA3-Referenz-API
│   ├── api-new/        # @tba3/api-new       – inio-Reportdaten
│   └── api-auth/       # @tba3/api-auth      – inio-Authentifizierung
├── public/             # Statische Assets, config.js(Vorlage: config.example.js) und config.json (nicht versioniert)
├── docker/
│   ├── Dockerfile              # Node 24 + pnpm + Java (für die Client-Generierung)
│   └── development.entrypoint  # pnpm install und Container offen halten
├── docker-compose.yml  # Dev-Container (Vorlage, lokal nicht versioniert)
├── .env.example        # Vorlage für Build-Umgebungsvariablen
├── vite.config.ts      # Build, Alias @ → src, Dev-/Preview-Proxys
├── vitest.config.ts    # Test-Setup (jsdom)
└── .gitlab-ci.yml      # CI: Audit der Produktionsabhängigkeiten
```

#### App-Struktur

```
src/
├── views/          # Seiten der Schritt-Navigation (eine pro Route)
├── components/     # Wiederverwendbare Komponenten inkl. Diagramme
├── composables/    # Datenbeschaffung und fachliche Auswertung (use*.ts)
├── queries/        # API-Konfiguration (Basis-URLs, API-Key)
├── stores/         # Pinia-Stores
├── router/         # Routendefinition der Schritte 1–7
├── services/       # Laden der Laufzeit-Konfiguration (config.json)
├── locales/        # Übersetzungen (de.json)
├── themes/         # Fonts, Farbvariablen, Icons
├── assets/         # Animationen, SVGs, Styles, Textvorlage
├── types.ts        # Fachliche Typen und Mappings (Kompetenzen, Leitideen, Stufen)
└── __tests__/      # Unit-Tests
```

#### Komponenten- und View-Struktur

Views und Komponenten liegen jeweils in einem **eigenen Unterordner** nach demselben Muster:

```
KomponentenName/
├── KomponentenName.vue   # Komponente (Composition API, <script setup>)
├── styles.module.css     # Lokale CSS-Module-Styles
└── icons/                # Nur von dieser Komponente genutzte Icons/Bilder
```

Übergreifende Styles liegen bewusst außerhalb: `src/themes/` (Fonts, Farb- und Designvariablen),
`src/assets/styles/base.css` (Basis-Styles) und `src/assets/styles/component-layout.module.css`
(geteiltes Layout-Modul). SVG-Icons werden über `vite-svg-loader` als Komponenten importiert:

```ts
import InfoIcon from '@/themes/icons/info.svg?component'
```

**Kernprinzip:** Views holen die Daten über Composables und geben sie als Props an die Darstellungskomponenten weiter.
Diagrammkomponenten (`BarCharts`, `SingleBarChart`, `StackedBarChart`) enthalten keine Fetch-Logik, sondern erhalten
ihre Werte von der einbindenden View.

---

### Anwendungsablauf

Die Rückmeldung ist als lineare Abfolge von Schritten aufgebaut. Der Code der Schüler:in wird als Query-Parameter
`?user=<code>` durch alle Schritte mitgeführt.

| Route             | View               | Inhalt                                                                    |
|-------------------|--------------------|---------------------------------------------------------------------------|
| `/` → `/step-1`   | `HomeView`         | Auswahl des Test-Codes und Einstieg                                       |
| `/step-2`         | `AnimationView`    | Animierte Einordnung des Gesamtergebnisses                                |
| `/step-3`         | `SpecialView`      | „Das ist uns aufgefallen" – richtig/falsch/nicht bearbeitet               |
| `/step-4/:subId?` | `CompetenceView`   | „Hier bist du stark" – stärkste Kompetenzen (`K1`–`K6`)                   |
| `/step-5/:subId?` | `GuidingIdeaView`  | Stärkste Leitideen (`L1`–`L5`)                                            |
| `/step-6/:subId?` | `BadResultsView`   | Bereiche mit Übungsbedarf                                                 |
| `/step-7`         | `FinalView`        | Abschluss mit Übungs- und Vertiefungslinks                                |

Die tatsächliche Schrittfolge ist **datenabhängig**: `src/composables/useNavigation.ts` baut aus den ermittelten
Top-Performern und schwachen Bereichen eine Liste aller Schritte auf. Für jede starke Kompetenz bzw. Leitidee entsteht
ein eigener Unterschritt (`:subId`); gibt es mehr als zwei schwache Bereiche, wird Schritt 6 zu einer einzelnen
Übersichtsseite zusammengefasst. Die Fußzeile in `App.vue` rendert daraus die Seitenindikatoren und die Vor-/Zurück-Navigation.

#### Composables

| Composable                | Aufgabe                                                                       |
|---------------------------|-------------------------------------------------------------------------------|
| `useUserItems`            | Items einer Schüler:in laden (`useUserItemsNew`), Trefferquote berechnen, Schulform und Testdaten |
| `useCompetencesNew`       | Auswertung je Kompetenz `K1`–`K6` inkl. Vergleichswerte Gym/Nicht-Gym          |
| `useGuidingIdeasNew`      | Auswertung je Leitidee `L1`–`L5`, Top- und schwache Bereiche                    |
| `useOverallResultsNew`    | Gesamtscore und Zuordnung zu einer Rückmeldestufe (`K1A`–`K5`)                  |
| `useSpecialCasesNew`      | Auffälligkeiten wie Bearbeitungsdauer und Anteil nicht bearbeiteter Aufgaben     |
| `useNavigation`           | Aufbau der datenabhängigen Schrittfolge                                        |

Die Varianten ohne `New`-Suffix (`useCompetences`, `useGuidingIdeas`, `useOverallResults`, `useSpecialCases`) sind die
älteren Implementierungen gegen die TBA3-Referenz-API und bleiben zu Vergleichszwecken erhalten. Aktiv verwendet werden
die `New`-Varianten gegen die inio-Reportdaten.

---

### TBA3-Schnittstelle

Die TBA3-Schnittstelle soll die Daten für Abbildungen möglichst standardisieren. Die Schnittstelle muss zwingend für
Abbildungen bedient werden (Projektziel von TBA3).

#### Dokumentation

Die Dokumentation der Schnittstelle liegt hier vor: https://apps.indibit.eu/tba3-api/docs

#### Mock Server

Beispieldaten können hier abgerufen werden: https://apps.indibit.eu/tba3-api

#### API-Clients

Die Clients werden **nicht von Hand geschrieben**, sondern mit dem OpenAPI Generator (`typescript-fetch`) aus den
jeweiligen Spezifikationen erzeugt und als Workspace-Pakete eingebunden. Jedes Paket enthält unter `src/` die Ordner
`apis/`, `models/` und `docs/` sowie `runtime.ts`.

| Paket                  | Spezifikation                                                             | Neu generieren                      |
|------------------------|---------------------------------------------------------------------------|-------------------------------------|
| `@tba3/api-resources`  | `github.com/indibit-eu/tba3` → `tba3-spec.yml`                            | `pnpm run generate:api-resources`   |
| `@tba3/api-new`        | `https://api.inio.de/swagger/report_data_tba3.json`                       | `pnpm run generate:api-new`         |
| `@tba3/api-auth`       | `https://api.inio.de/swagger/auth.json`                                   | `pnpm run generate:api-auth`        |

Die Generierung benötigt **Java 21** (der OpenAPI Generator ist ein Java-Tool); das Dev-Image in `docker/Dockerfile`
bringt es bereits mit. Die erzeugten Dateien werden anschließend automatisch mit Prettier formatiert. Generierter Code
sollte nicht manuell verändert werden – stattdessen die Spezifikation anpassen und neu generieren.

#### Dev-Proxys

Um CORS im Entwicklungsbetrieb zu umgehen, leitet Vite drei Pfade weiter (identisch konfiguriert für `dev` und
`preview`, s. `vite.config.ts`):

| Pfad          | Ziel                                        | Verwendet von         |
|---------------|---------------------------------------------|-----------------------|
| `/api-proxy`  | `https://apps.indibit.eu/tba3-api/`         | `@tba3/api-resources` |
| `/api-inio`   | `https://api.inio.de/report_data_tba3`      | `@tba3/api-new`       |
| `/api-auth`   | `https://api.inio.de/report_data_tba3`      | `@tba3/api-auth`      |

Im Production-Betrieb müssen die Endpunkte stattdessen über `public/config.js` als absolute URLs gesetzt werden.

---

### Tests & Qualitätssicherung

Die Anwendung nutzt **Vitest** in Kombination mit **Vue Test Utils** und **jsdom** für Unit- und Komponententests.

#### Commands

```sh
pnpm run test:unit            # Führt alle Unit-Tests einmalig aus
pnpm run test:unit --watch    # Watch-Modus für die lokale Entwicklung
pnpm run test:unit --coverage # Erstellt den Coverage-Bericht (html/text)
pnpm run type-check           # TypeScript Typprüfung via vue-tsc
pnpm run lint                 # Linter-Prüfung und automatische Fixes (oxlint/ESLint)
```

#### Abdeckung & Scope

- **Komponenten (`src/components/`)**: Visualisierung, Accessibility (Focus Traps, ARIA-Attribute), Event-Handling und Interaktionen (z. B. SideModal, Charts).
- **Store & Routing (`src/stores/`, `src/router/`)**: Pinia-Zustandsänderungen und Navigation.
- **Composables & Utils (`src/composables/`, `src/queries/`)**: Geschäftslogik, Daten-Aggregation und Berechnungen.

#### Test-Besonderheiten & Mocks

- **Web Speech API**: Die `speechSynthesis`-Schnittstelle für TTS (Text-to-Speech) im `SideModal` wird global gemockt.
- **Teleport & DOM**: Tests mit Teleport (`<Teleport to="body">`) werden mit `attachTo: document.body` gemountet, um echte DOM-Interaktionen und Tastatur-Events (`Escape`, `Tab`) zu verifizieren.

#### CI

`.gitlab-ci.yml` definiert die Stages `audit` und `test`. Aktiv ist der Job `audit`, der die Abhängigkeiten mit
`--frozen-lockfile --recursive` installiert und anschließend `pnpm audit --prod` ausführt. Als Image dient das
projekteigene Dev-Image aus der GitLab-Registry der Uni Jena; der pnpm-Store und `node_modules` werden je Branch
gecacht.

---

## Inhaltliche Dokumentation

### Ausgangslage und Zielsetzung

Zu Beginn der Entwicklung der Prototypen wurden der Ablauf von VERA und die beteiligten Akteur:innen aus Schulsicht systematisch betrachtet. Dabei wurde untersucht, welchen Nutzen die bestehenden Rückmeldungen bieten und an welchen Stellen sich ihr Mehrwert weiter erhöhen lässt.

Dabei haben sich mehrere Ansatzpunkte herauskristallisiert, die im Folgenden erläutert werden. Sie bilden die Grundlage für das Verständnis der entwickelten Prototypen und der ihnen zugrunde liegenden Entscheidungen.

Aus Sicht der Lehrkräfte sind Rückmeldungen ein wichtiger Baustein, um den Nutzen von VERA zu erhöhen. Sie bieten eine Orientierung für die pädagogische Arbeit und können Impulse für die Weiterentwicklung des Unterrichts geben. Entscheidend ist dabei, dass die Ergebnisse möglichst zeitnah vorliegen, die Informationen auf die jeweiligen Adressat:innen zugeschnitten sind und konkrete Anhaltspunkte für die weitere Arbeit bieten.

Darüber hinaus besteht der Wunsch, die Ergebnisse aus VERA mit denen anderer diagnostischer Verfahren zu verknüpfen. Auch Längsschnittbetrachtungen wären hilfreich, sowohl auf Ebene der einzelnen Schüler:innen als auch auf Ebene der Lehrkräfte und Schulen.

Im Rahmen des Projekts wurden zunächst vielfältige Ideen gesammelt. Anschließend lag der Schwerpunkt auf der konkreten Entwicklung von Rückmeldungen, die sich innerhalb der bestehenden Strukturen umsetzen lassen. Dabei wurden zwei zentrale Adressat:innengruppen identifiziert: Schüler:innen und Lehrkräfte.

Für diese beiden Gruppen wurden jeweils eigene Prototypen entwickelt (s. [Schüler:innenbericht](https://github.com/kompetenztestde/kompetenzstand-mathematik-schuelerinnenbericht) und [Klassenbericht](https://github.com/kompetenztestde/kompetenzstand-mathematik-klassenbericht)). Sie berücksichtigen die unterschiedlichen Informationsbedürfnisse und können unabhängig voneinander eingesetzt werden. Gleichzeitig verbindet sie ein gemeinsames visuelles Gestaltungskonzept.

### Vorüberlegungen

Aus Sicht der Entwicklungsgruppe lässt sich der Nutzen von VERA erhöhen, wenn Schüler:innen zeitnah eine Rückmeldung zu ihren Ergebnissen erhalten. Davon können auch ihre Erziehungsberechtigten profitieren, indem sie einen Einblick in die Testergebnisse und die bereits erworbenen Kompetenzen erhalten.

Gleichzeitig ist zu berücksichtigen, dass VERA nicht für eine Individualdiagnostik konzipiert ist. Die Ergebnisse erlauben daher keine uneingeschränkten Aussagen über die individuellen Fähigkeiten einzelner Schüler:innen. Dieser Einschränkung wurde bei der Entwicklung des Prototyps Rechnung getragen. Sie zeigt sich insbesondere in der vorsichtigen Formulierung der Rückmeldungen, der Auswahl der dargestellten Informationen und der Einordnung der Ergebnisse.

### Berechnung und Einordnung der Ergebnisse

Die Berechnung und Darstellung der Ergebnisse bringt verschiedene Herausforderungen mit sich, bietet zugleich aber Möglichkeiten für individuelle Rückmeldungen.

Da die VERA-Tests nicht für die Individualdiagnostik entwickelt wurden, ist bei der Interpretation der Ergebnisse besondere Zurückhaltung geboten. Entsprechend vorsichtig sind die Formulierungen im Bericht gewählt. Rückmeldungen auf Ebene einzelner Teilbereiche werden nur ausgegeben, wenn der jeweilige Teilbereich mindestens fünf Items umfasst.

Ein weiteres Ziel ist es, den Bericht möglichst zeitnah nach Abschluss der Korrektur bereitzustellen. So können sich die Schüler:innen noch gut an die Testsituation und die bearbeiteten Aufgaben erinnern. Um dies zu ermöglichen, wird im Schüler:innenbericht bewusst auf faire Vergleichswerte und Landesmittelwerte verzichtet. Auch Vergleiche auf Schul- oder Klassenebene sind nicht vorgesehen. Solche Vergleichswerte können zu Fehlinterpretationen führen, da ihre Aussagekraft unter anderem von der Leistungszusammensetzung der jeweiligen Vergleichsgruppe abhängt.

Eine reine Darstellung der Testergebnisse ohne Einordnung in den VERA-Kontext wäre jedoch ebenfalls wenig hilfreich und könnte zu Missverständnissen führen. Da die Schüler:innen den Bericht möglichst selbstständig verstehen sollen, benötigen sie einen verständlichen Orientierungsrahmen. Deshalb werden die Ergebnisse in Beziehung zu den Pilotierungswerten und den Kompetenzstufen gesetzt.

Ergänzend zu den eigentlichen Testergebnissen werden Informationen zur Durchführung des Online-Tests berücksichtigt. Dadurch lassen sich auch Auffälligkeiten im Bearbeitungsverhalten in die Rückmeldung einbeziehen.

### Zugriff auf den Bericht

Der Bericht steht den Schüler:innen zur Verfügung, sobald die Lehrkraft die Korrektur abgeschlossen hat.

Ursprünglich war vorgesehen, den Zugang über die Codekarten des Online-Tests zu ermöglichen. Dafür sollten die Schüler:innen entweder einen QR-Code scannen oder ihre Zugangsdaten verwenden können. In der aktuellen Umsetzung erfolgt der Zugriff über eine separate Login-Seite.

Der Bericht ist sowohl auf Computern als auch auf mobilen Endgeräten abrufbar. Damit kann er flexibel genutzt werden und ist für die Zielgruppe möglichst niedrigschwellig zugänglich.

### Aufbau und Gestaltung des Berichts

Ziel des Schüler:innenberichts ist es, zeitnah eine verständliche Rückmeldung zu den Testergebnissen zu geben und die Schüler:innen dabei zu unterstützen, ihre Ergebnisse einzuordnen.

Der Bericht ist als Abfolge einzelner Folien gestaltet. Er führt vom Gesamtergebnis schrittweise zu detaillierteren Informationen. So erhalten die Schüler:innen zunächst einen Überblick und können sich anschließend gezielt mit einzelnen Aspekten ihrer Ergebnisse beschäftigen. Die Texte sind bewusst kurz und verständlich formuliert.

Das farbenfrohe Design greift visuelle Elemente aus dem Themenfeld Mathematik auf. Die wichtigsten Informationen stehen jeweils im Vordergrund. Ergänzende Erläuterungen sind über Informationsicons abrufbar. Darüber hinaus bietet jede Seite Zugang zu allgemeinen Informationen über VERA und die Rückmeldung.

Der Bericht ist dynamisch aufgebaut: Abhängig von den individuellen Ergebnissen unterscheiden sich die Seiten nicht nur hinsichtlich ihrer Inhalte und Gestaltung, sondern auch darin, welche Seiten überhaupt angezeigt werden. Dadurch entsteht ein auf die jeweilige Ergebniskonstellation abgestimmter Bericht.

Allen Varianten gemeinsam ist eine positive und motivierende Darstellung der Leistungen. Der Bericht soll nicht nur Ergebnisse vermitteln, sondern auch sichtbar machen, was die Schüler:innen bereits gelernt haben und woran sie weiterarbeiten können.

### Gesamtergebnis und erste Einordnung

Zu Beginn des Berichts erhalten die Schüler:innen eine Rückmeldung zu ihrem Gesamtergebnis.

Ein spielerisches Element unterstützt den Einstieg: Ein fröhliches Pluszeichen taucht in stilisiertem Wasser ein, legt eine Strecke zurück und erscheint anschließend wieder an der Oberfläche. Die Animation soll Neugier wecken und dazu motivieren, den Bericht weiter anzuschauen. Das Pluszeichen stellt einen Bezug zur Mathematik her und vermittelt zugleich eine positive Botschaft. Das Wasser symbolisiert den bereits zurückgelegten Lernweg.

Beim Auftauchen erläutert das Pluszeichen das Testergebnis. Dabei wird bewusst auf die Darstellung konkreter Zahlenwerte verzichtet. Im Vordergrund stehen die Einordnung des Ergebnisses und die bereits erworbenen Kompetenzen.

Die Position des Pluszeichens bildet die prozentuale Lösungshäufigkeit im Test ab. Das Wasser dient als Orientierungsskala und ist in drei Bereiche unterteilt. Diese orientieren sich an den Kompetenzstufen, ohne sie explizit zu benennen:

* **Unterer Bereich:** Kompetenzstufen Ia und Ib.
* **Erwartungsbereich:** Kompetenzstufen II und III bei Schüler:innen an nichtgymnasialen Schulformen beziehungsweise Kompetenzstufe II bei Schüler:innen an gymnasialen Schulformen.
* **Optimalbereich:** Kompetenzstufen III bis V bei Schüler:innen an nichtgymnasialen Schulformen beziehungsweise Kompetenzstufen IV und V bei Schüler:innen an gymnasialen Schulformen.

Durch diese Darstellung erhalten die Schüler:innen eine erste Orientierung, ohne dass einzelne Zahlenwerte oder formale Kompetenzstufen im Mittelpunkt stehen.

### Auffällige Ergebnisse

Wenn bei der Testbearbeitung bestimmte Auffälligkeiten festgestellt werden, wird eine zusätzliche Seite mit einer entsprechenden Rückmeldung eingeblendet. Sie soll helfen, besondere Ergebniskonstellationen einzuordnen, ohne daraus vorschnell Schlussfolgerungen über die individuellen Fähigkeiten abzuleiten.

Folgende Situationen können berücksichtigt werden:

* Alle oder nahezu alle Aufgaben wurden richtig gelöst.
* Nur wenige oder keine Aufgaben wurden richtig gelöst.
* In allen Teilbereichen wurde ein Ergebnis im erwarteten Bereich erzielt.
* In mehreren Leitideen fiel das Ergebnis niedriger aus als erwartet.
* Der Test wurde sehr schnell bearbeitet, wobei viele Fehler auftraten.
* Es wurden nur wenige Aufgaben bearbeitet, von denen jedoch ein großer Anteil richtig gelöst wurde.

Wenn eine der definierten Bedingungen erfüllt ist, erhalten die Schüler:innen eine kurze, verständlich formulierte Information zur jeweiligen Auffälligkeit. Je nach Situation wird die Rückmeldung durch einen Hinweis oder einen Tipp für die weitere Arbeit ergänzt.

### Stärkeseite

Jede:r Schüler:in erhält eine Rückmeldung zu einer individuellen Stärke im Fach Mathematik. Dahinter steht die Annahme, dass jeder Mensch Stärken mitbringt, die durch eine gezielte Rückmeldung sichtbar gemacht werden können.

Als Stärke wird ein rückmeldbarer Teilbereich ausgewählt. Im Fach Mathematik kann dies beispielsweise eine prozessbezogene Kompetenz oder eine Leitidee sein. Ein kurzer Text benennt die Stärke und erläutert, worin sie besteht.

Ein Pokal und Konfetti unterstreichen die positive Botschaft. Über ein Informationsicon können die Schüler:innen zusätzliche Erklärungen zum jeweiligen Bereich aufrufen. Die kindgerechte Formulierung und die verständlichen Erläuterungen sind wichtige Bestandteile, damit die Rückmeldung nachvollziehbar bleibt.

Die Stärkeseite setzt damit einen bewussten Akzent: Neben der Einordnung des Gesamtergebnisses werden auch die Bereiche hervorgehoben, in denen die Schüler:innen bereits über Kompetenzen verfügen.

### Rückmeldung zu den Leitideen

Die detaillierten Rückmeldungen beziehen sich auf die mathematischen Leitideen. Auf weitere Ausdifferenzierungen wird im Schüler:innenbericht bewusst verzichtet. Leitideen sind ein zentraler Bestandteil des Mathematikunterrichts und bieten eine verständliche Orientierung dafür, welche Inhalte bereits sicher beherrscht werden und in welchen Bereichen weiteres Lernen sinnvoll sein kann.

Die Leitideen werden absteigend nach dem jeweiligen Testergebnis angeordnet. So sehen die Schüler:innen zunächst die Bereiche, in denen sie vergleichsweise gut abgeschnitten haben.

Die Einordnung erfolgt erneut über eine dreigeteilte Skala. Ein Smiley visualisiert das Ergebnis, während ein kurzer Text erläutert, wie es im Gesamtzusammenhang zu verstehen ist. Auch hier wird bewusst auf die Angabe konkreter Zahlenwerte verzichtet.

Zusätzlich werden die jeweilige Leitidee benannt und durch eine inhaltlich passende Grafik veranschaulicht. Ein Informationsicon bietet eine ergänzende Erklärung.

Wenn die Ergebnisse in drei oder mehr Teilbereichen niedriger ausfallen als erwartet, werden diese gemeinsam auf einer Seite dargestellt. Dadurch soll vermieden werden, dass eine Vielzahl einzelner negativer Rückmeldungen die Wahrnehmung des Berichts dominiert und die Motivation der Schüler:innen beeinträchtigt.

### Abschlussseite

Die Abschlussseite soll das Interesse an Mathematik aufgreifen und den Bericht mit einem positiven Impuls beenden. Dazu werden drei mathematische Rätsel oder Spiele vorgestellt, die über entsprechende Links erreichbar sind.

Die Aufgaben bieten eine Möglichkeit, sich über die reine Ergebnisrückmeldung hinaus mit mathematischen Inhalten zu beschäftigen. Der spielerische Abschluss soll dazu beitragen, Mathematik als anregendes Lernfeld erfahrbar zu machen.

### Mögliche Weiterentwicklungen

Bei der Konzeption des Schüler:innenberichts wurden weitere Ideen entwickelt, die im Rahmen des Prototyps aus Zeit- und Ressourcengründen nicht umgesetzt werden konnten.

Ein Ansatzpunkt ist die stärkere Verknüpfung der Abschlussseite mit den individuellen Testergebnissen. Denkbar wäre beispielsweise, für jede Leitidee ein leichteres und ein anspruchsvolleres Spiel anzubieten und abhängig vom Ergebnis passende Empfehlungen anzuzeigen. Auch die Verknüpfung mit konkreten Aufgaben zur Weiterarbeit könnte den Nutzen des Berichts erhöhen.

Eine weitere Idee war, die Schüler:innen während der Testbearbeitung um eine Selbsteinschätzung zu bitten. Durch den Abgleich dieser Einschätzung mit den tatsächlichen Testergebnissen könnten zusätzliche Anhaltspunkte entstehen, die eine Auseinandersetzung mit den eigenen Fähigkeiten unterstützen.

### Ausblick

Beide Prototypen (s. [Schüler:innenbericht](https://github.com/kompetenztestde/kompetenzstand-mathematik-schuelerinnenbericht) und [Klassenbericht](https://github.com/kompetenztestde/kompetenzstand-mathematik-klassenbericht)) wurden für den Einsatz im Fach Mathematik in Jahrgangsstufe 8 konzipiert. Eine Übertragung auf andere Fächer oder Jahrgangsstufen ist grundsätzlich denkbar, erfordert jedoch eine fachliche Prüfung und entsprechende Anpassungen. Dazu gehören insbesondere die inhaltliche Abstimmung der Rückmeldungen, die Auswahl geeigneter Vergleichswerte, die Überprüfung der Berechnungsgrundlagen sowie gegebenenfalls Anpassungen des Designs.

Die entwickelten Prototypen zeigen, wie Rückmeldungen aus VERA stärker an den Informationsbedürfnissen der jeweiligen Zielgruppen ausgerichtet werden können. Der Schüler:innenbericht stellt eine verständliche und motivierende Einordnung der individuellen Testergebnisse in den Mittelpunkt. Der Klassenbericht bietet Lehrkräften differenzierte Auswertungsmöglichkeiten und Ansatzpunkte für die weitere Unterrichtsentwicklung.

Beide Ansätze verfolgen damit das Ziel, Testergebnisse nicht nur bereitzustellen, sondern ihre Einordnung und Nutzung zu erleichtern. Zugleich bilden die Prototypen eine Grundlage für weitere Entwicklungen, insbesondere für die Verknüpfung mit bestehenden diagnostischen Angeboten und konkreten Möglichkeiten zur Weiterarbeit.

Für die nächsten Schritte empfehlen wir, beide Prototypen unter realen Einsatzbedingungen zu erproben und systematisch zu evaluieren. Dabei sollte insbesondere untersucht werden, wie verständlich die Rückmeldungen für die jeweiligen Zielgruppen sind, wie sie im schulischen Alltag genutzt werden und welche Funktionen den größten Mehrwert bieten. Die Ergebnisse einer solchen Evaluation können dazu beitragen, die Prototypen gezielt weiterzuentwickeln und ihre Eignung für einen breiteren Einsatz zu prüfen.

## Lizenz

Der **Quellcode** steht unter der [MIT-Lizenz](LICENSE), die **Animationen und Illustrationen** unter
[CC BY-SA 4.0](LICENSE-CC-BY-SA-4.0.txt). Welche Datei unter welcher Lizenz steht, listet
[`LICENSES.md`](LICENSES.md) auf – dort ist auch das Firmenlogo als Marke von beiden Lizenzen ausgenommen.

Die eingebundene Schriftart League Spartan steht unter der SIL Open Font License 1.1
(`src/themes/fonts/OFL.txt`). Die Lizenztexte aller gebündelten npm-Pakete erzeugt der Build in
`dist/THIRD-PARTY-NOTICES.txt`.
