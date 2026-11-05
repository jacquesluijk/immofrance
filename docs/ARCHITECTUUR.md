# ImmoFrance — Architectuurplan

> Doel: één overzichtssite voor **alle** gepubliceerde woningen in Frankrijk — van makelaars én particulieren — met zeer gedetailleerde filters, gericht op Nederlandse huizenkopers. Zeer plat, snel, mobiel-first, en maximaal vindbaar in Google en AI-zoekmachines.

---

## 1. Positionering en realiteitscheck

**Kernprobleem is data, geen technologie.** De site, filters en performance zijn bouwbaar. Het bijeenbrengen van "alles" is een dataverzamel-traject van maanden/jaren, geen programmeerklus.

Er bestaat in Frankrijk **geen officiële open bron** met alle aanbod. De realistische bronnenstrategie (paragraaf 3) bepaalt het groeipad: eerst een werkende site met schaalbare feed-infrastructuur, daarna stap voor stap dekking uitbreiden.

### 1.1 Doelgroep
- Nederlanders (en Vlamingen) die een huis in Frankrijk zoeken.
- Zoekgedrag: vertrekkend vanuit droomregio's (Dordogne, Provence, Bretagne, Ardèche…), budget, grond/oppervlakte, en praktische zaken als DPE-energielabel.

### 1.2 Kernwaarden van het product
1. **Volledigheid** — zo veel mogelijk echt aanbod, geen dode advertenties.
2. **Nauwkeurig filteren** — filters die de grote Franse portalen niet of slecht aanbieden (grondoppervlakte, DPE, type "fermette/mas/longère", afstand tot luchthaven/TGV).
3. **Snelheid** — pagina's < 1 seconde op mobiel (4G).
4. **Vindbaarheid** — Google én AI-assistenten (ChatGPT, Perplexity, Google AI Overviews).

---

## 2. Systeemoverzicht

```
┌──────────────────────────────────────────────────────────────────┐
│                          BRONNEN                                 │
│  XML/CSV-feeds (makelaarsoftware) · Handmatige invoer ·          │
│  Partner-overnames (later, evt. geselecteerde scrapes mits       │
│  juridisch getoetst)                                             │
└───────────────┬──────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────┐
│                     INGEST-PIPELINE (workers)                    │
│  1. Fetch        – feeds periodiek ophalen                       │
│  2. Normaliseren – naar één vast datamodel (§ 4.2)               │
│  3. Dedupliceren – op bronreferentie + fuzzy (adres/omschrijving) │
│  4. Verrijken   – geo (INSEE-code, departement, coördinaten),     │
│                   DPE-afleiding, afstanden (luchthaven/TGV)      │
│  5. Valideren   – pflichtvelden, prijs-plausibiliteit            │
└───────────────┬──────────────────────────────────────────────────┘
                │
      ┌─────────▼─────────┐         ┌────────────────────┐
      │  PostgreSQL       │◄────────│ Beheerinterface     │
      │  (bron van waar-  │         │ (feeds, makelaars,  │
      │   heid + media)   │         │ moderatie, stats)   │
      └─────────┬─────────┘         └────────────────────┘
                │
      ┌─────────▼─────────┐
      │  Meilisearch /    │  faceted search, filters op
      │  Typesense        │  milliseconden, typo-tolerantie
      └─────────┬─────────┘
                │
┌───────────────▼──────────────────────────────────────────────────┐
│                    WEBSITE (SSG + dunne API)                     │
│  Astro (of Next.js) · statische HTML · JSON-zoeklaag · PWA        │
└──────────────────────────────────────────────────────────────────┘
```

### 2.1 Keuzes
| Component | Keuze | Waarom |
|---|---|---|
| Frontend | **Astro** (SSG) | Nagenoeg nul JS naar de browser, eenvoudig, uitstekende Lighthouse-scores |
| Zoekindex | **Meilisearch** of **Typesense** | Facets/filters extreem snel, typo-tolerant, eenvoudiger dan Elasticsearch |
| Database | **PostgreSQL** | Bron van waarheid, relationeel, POSTGIS voor geo |
| Ingest | Node.js/TypeScript workers + cron | Feeds zijn XML/CSV; TypeScript hergebruik van datamodel met frontend |
| Hosting | Vercel/Cloudflare Pages (frontend) + kleine VPS of managed DB/search | Lage kosten, wereldwijde CDN |
| Media | Onze CDN-proxy met `width`-varianten (WebP/AVIF) | Snel op mobiel; origineel bij bron blijven staan i.v.m. auteursrecht |

