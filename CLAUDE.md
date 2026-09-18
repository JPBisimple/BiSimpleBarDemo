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
> `PROGRESS.md` (statusdokument holdt uden for dette repo — se "Filer
> uden for repoet" nederst).

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
  `PROGRESS.md` (uden for repoet) beskriver — den fil er forældet på
  dette punkt. Ændringer laves lokalt og pushes til `main`.
- `renderSetup()` er en død kodesti fra den oprindelige skabelon — den
  henviser til `schema.sql`/`seed.sql`, som ikke ligger i dette repo, men
  findes som separate filer uden for git (se nederst).

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
kan ikke skelnes fra klienten). <!-- UDFYLD: er grant + begge policies bekræftet anvendt på menu_groups/menus/menu_cocktails/events/event_cocktails/event_prep_responsibility (tilføjet september 2026), eller kun på de oprindelige tabeller? -->

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

### 🔴 Kendt, uløst fejl — cost/margin synlig for ikke-admin

`renderCocktailDetail` viser kostpris-kortet **uden rolletjek**, og
margin-chippen i cocktail-listen har samme problem. Høj prioritet,
**ikke rettet endnu**. Ret dette, hvis du rører ved
`renderCocktailDetail`/`renderCocktails` — det er ikke en del af
opgaven i sig selv, men bør nævnes/tjekkes, hvis du alligevel er i det
område.

### Andre kendte svagheder

- Ingen rolletjek i selve mutation-funktionerne, kun i render-laget —
  hold dig til samme UI-gating + RLS-mønster for nye admin-only
  handlinger.
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
`memberships.role` for aktiv tenant). Andre rolleværdier er ikke
begrænset af klienten til et fast sæt, men hvilke der reelt bruges ud
over `'admin'`, er ikke bekræftet. <!-- UDFYLD: hvilke roller findes i memberships.role ud over 'admin' — er der en "bartender"/medarbejder-rolle? -->

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
  `backbar_seed.json` (uden for repoet) bruger stadig det gamle
  kolonnenavn — **kør det ikke direkte**, det vil fejle.
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
| Cocktails | `#cocktails` | Admin CRUD, bartender read | ✅ (men se 🔴 cost/margin-lækage ovenfor) |
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

## Filer uden for repoet

Disse ligger **ikke** i git, men i et separat Claude-projekt/OneDrive —
relevante at spørge efter, hvis en opgave kræver dem:

| Fil | Beskrivelse |
|---|---|
| `PROGRESS.md` | Statusdokument — source of truth, dette CLAUDE.md er delvist bygget på det |
| `rezypedia_change_request.md` | Ændringsønsker fra medstiftere, juli 2026 |
| `backbar_app_prompt.md` | Oprindelig kravspecifikation (gammelt navn) |
| `backbar_seed.json` | Seed-data — **bruger det forkerte kolonnenavn `purchase_price_dkk`**, kør ikke direkte |
| `manual_bartender.html` / `manual_admin.html` | Kundemanualer, dansk — dækker **ikke** Menus, Events eller de nye udskrifter endnu |
| `Razzia_Cocktail_Index.xlsx` | — |
| `schema.sql` / `seed.sql` | Nævnt af `renderSetup()`s onboarding-tekst i koden, men ikke fundet i repoet — ligger formentlig sammen med ovenstående |

Den aktuelle `index.html` kan altid hentes direkte fra det offentlige
repo uden upload: `https://raw.githubusercontent.com/JPBisimple/BiSimpleBarDemo/main/index.html`.

---

## Hvad der stadig mangler afklaring

1. **Grant + policies bekræftet på de nye tabeller** (menu/event-gruppen,
   tilføjet september 2026) — er de sat op efter samme mønster som de
   oprindelige tabeller?
2. **Roller ud over `'admin'`** i `memberships.role`.
3. **🔴 Cost/margin-lækagen til ikke-admin** (se "Sikkerhedsmodel") er
   bevidst ikke rettet endnu — vent med at røre den, indtil der bedes om det.
