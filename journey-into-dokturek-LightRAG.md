# Journey Into dokturek-LightRAG

*Technicko-historická analýza vývojové časové osy projektu z perzistentní paměti claude-mem*

Časové rozpětí: **6.–10. června 2026** · 104 pozorování · 6 sezení · 2 459 706 tokenů discovery work · pouze 35 623 tokenů reálně přečteno

---

## 1. Genesis projektu

Příběh `dokturek-LightRAG` nezačíná psaním kódu, ale aktem orientace. **6. června 2026 v 11:01 (#17800)** padlo úplně první pozorování — mapování zásobníku a struktury projektu. V rychlém sledu pak během minuty přišla pozorování #17801, #17802 a #17803, která dohromady tvoří kompletní úvodní audit: jádro LightRAG (mixin kompozice, čtyři typy storage, kontrakt pipeline concurrency), dev workflow s rozložením testů a kritickými pitfally kolem init/embedding, a konečně git topologie forku včetně `.clinerules`.

Tento čtyřnásobný úvodní sken nebyl náhodný. Projekt je **fork `HKUDS/LightRAG` pro Dokturek.ai**, a sezení **S2293/S2292** explicitně spustilo `/init` pro vygenerování `CLAUDE.md`. Místo psaní funkcionality se nejprve budovala mapa terénu — rozhodnutí, které se vyplatí v každé následující debugging sáze, protože pozdější pozorování se opakovaně vracejí k těmto úvodním zjištěním o tom, kde leží embedding pitfally a jak je propojený git remote (`origin` = `petrsovadina/dokturek-LightRAG`, `upstream` = `HKUDS/LightRAG`).

Druhý prompt sezení — **S2296, "a co postgres a neo4j?"** — je v retrospektivě klíčový. Jediná, téměř mimoděčná otázka uživatele otevřela největší architektonický pivot celé historie. Než se ale dalo cokoliv provisionovat, bylo potřeba pochopit existující nasazení: pozorování #17804 až #17808 (11:05–11:13) zmapovala, že **Railway projekt SVL.ai** už hostí dvě běžící služby (`dokturek-LightRAG` a `MinerU`), že produkční prostředí je zdravé, a — zásadní zjištění #17806 — že **reálná produkční konfigurace se liší od toho, co tvrdí `.clinerules`**. Toto napětí mezi dokumentací a realitou je leitmotiv, který se táhne celým projektem.

---

## 2. Evoluce architektury

Migrace z file-storage na databázové backendy je nejdramatičtějším obloukem 6. června a celá se odehrála v jediném zhuštěném dopoledni mezi 11:13 a 11:32.

Sezení **S2297** stanovilo cíl: provisionovat Postgres+pgvector a Neo4j na Railway a opustit file storage. Pozorování #17808 (11:13) potvrdilo dostupnost Railway šablon, #17809 (11:20) instalaci a autentizaci Railway CLI v5.0.0. Pak začalo nasazování — a okamžitě naráželo. To rozebírám detailně v sekci 6, ale architektonický výsledek je čistý: **#17821 (11:31)** zachycuje finální mapu service ID se **čtyřmi službami** po dokončení DB provisioningu, a **#17825 (11:32)** dokumentuje moment, kdy bylo LightRAG propojeno s Postgresem i Neo4j a storage selektory byly překlopeny na DB backendy.

Tím se architektura ustálila do podoby, kterou popisuje i `CLAUDE.md`: `PGKVStorage` / `PGVectorStorage` / `PGDocStatusStorage` na službě pgVector-Railway plus `Neo4JStorage` na samostatné Neo4j službě, vše komunikující přes Railway private DNS (`*.railway.internal`). Pozorování #17822 zachytilo connection variables pro Postgres, #17826 potvrdilo, že Neo4j Bolt naběhl při prvním startu.

Migrace ovšem nebyla "hotová" tím, že se přepnuly selektory. Skutečné potvrzení přišlo až o hodinu a čtvrt později: **#17827 (12:28)**, jediné pozorování typu `feature` v této fázi, konstatuje *"LightRAG live on Postgres+pgvector and Neo4j backends — migration confirmed working"*. Rozdíl mezi "překlopeno" (11:32) a "ověřeno funkční" (12:28) je přesně ta mezera, kde se schovávají bugy — a opravdu se v ní jeden schoval.

---

## 3. Klíčové průlomy

Tři "aha" momenty definují tento projekt, a všechny tři se shlukují kolem zprovoznění end-to-end RAG.

**Průlom první — Jina embedding endpoint fix.** Po nastavení API klíčů (#17851, 14:29) první ingest sice formálně uspěl (#17852, 14:30, *"first document ingest succeeded"*), ale o čtyři minuty později **#17853 (14:34)** odhalil pravdu: dokument uvázl ve `failed` statusu. Diagnostika v #17854 (14:35) vystopovala příčinu k Jina embedding endpointu vracejícímu **404 "Invalid endpoint"**. Klíčové zjištění #17855 — čtení samotného kódu `jina.py` ukázalo, že `EMBEDDING_BINDING_HOST` **nesmí** obsahovat suffix `/embeddings`. Oprava #17856 (14:36) byla mechanicky triviální, ale konceptuálně to byl skutečný průlom: pochopení, že jina klient POSTuje host tak, jak je. (Toto je dnes zakotveno přímo v `CLAUDE.md` jako pitfall: host musí být plné `https://api.jina.ai/v1/embeddings`.)

**Průlom druhý — ingest pipeline opraven.** Mezi 14:36 a 16:46 nastala dvouhodinová mezera (viz sekce 4), po níž **#17862 (16:46)** ohlásil: *"Ingest pipeline fixed — documents now reach PROCESSED"*. Dokumenty konečně procházely celým řetězcem.

**Průlom třetí — end-to-end RAG s citacemi.** Bezprostředně poté **#17863 (16:46)**, druhé `feature` pozorování, potvrdilo *"End-to-end RAG query verified with citations"*. To byl skutečný cíl celého dne: ne "databáze běží", ale "uživatel dostane odpověď s citacemi". Disciplína projektu se ukázala vzápětí — **#17864** smazal smoke-test dokumenty, aby produkční DB zůstaly čisté. Linear DEV-110 byl označen jako Done (#17865) a vyřešení zdokumentováno (#17866).

---

## 4. Vzorce práce

Časová osa odhaluje tři jasně odlišitelné režimy práce.

**Provisioning sprint (6. června, 11:01–11:32).** Hustý, téměř minutový rytmus pozorování. Třicet pozorování za třicet jedna minut — to je infrastrukturní práce s CLI, kde každý příkaz generuje okamžité zjištění. Tady dominuje typ `discovery` proložený `change` a `security_note`.

**Debugging cyklus (6. června, 14:34–16:46).** Klasický oblouk *problém → vyšetřování → řešení*: #17853 (selhání) → #17854/#17855 (vyšetřování) → #17856 (oprava) → **dvouhodinová mezera** → #17862 (ověřeno). Ta mezera mezi 14:36 a 16:46 je telling — Jina endpoint fix sám o sobě ingest neopravil úplně; muselo proběhnout další, v paměti nezaznamenané ladění, než pipeline skutečně dosáhla `PROCESSED`. Mezery v časové ose jsou stejně výmluvné jako pozorování sama.

**Feature sprint / rebranding (10. června).** Po dvoudenní pauze (7. června jen drobné konfigurační exploration, 8.–9. nic) přišel sprint úplně jiného charakteru. Od #18051 (2:59) přes #18102 (22:19) — devatenáctihodinový den věnovaný designu a rebrandingu LightRAG WebUI na vizuální identitu Doktůrek.ai. Tady dominuje typ `feature` (12 z celkových 12 `feature` pozorování spadá sem nebo do migrace) a `change`. Rytmus je nárazový: ranní exploration design tokenů (#18051–#18059), polední audit živého webu (#18060–#18068), večerní implementační dávka (#18069–#18102).

**Exploration fáze (7. června, 1:37–2:44).** Krátké, klidné sezení věnované pochopení konfigurace JSON entity extraction (#17882–#17889) a auditu `env.example` proti živé Railway konfiguraci (#17895). Žádné psaní, jen čtení a mapování — příprava půdy.

---

## 5. Technický dluh

Tento projekt je zajímavý tím, jak **rychle** se technický dluh splácel — často v řádu minut, ne týdnů.

**Duplicitní pgvector služby.** Nejviditelnější zkratka vznikla nechtěně: opakované volání `railway deploy --template` (které mate, protože vypisuje jen "Creating..." a končí exit 0) provisionovalo **dvě** pgvector služby (#17814, 11:22). Obě běžely Online s odlišnými service ID (#17815). Dluh ale nepřežil ani deset minut — po komplikacích s mazáním (#17816, #17817) byla duplicitní služba odstraněna v #17818 (11:28).

**Neo4j bez autentizace.** Nejostřejší bezpečnostní zkratka: Neo4j šablona se nasadila s `NEO4J_AUTH=none` (#17823, 11:31, `security_alert`). Tady byl dluh splacen **okamžitě v následujícím pozorování** — #17824 zapnul autentizaci a vyřešil no-auth expozici dříve, než stihla být problémem. Pozorování #17826 pak potvrdilo, že auth se aplikoval při prvním startu a Bolt běží.

**Veřejné TCP proxy.** Postgres (5432) i Neo4j (7687) měly zprvotně veřejné TCP proxy (#17840, 14:21). Tato povrchová plocha byla zlikvidována v #17841 a ověřena v #17842 — obě DB jsou teď private-only, jak dnes potvrzuje i `CLAUDE.md`.

**Hardcoded barvy v WebUI.** Designový dluh z 10. června: rebranding nejprve vypadal "too subtle" (sezení **S2317**), protože swap palety v `index.css` přebarvil jen shadcn primitivy, zatímco feature surfaces měly hardcoded `emerald`/`teal` třídy obcházející theme. Příkladem je `PropertiesView` s natvrdo zapsaným `text-emerald-700` (#18098). Dluh byl splacen systematicky — osm hardcoded-color míst opraveno, #18099 přebarvil heading na `text-primary`, #18100 ověřil nulové reziduum emerald/teal a čistý `tsc --noEmit`.

Vzorec je konzistentní: zkratky tu vznikají jako vedlejší produkt rychlosti, ale jsou téměř vždy zachyceny a splaceny v rámci téhož sezení. To je možné jen proto, že paměťový systém okamžitě dokumentuje i ten dluh (`security_alert`, `security_note`), takže nezapadne.

---

## 6. Výzvy a debugging ságy

**Sága Railway CLI — "exit 0 but failed".** Nejúpornějším protivníkem provisioningu nebyl složitý bug, ale **prolhaný exit kód**. Pozorování #17813 (11:20) zachytilo, že `railway deploy --template` vypíše jen "Creating..." a skončí exit 0 — bez ohledu na to, co se reálně stalo. Tento vzorec přímo způsobil duplicitní pgvector služby (operátor opakoval příkaz, protože nevěděl, že první proběhl). A pak udeřil znovu z opačné strany: #17816 (11:24) — `railway service delete` selhal s response-decode chybou, ale **opět skončil exit 0**, takže #17817 musel konstatovat, že smazání nezabralo. Tato dvojí zrada (úspěch hlášený jako úspěch i selhání hlášené jako úspěch) je nejcennější ponaučení celé infrastrukturní fáze.

**Sága Jina 404.** Popsáno v sekci 3 — endpoint vracel "Invalid endpoint", protože konfigurace přidávala suffix `/embeddings`, který jina klient přidávat nesmí. Klasický příklad chyby, kterou nelze uhádnout a je nutné ji vyčíst z kódu (#17855 četl `jina.py`).

**Sága Linear OAuth — opakované selhání.** Tohle byla nejúmornější epizoda celé historie, rozprostřená přes sedm sezení (**S2300–S2306**). Linha selhání:
- #17833 (12:28) — Linear MCP server přidán přes SSE, čeká na OAuth.
- **S2301** — OAuth zrušen uživatelem.
- **S2302** — sezení zablokováno, čeká na rozhodnutí Linear vs. Railway hardening.
- **S2303** — re-verifikace, opětovná výzva k dokončení OAuth.
- #17834 (14:12) — diagnostika: programmatic auth selhává kvůli **SSE/MCP endpoint mismatch**.
- #17835 (14:16) — re-registrace na HTTP `/mcp` endpoint jako pokus o opravu.
- #17836 — auth **stále** selhává (stale session) i po HTTP re-registraci.
- #17837 (14:18) — **konečně připojeno**; jako vedlejší zjištění odhaleni dva uživatelé "Sovadina" v workspace.

Trvalo tedy zhruba dvě hodiny a šest sezení překlopit Linear z "nefunguje" na "připojeno". Jakmile připojení fungovalo, hodnota přišla rychle: #17844–#17847 vytvořily issue DEV-109 (migrace + hardening) s child issues DEV-110 (produkční API klíče) a DEV-111 (re-ingest dokumentů), správně propojené relací blocked-by (#17847).

**Bezpečnostní incident — plaintext API klíče.** #17850 (14:28, `security_alert`) zaznamenává, že **živé API klíče byly vloženy v plaintextu přímo do promptu**. To je reálná expozice; klíče se sice vzápětí nasadily správně (#17851), ale incident zůstal v paměti zaznamenán jako varování.

---

## 7. Paměť a kontinuita

claude-mem fungoval v tomto projektu jako **institucionální paměť napříč přerušeními**. Projekt běžel v šesti sezeních rozprostřených přes pět dní s velkými mezerami (7.–9. června prakticky bez aktivity). Bez perzistentní paměti by každé sezení po pauze začínalo znovu od mapování terénu.

Konkrétní příklady kontinuity: úvodní audit z 6. června (#17800–#17806) sloužil jako základna, ke které se vracela každá pozdější diagnostika — embedding pitfall zachycený hned v #17802 se zhmotnil v Jina 404 sáze o tři hodiny později. Mapa service ID v #17821 zůstala referenčním bodem pro celou pozdější práci s Railway. A rebranding 10. června navázal na design-system exploration, jejíž tokeny byly extrahovány a uloženy už v rané fázi téhož dne (#18051–#18062), takže večerní implementační dávka (#18069+) mohla čerpat z ranního auditu, aniž by se web musel znovu skenovat.

Osm pozorování (z analýzy narrativ) explicitně odkazuje na dřívější kontext nebo "recall" — tedy zhruba 8 % pozorování staví na již uložené paměti. To je nízká absolutní hodnota, ale projekt je krátký a hustý; v delším projektu by tento podíl rostl. Důležitější je, že **žádné sezení nezačínalo od nuly** — všech šest navazovalo na zaznamenaný stav.

---

## 8. Token Economics & Memory ROI

Kvantitativní jádro hodnoty paměťového systému. Všechny hodnoty z `~/.claude-mem/claude-mem.db`, projekt `dokturek-LightRAG`.

| Metrika | Hodnota |
|---|---|
| Celkem discovery tokenů (odvedená "práce") | **2 459 706** |
| Celkem reálně přečtených tokenů (uložený obsah / 4) | **35 623** |
| Počet pozorování | 104 |
| Počet sezení | 6 |
| Pozorování s discovery_tokens > 0 | 104 (100 %) |
| Průměrné discovery tokeny / pozorování | 23 651 |
| Průměrné read tokeny / pozorování | 343 |
| Poměr discovery : read (na pozorování) | ≈ **69 : 1** |

### Top 5 nejdražších pozorování (dle discovery_tokens)

| ID | Title | discovery_tokens |
|---|---|---|
| #18102 | Dokturek rebrand visually verified in browser at localhost:5174 | **367 196** |
| #18074 | Verified dokturek rebrand renders on LightRAG WebUI login page | **329 043** |
| #18058 | .stitch output directory created for DESIGN.md | **130 718** |
| #18057 | dokturek.ai component styles: buttons, badges, nav extracted | **130 433** |
| #18056 | Full color, radius, shadow, gradient audit of dokturek.ai marketing site | **124 932** |

Je výmluvné, že **všech pět nejdražších pozorování pochází z rebranding/design sprintu 10. června**. Browser-based ověřování a DOM audity (živý web, computed styles, full-page screenshoty) jsou tokenově nejnáročnější operace — jediné ověření rebrandu v prohlížeči (#18102) spotřebovalo 367 tisíc tokenů discovery práce, ale do paměti se z něj uložilo jen ~340 tokenů destilovaného závěru. Naproti tomu infrastrukturní pozorování z 6. června jsou tokenově levná (CLI příkazy vrací krátké výstupy).

### Měsíční rozpad

| Měsíc | Pozorování | Discovery tokeny | Sezení |
|---|---|---|---|
| 2026-06 | 104 | 2 459 706 | 6 |

(Celý projekt proběhl v jednom kalendářním měsíci.)

### Odhad ROI

ROI = celková odvedená práce / reálně investované přečtené tokeny:

> **2 459 706 / 35 623 ≈ 69×**

Jinak řečeno: paměťový systém destiloval přibližně **2,46 milionu tokenů průzkumné práce do 35,6 tisíce tokenů znovupoužitelné paměti** — kompresní/úsporný poměr zhruba **69 : 1**, což odpovídá hlavičce timeline uvádějící **98 % úspory**. Každý token uložený do paměti reprezentuje téměř sedmdesát tokenů práce, kterou by jinak bylo nutné zopakovat při každém návratu k projektu po pauze. Při šesti sezeních rozprostřených přes pět dní s velkými mezerami je tato úspora reálná, ne teoretická.

---

## 9. Statistiky timeline

**Časové rozpětí:** 6. června 2026 09:01:50 UTC → 10. června 2026 20:19:02 UTC (≈ 4,5 dne)
**Celkem:** 104 pozorování · 6 sezení · 2 459 706 discovery tokenů · 35 623 read tokenů

### Rozpad podle typu

| Typ | Počet | Podíl |
|---|---|---|
| 🔵 discovery | 58 | 56 % |
| ✅ change | 24 | 23 % |
| 🟣 feature | 12 | 12 % |
| 🔐 security_note | 4 | 4 % |
| 🔴 bugfix | 3 | 3 % |
| 🚨 security_alert | 2 | 2 % |
| 🔄 refactor | 1 | 1 % |

Profil je typický pro fork v rané fázi: **dominuje discovery (56 %)** — projekt strávil většinu času orientací a mapováním existujícího kódu a infrastruktury, ne psaním nového. Vysoký podíl `change` (23 %) odráží konfigurační a wiring práci (Railway proměnné, Linear issues, brand tokeny). Pouhé tři `bugfix` pozorování jsou všechny z jediné Jina/ingest ságy (#17853, #17856, #17862). Dva `security_alert` (Neo4j no-auth, plaintext klíče) plus čtyři `security_note` ukazují, že bezpečnost byla aktivně sledována, ne ignorována.

---

## 10. Ponaučení a meta-pozorování

**Orientace před akcí se vyplácí.** Čtyři úvodní `discovery` pozorování (#17800–#17803) byla investice, která se vrátila v každé pozdější diagnostice. Embedding pitfall zaznamenaný v první hodině projektu předznamenal Jina 404 ságu o tři hodiny později — tým "věděl, kde hledat", protože si to dříve zmapoval.

**Nástroje lžou; ověřuj nezávisle.** Hlavní procedurální ponaučení celého projektu je nedůvěra k exit kódům. Railway CLI hlásil exit 0 jak při tichém úspěchu (vznik duplicitní služby), tak při tichém selhání (neúspěšné smazání). Skutečný stav se zjišťoval až nezávislým dotazem na inventář služeb (#17812, #17815, #17817). Vzorec "nedůvěřuj návratovému kódu, ověř výsledek" se opakuje i u ingestu — formálně "úspěšný" první ingest (#17852) ve skutečnosti uvázl ve `failed` (#17853).

**Dluh splácený okamžitě nezůstává dluhem.** Na rozdíl od mnoha projektů tady technický dluh nepřežíval. Neo4j no-auth opraven v následujícím pozorování, duplicitní služba do deseti minut, TCP proxy a hardcoded barvy v rámci téhož sezení. Klíčový enabler je, že paměťový systém dluh **explicitně zaznamenal** (jako `security_alert`/`security_note`), takže nemohl tiše zapadnout.

**"Hotovo" má dvě úrovně.** Opakovaně se ukazuje mezera mezi "překlopeno/nasazeno" a "ověřeno funkční": migrace překlopena v 11:32, ale potvrzena až ve 12:28; rebrand aplikován, ale označen "too subtle", dokud se nedohledala příčina v obcházení theme. Skutečná hodnota se uznávala až po vizuálním/funkčním ověření, ne po formálním provedení změny.

**Browser/design práce je tokenově nejdražší.** Pět nejnákladnějších pozorování jsou všechno DOM audity a browser ověřování. Pokud by se hledaly úspory, optimalizace průzkumu živého webu (cílenější selektory, méně full-page screenshotů) by měla největší dopad. Infrastrukturní CLI práce je naproti tomu levná.

**Jediná otázka uživatele může zlomit směr projektu.** "a co postgres a neo4j?" (S2296) — pět slov — spustilo celou architektonickou migraci, hardening, Linear tracking i následný rebranding. Drobné prompty s velkými následky jsou v této historii pravidlem, ne výjimkou.

---

*Konec zprávy. Vygenerováno z claude-mem perzistentní paměti, 11. června 2026.*