---

## 3. Bronnenstrategie (het eigenlijke product)

### 3.1 Fase 1 — Feed-netwerk (legaal, schaalbaar)
Franse makelaars werken vrijwel allemaal met makelaarsoftware die XML/CSV-export ondersteunt:
- **Apimo, CustoFloos, EviWeb (SeLoger-groep), Proprioo, Uptimo, Folioh**
- Aanpak: software-aanbieders benaderen voor partnerfeeds (zij hebben er belang bij: extra zichtbaarheid voor hun makelaars), én individuele makelaars via een "plaats gratis uw aanbod"-programma.
- Voordeel: originele data, geen juridisch risico, directe relatie met de bron.

### 3.2 Fase 2 — Particulieren
- Eigen, gratis invoeromgeving voor particulieren (à la PAP, maar NL-talig).
- Later: partnerships met entre-particuliers-platforms.

### 3.3 Wat we NIET doen (i.v.m. Databankrichtlijn/ToS)
- Massaal schrapen van SeLoger, Leboncoin, LogicImmo, PAP en vergelijkbare portalen. Leidt tot blokkades, claims en een broos product.
- Eventuele latere scrapes alleen na juridische toets en selectief.

### 3.4 Kwaliteit boven kwantiteit
Elke listing krijgt een `source_status`; advertenties ouder dan X dagen zonder update worden automatisch gedeactiveerd (veel portalen staan vol dode advertenties — daar is onze kans).

---

## 4. Datamodel (kern)

