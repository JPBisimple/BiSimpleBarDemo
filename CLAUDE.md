# Rezypedia (BiSimpleBarDemo)

Bar-/cocktail-driftssystem: opskrifter, prep, lager, events og menuer for
barer/restauranter.

**Brandnavn:** Rezypedia, skrives `rezy·pedia` (lille bogstav, midterprik
U+00B7). Domænet `rezypedia.com` skrives uden prik. Omdøbt fra det
tidligere navn "Backbar" — beslutningen er endelig og skal ikke
genåbnes; baggrunden er trademark-risiko over for konkurrenten
getbackbar.com. Gamle referencer til "Backbar" i filnavne
(`backbar_app_prompt.md`, `backbar_seed.json`) og i manualerne er
efterladte rester af det gamle navn, ikke et aktivt navn.

> Afsnit markeret `<!-- UDFYLD -->` er stadig ubekræftet. Alt andet er
> enten verificeret direkte mod `index.html`, eller kommer fra
> `PROGRESS.md` i det private repo [JPBisimple/Rezypedia-internal](https://github.com/JPBisimple/Rezypedia-internal)
> — se "Filer uden for repoet" nederst.

---

## Stak

- Statisk HTML/CSS/JS i én fil (`index.html`), ingen byggeproces, intet framework
- **Backend:** Supabase (Postgres + Auth + Storage), region Frankfurt
  (`eu-central-1`), projekt `gnhmnsstchpmjibujthb.supabase.co`
- **Hosting:** GitHub Pages, repo `JPBisimple/BiSimpleBarDemo`, branch `main`
  (`CNAME` → rezypedia.com). DNS hos DanDomain, HTTPS enforced.
- Ingen `config.js` — Supabase-URL og publishable-nøgle står hardkodet
  direkte i `index.html` (`SUPABASE_CONFIG`). Ikke et brud i sig selv
  (nøglen er `sb_publishable_...`, beregnet til at være offentlig).
- **Workflow er nu lokal git/CLI**, ikke længere kun GitHub web UI som
  `PROGRESS.md` beskriver — den fil er forældet på dette punkt.
  Ændringer laves lokalt og pushes til `main`.
- `renderSetup()` er en død kodesti fra den oprindelige skabelon — den
  henviser til `schema.sql`/`seed.sql`, som ikke ligger i dette repo, men
  i [JPBisimple/Rezypedia-internal](https://github.com/JPBisimple/Rezypedia-internal) (privat, se nederst).

## Sikkerhedsmodel

**Klienten gør kun UI-gating — den reelle spærre ligger i RLS.**

`currentRole() === 'admin'` styrer udelukkende, om admin-knapper/-sider
*vises*. De bagvedliggende funktioner (`openCocktailEditor`,
`openIngredientEditor`, `openPrepEditor`, `openEventEditor`,
`openGroupEditor`, `openMenuEditor`, alle delete-handlers,
`openTenantSettings`, m.fl.) har ingen internt rolletjek — de regner med,
at RLS stopper det, UI'et ikke gør.

**RLS-mønsteret er bekræftet** og er ens for alle tabeller:

```sql
CREATE POLICY <tabel>_select ON <tabel>
  FOR SELECT USING (is_tenant_member(tenant_id));

CREATE POLICY <tabel>_admin_write ON <tabel>
  FOR ALL USING (is_tenant_admin(tenant_id)) WITH CHECK (is_tenant_admin(tenant_id));
```

Hjælpefunktioner i databasen: `is_tenant_member(uuid)`,
`is_tenant_admin(uuid)`, `get_user_tenants()`. Nye tabeller skal *selv*
have `GRANT SELECT, INSERT, UPDATE, DELETE ... TO authenticated` —
RLS-policies alene er ikke nok (samme faldgrube som i HKOEDBooking:
manglende grant fejler med samme fejlkode som en policy-afvisning, og de
kan ikke skelnes fra klienten).

**Bekræftet 2026-09-18 via en frisk `pg_policies`-forespørgsel:**
policies findes for alle 6 nye tabeller
(`menu_groups`/`menus`/`menu_cocktails`/`events`/`event_cocktails`/
`event_prep_responsibility`), efter samme select+admin_write-mønster som
de oprindelige tabeller. De kørte oprindeligt på rolle `public` i stedet
for `authenticated` som resten af skemaet — **rettet samme dag** med
`alter policy ... to authenticated` direkte i databasen, og bekræftet
med en ny `pg_policies`-forespørgsel (alle 12 policies står nu til
`{authenticated}`).

**Grants bekræftet 2026-09-18** via en frisk
`information_schema.role_table_grants`-forespørgsel: `authenticated` har
`SELECT, INSERT, UPDATE, DELETE` på alle 6 nye tabeller. `anon` har
(som forventet, Supabase-standard ved tabelloprettelse — samme mønster
som i HKOEDBooking) også fuldt grant inkl. `DELETE`/`TRUNCATE`, men har
**ingen matchende RLS-policy** (alle policies er nu `to authenticated`),
så `anon` er reelt blokeret uanset det brede grant.

**Tenant-isolation på skrivninger er dobbelt sikret:** klienten stempler
selv `tenant_id` ved insert, men `is_tenant_admin(tenant_id)`/
`is_tenant_member(tenant_id)` i RLS tjekker uafhængigt medlemskab i
`memberships`. Composite foreign keys (`UNIQUE (id, tenant_id)` på
`cocktails`, `preps`, `menu_groups`, `menus`, `events`) bruges
gennemgående til at forhindre krydstenant-reference på FK-niveau.

**Ingen `esc()`-lignende hjælpefunktion findes eller er nødvendig:**
DOM-hjælperen `el()` sætter tekstbørn via `document.createTextNode`, så
brugerdata escapes automatisk. Attributten `html:` (sætter `innerHTML`)
bruges kun til statisk brand-markup ("rezy·pedia") — aldrig med bruger-
eller DB-data. Hold det sådan.

### ✅ Rettet 2026-09-18 — cost/margin var synlig for ikke-admin

`renderCocktailDetail` viste kostpris-kortet (kost/profit/margin) uden
rolletjek, og margin-/⚠-chippen i cocktail-listen (`renderCocktails`)
havde samme problem. Begge steder er nu gatet bag `isAdmin` — kun
salgsprisen (`sell_price`, kundens pris, ikke fortrolig) vises stadig
for ikke-admins, flyttet ind i cocktail-detaljens hero-sektion i stedet
for i cost-kortet. **Bemærk:** dette er kun rettet i klienten
(`index.html`) — der er ingen RLS-mekanisme til at skjule enkelte
kolonner i en række, så en teknisk bruger, der selv kalder
`state.sb.from('cocktails').select('sell_price,...')`, kan stadig se
`sell_price` (aldrig kost/margin, da de udregnes klient-side af
`cocktailCost()`, ikke gemt i databasen som en kolonne).

### Andre kendte svagheder

- Ingen rolletjek i selve mutation-funktionerne i klienten, kun i
  render-laget — hold dig til samme UI-gating + RLS-mønster for nye
  admin-only handlinger. **Undtagelse:** `ingredients` har rent faktisk
  en server-side trigger (`guard_ingredient_update` i `schema.sql`), der
  forhindrer en ikke-admin i at ændre andet end `current_stock`, selvom
  RLS-policyen tillader opdatering for enhver tenant-medlem. Det er et
  reelt, håndhævet værn — ikke kun UI-gating — og et mønster, der er
  værd at genbruge, hvis en fremtidig tabel har samme behov (en kolonne
  alle må ændre, resten kun admin).
- Profiler er ikke tenant-afgrænsede: `profiles_select`-policyen i
  `schema.sql` er `using (true)` for enhver `authenticated` bruger —
  enhver indlogget bruger kan altså se navn/mail på enhver anden bruger
  i systemet, uanset tenant. Bevidst (til at vise navne på tværs), men
  værd at kende, hvis `profiles` nogensinde bruges til mere end det.
- `SUPABASE_CONFIG` hardkodet i `index.html` i stedet for separat
  `config.js` — gør det sværere at have adskilte dev/prod-miljøer.
- `FEATURES.orders = false` slår hele Orders-sektionen fra, koden er
  bevaret men ikke vedligeholdt/testet, mens flaget er falsk.
- **Supabase Auth URL Configuration er ikke testet med en reel
  logout+login.** `https://rezypedia.com` skal stå som Site URL og i
  Redirect URLs — en eksisterende session i browseren beviser ingenting.

---

## Multi-tenant

Én installation betjener flere barer ("tenants"). En bruger kan være
medlem af flere tenants via `memberships` og skifter mellem dem i en
dropdown i topbaren. Aktiv tenant gemmes i
`localStorage['activeTenantId']`. Alle tenant-ejede tabeller filtreres på
`tenant_id`, håndhævet af RLS (se ovenfor), ikke kun af klientkoden.

### Tenants

| Tenant | UUID | By | Tier | Status |
|---|---|---|---|---|
| Razzia | `11111111-1111-1111-1111-111111111111` | Zürich | paid | 60 ingredienser, 12 preps, 8 cocktails. Én menu ("Main Menu"), alle 8 cocktails tilknyttet |
| Paradiso | `22222222-2222-2222-2222-222222222222` | Barcelona | free_pilot | 5 grupper / 10 menuer oprettet. **Cocktails er endnu ikke lagt på menuerne.** Ingredienser mangler indkøbspriser |
| Aldea | — | — | — | **Ikke oprettet endnu.** Tænkt som simpel single-menu demo ved siden af Paradisos komplekse opsætning |

En kodekommentar i `index.html` nævner "Razzia, Aldea" sammen — det er
en fremtidig/planlagt reference, ikke et tegn på at Aldea findes i
databasen endnu.

Paradisos menustruktur (oprettet, cocktails ikke tilknyttet endnu):
```
Main              → Oltre, Paradiso Classics
Terrace           → Terrace
Macallan          → Macallan
Previous          → Evolution of the Humankind, Universo, Illusionist Menu,
                     Mediterranean Menu, Twist on Classics
Classic Cocktails → Classic Cocktails
```
"Previous" er tænkt som arkiv — planen er at flytte de fem menuer til
Main og sætte dem `is_active = false`, men **det er ikke gjort endnu.**

### Roller

Kun `'admin'` tjekkes eksplicit i klientkoden (`currentRole()`, læst fra
`memberships.role` for aktiv tenant). **Bekræftet fra `schema.sql`:**
`memberships.role` har `check (role in ('admin','bartender'))` — der
findes kun disse to roller. En `'bartender'` får ingen særlig
klient-adfærd (klienten tjekker kun eksplicit for `'admin'`), men er
den, der reelt bruges til alle ikke-admin-medlemmer.

### Login

Kun e-mail + password (`signInWithPassword`). Ingen selvregistrering —
brugere oprettes manuelt af admin i Supabase.

---

## Datamodel (verificeret Supabase-skema, september 2026)

### Oprindelige tabeller
```
tenants(id, slug, name, city, subscription_status, subscription_tier, theme jsonb, created_at, currency, locale)
profiles(id, email, full_name, created_at)
memberships(id, user_id, tenant_id, role, created_at)
ingredients(id, tenant_id, name, category, supplier, purchase_size, purchase_unit, purchase_price, current_stock, min_stock, barcode, image_url, created_at)
preps(id, tenant_id, name, type, yield_quantity, yield_unit, procedure, shelf_life, created_at, event_id)
prep_components(id, prep_id, tenant_id, ingredient_id, child_prep_id, quantity, unit)
cocktails(id, tenant_id, name, menu_wording, flavour_profile, allergens, glassware, ice, garnish, tools, procedure, alcohol_percent, sell_price, is_active, image_url, created_at, event_id)
cocktail_components(id, cocktail_id, tenant_id, ingredient_id, prep_id, quantity, unit)
```

`profiles` findes i skemaet, men klienten (`index.html`) kalder aldrig
`.from('profiles')` direkte — brugerens navn/mail kommer udelukkende fra
Supabase Auth-sessionen.

### Menu-tabeller (tilføjet september 2026)
```
menu_groups(id, tenant_id, name, sort_order, is_active, created_at)
menus(id, tenant_id, group_id, name, sort_order, is_active, created_at)
menu_cocktails(id, tenant_id, menu_id, cocktail_id, sort_order, created_at)
```

### Event-tabeller (tilføjet september 2026)
```
events(id, tenant_id, name, event_date, customer, guest_count, notes, is_active, created_at)
event_cocktails(id, tenant_id, event_id, cocktail_id, quantity, sort_order, created_at)
event_prep_responsibility(id, tenant_id, event_id, prep_id, responsible, created_at)
```
`responsible` er `'bar'` eller `'host'`, håndhævet med CHECK-constraint
i databasen (ikke kun konvention i klienten).

### Vigtige faldgruber i skemaet

- Kolonnen hedder **`purchase_price`** — ikke `purchase_price_dkk`.
  Et ældre `backbar_seed.json` (ikke en del af `Rezypedia-internal`)
  bruger stadig det gamle kolonnenavn — **kør det ikke direkte**, det
  vil fejle. `seed.sql` i `Rezypedia-internal` er den gyldige, opdaterede version.
- **Navneunikhed er ikke ét simpelt constraint.** De gamle
  `cocktails_tenant_id_name_key` / `preps_tenant_id_name_key` er droppet
  og erstattet af partielle unikke indekser, så en eventkopi må hedde
  det samme som originalen:
  ```
  cocktails_name_normal_uq  ON cocktails (tenant_id, name)            WHERE event_id IS NULL
  cocktails_name_event_uq   ON cocktails (tenant_id, event_id, name)  WHERE event_id IS NOT NULL
  preps_name_normal_uq      ON preps     (tenant_id, name)            WHERE event_id IS NULL
  preps_name_event_uq       ON preps     (tenant_id, event_id, name)  WHERE event_id IS NOT NULL
  ```
  **Konsekvens:** `ON CONFLICT (tenant_id, name)` virker ikke længere på
  `cocktails`/`preps` — skal være
  `ON CONFLICT (tenant_id, name) WHERE event_id IS NULL`. Rammer kun
  manuelle SQL/seed-scripts, ikke selve appen. `menus` har tilsvarende to
  partielle indekser, så samme menunavn må gå igen i forskellige grupper.
- Schema-introspektion i Supabase:
  `information_schema.columns WHERE table_schema = 'public' ORDER BY table_name, ordinal_position`
  — `pg_get_ddl` findes ikke i Supabase.

### Events — kopimodel (vigtigt designvalg)

En cocktail lagt på et event **kopieres** ind sammen med sine
komponenter — originalen i biblioteket røres aldrig, og der er **ingen
levende reference** tilbage til den. Det er et bekræftet designvalg, ikke
en begrænsning der skal rettes. `normalCocktails()`/`normalPreps()`
filtrerer biblioteket til kun at vise rækker uden `event_id`.

Fast værdisæt for `preps.type` i klienten (ikke en DB-enum, men en
hardkodet JS-liste, brugt til gruppering i UI'et):
`PREMIX`, `BATCH`, `SYRUP`, `INFUSION`, `CORDIAL` (+ "Other"-bucket for
ukendte typer).

---

## App-sektioner og status

| Sektion | Hash | Adgang | Status |
|---|---|---|---|
| Home (hub) | `#index` | Alle | ✅ |
| Cocktails | `#cocktails` | Admin CRUD, bartender read | ✅ |
| Prep | `#prep` | Admin CRUD, bartender read | ✅ |
| Inventory | `#inventory` | Admin CRUD, bartender stock | ✅ |
| Service | `#guest` | Alle | ✅ |
| Menus | `#menus` | Admin only | ✅ |
| Events | `#events`, `#event/<id>` | Alle ser, admin redigerer | ✅ |
| Event-menu (print) | `#eventmenu/<id>` | Alle | ✅ |
| Værtsdokument (print) | `#eventhost/<id>` | Alle | ✅ |
| Orders | `#orders` | — | 🔕 Slukket bag `FEATURES.orders = false`, kode bevaret |

**Events var ikke en del af det oprindelige change request fra
medstifterne** — det er en tilføjelse, og bør vendes med dem hvis scope
er et tema.

---

## Forretningslogik

### Cocktails, preps og komponenter

- En prep-komponent peger på enten en `ingredient_id` eller en
  `child_prep_id` (preps kan indeholde andre preps, rekursivt).
  Selvreference opdages og springes over med en advarsel.
- **Bevidst begrænsning, ikke en fejl: ingen enhedsomregning.** Gram og
  milliliter lægges sammen som rene tal, både i kostprisberegning og i
  events' pakkelister. Markeres med `⚠` når enheder blandes i samme
  post. Regn ikke dette om stiltiende — flere steder i koden regner
  bevidst med den nuværende (simple) opførsel.
- **Værktøjsliste kan ikke samles automatisk.** `tools` er fritekst pr.
  cocktail, så pakkelisten viser værktøj pr. cocktail, ikke en samlet
  liste.
- Kategori/enhed på ingredienser (`category`, `purchase_unit`, `unit`,
  `yield_unit`) er fritekst — filtre i UI'et bygges dynamisk fra
  eksisterende værdier, ikke fra en fast liste.
- **Manglende priser vises som `—`, aldrig `0,00`.** Bevidst valg: et
  nul ville give en falsk 100 %-margin i et salgsmøde.

### Events

Ansvarsfordeling pr. prep (`event_prep_responsibility`: `'bar'` eller
`'host'`, default `'bar'`). En bar-prep udfoldes rekursivt ned i
råvarer; en host-prep stopper barens beregning og lægger hele
undertræet på værtens liste i stedet — værten leverer selv den
færdige prep.

**Tre adskilte udskrifter, bevidst adskilt:**
- **Arbejdsark** — med kostpris/margin, kun internt. UI advarer
  eksplicit mod at sende det til kunde/vært.
- **Cocktailmenu** (`#eventmenu/:id`) — gæstevendt, ingen priser/margin.
- **Værtsdokument** (`#eventhost/:id`) — indkøbsliste + opskrifter til
  værten, ingen priser/margin.

Enhver ny events-relateret visning skal respektere denne adskillelse.

### Menuer

`menu_groups` → `menus` → `menu_cocktails`. En cocktail kan ligge på
flere menuer samtidig. Kun admin kan redigere (`renderMenus()` er
admin-only, plus RLS `_admin_write`-policy).

### Lager (Inventory)

`current_stock`/`min_stock` på `ingredients`; rækker under minimum
markeres rødt. Én realtime-subscription i hele appen: `UPDATE` på
`ingredients`, filtreret på aktiv tenant. **De nye tabeller (menu/event)
opdaterer ikke live mellem brugere** — kun ingrediens-lager har
realtime.

### Orders (deaktiveret)

Styret af `FEATURES.orders = false`. To undertilstande i koden: Forecast
(indtast forventet salg → mangelliste pr. leverandør) og Stock
(mangelliste ud fra `current_stock < min_stock` alene). Ikke nået via
routeren, mens flaget er falsk — ikke vedligeholdt.

---

## Kodekonventioner

- Ingen `esc()`-helper nødvendig — `el()` escaper automatisk tekstbørn
  via `createTextNode`. Brug **aldrig** `html:`-attributten med data fra
  Supabase eller et inputfelt, kun med statisk markup.
- Beløb vises altid gennem `money()` (alias `dkk()`, bevaret for
  bagudkompatibilitet) — `Intl.NumberFormat` med tenantens
  `currency`/`locale` (fallback `DKK`/`da-DK`). Rene tal uden valuta
  gennem `num()`/`unitPrice()`. Manglende/NaN vises som `—`, aldrig `0`.
- Ingen datoformatering — `event_date` vises som rå streng fra
  `<input type="date">`.
- Routing er rent hash-baseret (`location.hash`, `renderRoute()`), ingen
  History API, ingen query-strings. Parametriserede ruter bruger
  `#prefix/id` (`#cocktail/:id`, `#event/:id`, osv.) — følg samme
  mønster for nye detaljevisninger.
- Kun én realtime-subscription i hele appen (ingrediens-lager) — tilføj
  ikke flere uden grund.
- Ingen afhængigheder ud over `supabase-js` fra CDN og Google Fonts.
  Foreslå ikke et framework, en bundler eller et npm-projekt.
- `FEATURES`-objektet er mønsteret for at slå ufærdige/pausede sektioner
  fra uden at slette koden — brug samme mønster fremover.
- Nye inline-formularer (fx "opret ingrediens" inde i en anden editor)
  skal være paneler, ikke `modal()` — `modal()` lukker den underliggende
  editor og ville smide brugerens ikke-gemte arbejde væk.

---

## Bevidst ikke bygget

- PDF-generering/hostet PDF-link i browseren — kræver et uverificeret
  bibliotek, en Supabase Storage-bucket med policies, og en beslutning om
  linkets levetid. Løst i stedet med "Gem som PDF" i browserens egen
  printdialog.
- Sortering af cocktails inden for en menu (alfabetisk i dag).
- R&D-dokumentbibliotek (`rd_findings`) — står i change request'et, ikke
  bygget.
- i18n / spansk UI — udskudt til efter feature freeze.
- Training-sektion, offline-support, OCR.
- Lager-advarsler i UI'et (kolonnerne findes i databasen, men bruges
  ikke endnu til proaktive advarsler).

---

## Deadline og risiko

Salgsmøder slutter oktober 2026. Succeskriterie er en defineret
"demo-ready"-tjekliste. **Scope creep er den centrale risiko** — Events
var ikke i det oprindelige change request. Hvis tiden bliver knap:
prioritér at lukke de åbne 🔴-punkter ovenfor først; et halvbygget
Events-modul, der fejler foran en kunde, er værre end slet intet Events.

---

## Filer uden for dette repo

Dette repo (`BiSimpleBarDemo`) er **offentligt** — det er derfor GitHub
Pages kan hoste det gratis. Interne dokumenter, der ikke må være
offentlige (kundenavne, deadlines, skema, seed-data), ligger derfor i et
**separat, privat** repo: [JPBisimple/Rezypedia-internal](https://github.com/JPBisimple/Rezypedia-internal).

| Fil | Ligger i | Beskrivelse |
|---|---|---|
| `PROGRESS.md` | `Rezypedia-internal` | Statusdokument — source of truth, dette CLAUDE.md er delvist bygget på det |
| `schema.sql` / `seed.sql` | `Rezypedia-internal` | Det gyldige, opdaterede skema/seed — nævnt af `renderSetup()`s onboarding-tekst i koden |
| `Manualer/manual_admin.html` + `manual_bartender.html` (+ `_es`-varianter, samt PDF-udgaver) | `Rezypedia-internal` | Kundemanualer, dansk (`lang="da"`) og spansk — dækker **ikke** Menus, Events eller de nye udskrifter endnu |
| `rezypedia_change_request.md` | Stadig kun i OneDrive (`__BackBar`) | Ændringsønsker fra medstiftere, juli 2026 — ikke flyttet endnu |
| `backbar_app_prompt.md` | Stadig kun i OneDrive | Oprindelig kravspecifikation (gammelt navn) — ikke flyttet endnu |
| `backbar_seed.json` | Stadig kun i OneDrive | **Forældet** — bruger det forkerte kolonnenavn `purchase_price_dkk`. Brug `seed.sql` i `Rezypedia-internal` i stedet |
| `Razzia_Cocktail_Index.xlsx` | Stadig kun i OneDrive | — |

Den aktuelle `index.html` kan altid hentes direkte fra det offentlige
repo uden upload: `https://raw.githubusercontent.com/JPBisimple/BiSimpleBarDemo/main/index.html`.

---

## Hvad der stadig mangler afklaring

Alle punkter fra den oprindelige liste er nu lukket. Ét mindre punkt
tilbage, opdaget undervejs:

1. **Composite FK-indekser (`UNIQUE(id, tenant_id)`) på `events` og
   `menu_groups`** — tilføjet i `schema.sql` efter samme mønster som
   `cocktails`/`preps`/`menus` (nu bekræftet), men ikke friskverificeret
   for lige netop disse to tabeller via `pg_indexes`.

~~Kolonnerne på `tenants`/`profiles` og de partielle unikke indekser på
`menus`~~ — bekræftet 2026-09-18.

~~Grants på de 6 nye tabeller~~ — bekræftet 2026-09-18, se
"Sikkerhedsmodel".

~~`to public` vs. `to authenticated` på de 6 nye tabellers RLS~~ —
rettet 2026-09-18, se "Sikkerhedsmodel".

~~🔴 Cost/margin-lækagen til ikke-admin~~ — rettet 2026-09-18, se
"Sikkerhedsmodel".