### 4.1 Entiteiten
- **Listing** (het huis)
- **Broker / Seller** (agent of particulier)
- **Feed** (bron + configuratie + polling-interval)
- **Location** (INSEE-code, gemeente, departement, regio, coördinaten)
- **Media** (foto's, verwijzing naar CDN-varianten)

### 4.2 Listing — filterbare velden
| Veld | Type | Filterbaar |
|---|---|---|
| `price` | EUR, incl. honoraires-indicator | ✔ bereik |
| `property_type` | maison, villa, fermette, mas, longère, appartement, grond… | ✔ multi |
| `living_area_m2` | int | ✔ bereik |
| `land_area_m2` | int | ✔ bereik (kernfilter voor NL-kopers) |
| `bedrooms` / `rooms` | int | ✔ |
| `dpe_score` / `dpe_label` | A–G | ✔ (kernfilter) |
| `construction_year` | int | ✔ |
| `condition` | nieuw, gerenoveerd, te renoveren | ✔ |
| `location` | gemeente/departement/regio + geo | ✔ + geo-radius |
| `dist_airport_km`, `dist_tgv_km` | afgeroepd bij ingest | ✔ bereik |
| `features` | zwembad, schuur, haard, fibre… | ✔ multi |
| `description` | NL-vertaald (machine) + FR origineel | zoeken |
| `external_url` | link naar bron | — |

### 4.3 Deduplicatie
1. Harde key: `feed_id + external_ref`.
2. Vervolgens fuzzy: normalisatie van adres + prijs ± 2% + oppervlakte ± 5% → zelfde pand → samenvoegen, alle bronnen tonen ("aangeboden door 2 makelaars").

---

## 5. Frontend & UX (zeer plat, mobiel-first)

### 5.1 Structuur
```
/                                homepage + snelle zoekbalk
/huizen                          zoekresultaten (server-side gerenderd + client-filter)
/huizen/{slug}                   detailpagina (statisch per listing)
/regio/{departement}             landingpagina (Dordogne, Provence…)
/regio/{departement}/{type}      gecombineerde landingspagina
                     bijv. /regio/dordogne/fermette-met-grond
/gids                            kennisartikelen (aankoopproces, DPE, notaris…)
/makelaars                       gratis aanbod plaatsen (makelaar/particulier)
```

### 5.2 UI-principes
- **Mobiel eerst**: filterdrawer onderaan, sticky zoekbalk, grote tikdoelen, kaart/lijst-wissel.
- **Geen zware frameworks**: Astro met eenvoudige vanilla JS voor filterinteractie; resultaten via `fetch` op JSON-endpoint.
- **Resultaten per pagina**: 24; oneindig scrollen met fallback-paginering (SEO).
- **Snelheid**: statische HTML + CDN; foto's lazy, AVIF/WebP, responsive `srcset`; streefdoel LCP < 1,5s op 4G, Lighthouse mobiel 95+.

### 5.3 Detailpagina
- Fotogalerij, kernwaarden direct zichtbaar (prijs, opp., grond, DPE, slaapkamers).
- NL-vertaalde beschrijving + FR origineel (tab).
- Landkaartje met omgeving (statisch, geen zware JS-kaart bij eerste load).
- CTA naar bronwebsite (opent in nieuw tabblad, `rel="nofollow sponsored"` indien commercieel).

---

## 6. SEO & AI-vindbaarheid

### 6.1 Techniek
- Statische HTML, semantische tags, correcte headings.
- `sitemap.xml` (alle listings + alle landingpagina's), `robots.txt`.
- Structured data: `Product`/`Offer` (prijs, valuta), `RealEstateListing`-velden via `Product`-variant, `BreadcrumbList`, `FAQPage` op gidsartikelen.
- `hreflang`: `nl` primair, later `fr-FR` en `en` varianten.

### 6.2 Programmatische landingpagina's (de kern van de strategie)
Combinaties van regio × type × kenmerk genereren echte pagina's met unieke teksten en live gefilterde resultaten:
- `/regio/dordogne/huis-met-grond`
- `/regio/provence/mas-met-zwembad`
- `/regio/bretagne/te-renoveren-onder-100000`

Elke pagina krijgt unieke, nuttige inhoud (marktinfo, gemiddelde prijzen, kenmerken van de regio) — geen dunne duplicate content.

### 6.3 AI-zoekmachines (ChatGPT, Perplexity, AI Overviews)
- Feiten gestructureerd op de pagina (niet alleen in afbeeldingen/JS) — AI-crawlers lezen platte HTML.
- `llms.txt` met een gestructureerde sitetekst.
- Gidsartikelen die vragen beantwoorden ("Wat kost een huis in de Dordogne?", "Wat is DPE en waarom belangrijk?") — dit is wat AI-assistenten citeren.
- Snelle laadtijd en stabiele, semantische URL-structuur.

---

## 7. Juridisch
| Onderwerp | Aanpak |
|---|---|
| Databankrichtlijn / ToS | Geen massale scrapes; feeds en eigen invoer als fundament |
| Fotocopyright | Foto's blijven herleidbaar naar bron; CDN-proxy met bronvermelding; verwijdering bij verzoek |
| AVG | Geen persoonsgegevens van verkopers opslaan; contact loopt via bron; privacyverklaring |
| Hoguet-wet (FR makelaarsrecht) | Verwijderen naar bron-makelaar; geen eigen bemiddeling zonder vergunning |
| Verplichte advertentie-info FR | Honoraires, DPE, geotechnische risico's (ERNMT) tonen zoals de bron die vermeldt |

---

## 8. Roadmap
| Fase | Scope | Resultaat |
|---|---|---|
| **M1 – MVP** | Datamodel, ingest-pipeline (XML/CSV), zoekindex, homepage, resultaten met volledige filters, detailpagina's, mobiel-first UI, demo-data van 200 realistische woningen | Werkende, snelle site |
| **M2 – Echte data** | 3–5 echte feeds (Apimo/CustoFloos/EviWeb-makelaars), gratis invoeromgeving, deduplicatie | Eerste echte dekking |
| **M3 – SEO-offensief** | Programmatische landingpagina's, gidsartikelen, sitemaps, structured data | Organische groei |
| **M4 – Schaal** | Feed-netwerk uitbreiden, partnerships, FR/EN-talen, alerts (e-mail/PWA-push) | Marktdekkend doel |

---

## 9. Openstaande beslissingen
1. Astro (voorkev: maximale platheid) vs. Next.js (meer ecosysteem) — beide passen in dit plan.
2. Meilisearch (hosted managed) vs. Typesense (iets robuuster op schaal).
3. Eerste makelaarsoftware-partner: Apimo of CustoFloos eerst benaderen.
4. Vertalingen: machinevertaling FR→NL (DeepL) bij ingest, of tonen we FR met NL-filterlabels?
