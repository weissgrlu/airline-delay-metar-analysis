# airline-delay-metar-analysis
The Hidden Cost of Weather | Datový audit odhalující indukovaná meteorologická zpoždění v rotacích letadel aerolinek v USA pomocí BTS TranStats, METAR (IEM), Pythonu a Tableau.

## 1. FÁZE: ASK

### Business kontext
V komerčním letectví v USA (dle metodiky Bureau of Transportation Statistics) je každé zpoždění striktně kategorizováno do jedné z pěti kategorií: *Carrier*, *Weather*, *National Aviation System (NAS)*, *Security* a *Late Aircraft Arrival*.

Tento způsob reportingu vytváří zásadní analytické zkreslení:
* Jako **Weather Delay** je vykázáno pouze primární zpoždění na konkrétním letu, kdy meteorologické podmínky přímo znemožní odlet či přílet.
* Pokud však ranní bouřková fronta způsobí zpoždění stroje o 90 minut, všechny jeho navazující lety v daném dni jsou v oficiálních statistikách vykazovány pod kategorií **Late Aircraft Arrival**.
* Výsledkem je, že vedení letového provozu (Flight Operations / OCC) čelí neoprávněné kritice za selhání provozní spolehlivosti sítě, ačkoliv reálnou příčinou  je neschopnost letového řádu absorbovat prvotní vliv nepříznivého počasí.

### Business Task
Kvantifikovat a vizualizovat skutečný dopad konvektivního (bouřkového) počasí na letový provoz v červenci 2023 napříč 4 páteřními huby (**KORD (Orlando), KATL (Atlanta), KDFW (Dallas), KJFK (New York)**). Prostřednictvím analýzy kompletních denních rotací jednotlivých letadel (*Tail Numbers*) odhalit objem „indukovaného počasí“ skrytého v kategorii *Late Aircraft Arrival*, spočítat kaskádový index řetězení a vyčíslit finanční dopad pro optimalizaci provozních nákladů.

### Klíčoví stakeholdeři a cílové rozhodnutí
* **Primární stakeholder:** Viceprezident letového provozu (VP of Flight Operations) a ředitel střediska řízení provozu (OCC).
* **Sekundární stakeholder:** Finanční controlling / Plánování letových řádů (Network Planning).
* **Cílové rozhodnutí (Actionable Decision):** 
  * Úprava plánovaných pozemních časů otoček (tzv. turnaround time) na exponovaných hubech v konvektivních odpoledních oknech.
  * Strategické umisťování záložních letadel a pohotovostních posádek pro aktivní přerušení kaskádového řetězení zpoždění.

### Klíčové metriky (KPIs)
* **Primární meteorologické zpoždění ($D_{prim}$):** Zpoždění prvního letu v rotaci přímo navázané na meteorologické podmínky na letišti (METAR: IMC/LIFR podmínky, TSRA apod.).
* **Indukované zpoždění ($D_{ind}$):** Následná zpoždění na odletu/příletu dalších legů stejného letadla (*Tail Number*) vzniklá jako důsledek pozdního příletu (*Late Aircraft Arrival*).
* **Kaskádový index (Propagation Multiplier):** Průměrný počet navazujících zpožděných letů a celkový objem minut vygenerovaných jedním primárně zasaženým letem.
* **Absorpční cutoff:** Hranice zpoždění na příletu pod 15 minut nebo noční odstávka letadla, kdy se kaskáda považuje za úspěšně absorbovanou/ukončenou.
* **Finanční dopad (Direct Operating Cost):** Celková finanční ztráta kalkulovaná průmyslovým standardem FAA/A4A ve výši **100 USD za každou minutu zpoždění**.

## 2. FÁZE: PREPARE

### Datové zdroje a licenční rámec

Naše analýza primárně čerpá ze dvou veřejně dostupných datových sad.

1. **BTS On-Time Performance Data (Bureau of Transportation Statistics, U.S. DOT)**
   * **Obsah:** Detailní záznamy o komerčních vnitrostátních letech v USA za **červenec 2023** včetně identifikátorů strojů (*Tail Number*), letových časů (plánované vs. skutečné časy odletů a příletů) a oficiálního rozkladu zpoždění (*Carrier*, *Weather*, *NAS*, *Late Aircraft*, *Security*).
   * **Zdroj:** U.S. Department of Transportation, Bureau of Transportation Statistics (transtats.bts.gov).
   * **Licence:** Public Domain (dílo federální vlády USA dle 17 U.S.C. § 105). Volně dostupné pro komerční i analytické využití bez restrikcí.
   * **Omezení soukromí (Privacy/PII):** Data neobsahují žádné osobní údaje cestujících ani posádek; pracují výhradně s agregovanými provozními identifikátory (číslo letu, imatrikulace stroje, IATA/ICAO kódy letišť).

2. **ASOS/AWOS METAR Historical Archives (IEM / NOAA / NWS)**
   * **Obsah:** Archiv standardizovaných leteckých meteorologických zpráv METAR a SPECI z letištních stanic **KORD, KATL, KDFW a KJFK** za období **1. 7. 2023 – 31. 7. 2023**. Zprávy obsahují čas pozorování (UTC), směr a sílu větru, dohlednost, jevy počasí (TSRA, FG, RA apod.), oblačnost (základny a pokrytí) a stav tlaku.
   * **Zdroj:** Iowa Environmental Mesonet (IEM), Iowa State University / National Oceanic and Atmospheric Administration (mesonet.agron.iastate.edu).
   * **Licence:** Open Access / Public Domain (otevřená akademická a vládní data).
   * **Formát:** Textové řetězce METAR v časových řadách (strukturované CSV/JSON přes IEM API).
  
 ### Ověření důvěryhodnosti dat
 | Dimenze ROCCC | Hodnocení | Odůvodnění pro BTS TranStats | Odůvodnění pro IEM ASOS (METAR) |
| :--- | :---: | :--- | :--- |
| **Reliable** (Spolehlivost) | **Vysoká** | Hlášení je pro certifikované americké dopravce zákonnou povinností podléhající federálnímu auditu DOT. | Měření provádějí certifikované letištní kalibrované senzory ASOS/AWOS s pravidelnou validací NWS/FAA. |
| **Original** (Původnost) | **Vysoká** | Primární repozitář federální vlády, nikoliv agregátor či odhad třetí strany. | Primární vědecký archiv stanic NOAA/NWS spravovaný univerzitním mesonetem. |
| **Comprehensive** (Komplexnost) | **Vysoká** | Obsahuje kompletní přehled letů velkých aerolinek a klíčový identifikátor letadla (`Tail_Number`) nutný pro trasování rotace. | Zahrnuje jak pravidelné zprávy, tak mimořádné zprávy SPECI zachycující přesný nástup bouřek (TSRA) či pokles dohlednosti. |
| **Current** (Aktuálnost) | **Vysoká** | Stabilizovaná, validovaná historická data za klíčové referenční léto 2023. | Kompletní, neměnná historická časová řada za sledované období červenec 2023. |
| **Cited** (Citovatelnost) | **Vysoká** | Standardizovaná metodika DOT pro vykazování provozní spolehlivosti komerčních letů. | Mezinárodní formát WMO/ICAO pro kódování zpráv METAR s jasně doloženou dokumentací API. |

### Rozsah datových souborů
Zprávy METAR dohromady obsahují asi 36 000 řádků (přibližně 9 000 řádků na jedno letiště), velikost souborů je v řádech jednotek MB.
Údaje o leteckém provozu v USA za měsíc červenec obsahují přibližně 600 000 řádků a datový soubor má velikost přibližně 250 MB.

## 3. FÁZE: PROCESS
### Výběr nástroje
Jako primární nástroj pro zpracování a transformaci dat jsme zvolili Python (knihovna Pandas) v prostředí Jupyter Notebook / Google Colab. Python jsme zvolili hlavně z důvodu, že jsme museli relativně složitě spojovat dva datové soubory.

### Kontrola kvality dat a transformace
Před samotnou tvorbou vyčištěného datasetu jsme provedli kontrolu obou zdrojů (BTS On-Time Reporting a IEM ASOS METAR) a aplikovali následující čistící kroky:
* **Zrušené lety (`Cancelled == 1`):** Ze surového souboru BTS bylo odstraněno ~11 000 neuskutečněných letů, jelikož nenesou reálné časy a netvoří provozní rotaci stroje.
* **Chybějící registrace letadel (`Tail_Number is NaN`):** Záznamy bez unikátní imatrikulace letadla byly vyřazeny; bez tohoto identifikátoru nelze sledovat návaznost rotací v kaskádové analýze.
* **Ošetření NULL hodnot u složek zpoždění:** Vládní statistika BTS vykazuje `NaN`, pokud zpoždění nevzniklo nebo bylo kratší než 15 minut. Sloupce `CarrierDelay`, `WeatherDelay`, `NASDelay` a `LateAircraftDelay` byly korektně nahrazeny nulou (`0`).
* **Normalizace časů a konverze do UTC:** Lokální časy z letišť (uváděné jako float/int např. `905.0` či půlnoční anomálie `2400.0`) byly převedeny na standardní časové řetězce, namapovány na příslušná časová pásma hubů (EDT pro ATL/JFK, CDT pro ORD/DFW) a sjednoceny do jednotného světového času (`UTC`) pro možnost párování.
* **Duplicity v meteorologických hlášeních:** Záznamy v archivu METAR byly deduplikovány na úrovni primárního kompozitního klíče `[station, valid]`, čímž se eliminovala vícenásobná hlášení v identický čas.
* **Rozložení zpráv METAR:** Nestrukturovaný text hlášení byl pomocí výrazů rozložen na konkrétní metriky:
  * Dohlednost v mílích (`visibility_sm`) včetně zlomků (např. `2 1/2SM` -> `2.5`).
  * Strop oblačnosti (`ceiling_ft`) vyhodnocením nejnižších souvislých vrstev `BKN` nebo `OVC`.
  * Detekce bouřkových a konvektivních jevů (`has_storm` – výskyt kódů `TS`, `TSRA`, `VCTS`, `CB`).
* **Kategorizace letových podmínek:** Z výše uvedených hodnot byla exaktně odvozena oficiální letová kategorie FAA: **VFR**, **MVFR**, **IFR** a **LIFR**.
* **Spárování datasetů (As-of Join):** Spojení odletů s počasím proběhlo přes `pd.merge_asof` s tolerančním oknem 75 minut zpětně (`direction="backward"`), což zajistilo přiřazení nejčastěji naposledy vydaného platného METARu před plánovaným časem odletu.

### Tvorba nových metrik a příznaků
* `CRSDepTime_str` & `CRSDep_dt_utc`: Normalizovaný plánovaný čas odletu v UTC pro jednotné časové srovnání.
* `visibility_sm`: Číselně vyjádřená horizontální dohlednost ve statutárních mílích.
* `ceiling_ft`: Výška základny nejnižší souvislé oblačnosti v absolutních stopách AGL.
* `has_storm`: Booleovský příznak (`True`/`False`) indikující přítomnost bouřkové činnosti na hubu.
* `flight_category`: Oficiální klasifikace meteorologických podmínek dle FAA standardu (VFR, MVFR, IFR, LIFR).

### Bilance čištění a export
Veškeré operace čištění a úpravy jsou provedeny v `notebooks/project_jupyter_files.ipynb`. Výsledný čistý dataset byl exportován do `data/processed/clean_flights_weather.csv`.

| Metrika / Parametr | Surová data (Raw) | Vyčištěná data (Processed) | Poznámka |
| :--- | :--- | :--- | :--- |
| **Lety z cílových hubů (ORD, ATL, DFW, JFK)** | 87 655 odletů | **84 148 odletů** | Úspěšně spárováno **96,00 %** letů |
| **Počet sloupců** | > 100 v BTS + surový METAR | **23 vybraných a odvozených** | Redukce paměťové náročnosti |
| **Zrušené a neidentifikovatelné lety** | Zahrnuty | **0** | 100% integrita rotací stroje |
| **Časová reference** | Lokální časy (CDT / EDT) | **Jednotný UTC** | Zajištěna synchronizace s METAR |
| **Meteorologický kontext** | Surový textový řetězec | **Strukturované FAA kategorie** | Připraveno pro analytické modelování |

## 4. FÁZE: ANALYSE
V této části uvedeme výsledky analýzy dat, která je v Pythonu provedena v souboru `project_jupyter_files.ipynb`. Analýza je logicky rozdělená do 4. odstavců.

### 1. analýza
Porovnání odletového zpoždění (`DepDelay`) a oficiálních složek zpoždění napříč letovými kategoriemi FAA za červenec 2023 (84 148 odletů ze 4 hubů - letišť):

| Letová kategorie (FAA) | Celkem letů | Zpoždění > 15 min (%) | Průměrné zpoždění | Medián | Weather Delay (min) | Late Aircraft Delay (min) | Poměr (Late / Weather) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **VFR** (Vizuální podmínky) | 77 597 | 30,3 % | 23,0 min | 0 min | 100 841 | 742 583 | **7,4×** |
| **MVFR** (Mezní podmínky) | 5 170 | 30,9 % | 26,1 min | 0 min | 12 474 | 50 036 | **4,0×** |
| **IFR** (Přístrojové podmínky) | 1 439 | 40,0 % | 30,8 min | 7 min | 5 997 | 15 500 | **2,6×** |
| **LIFR** (Nízké přístrojové) | 26 | 57,7 % | 74,6 min | 28 min | 126 | 391 | **3,1×** |
| **Celkový součet / průměr** | **84 148** | **30,6 %** | **23,4 min** | **0 min** | **119 438** | **808 510** | **6,8×** |

> **Klíčové zjištění 1:** Oficiální statistika zachycuje pouze zlomek problému. Na každou 1 minutu oficiálně přiznaného meteorologického zpoždění připadá v síti 6,8 minut sekundárních zpoždění vykazovaných jako čekání na pozdní přílet letadla (`LateAircraftDelay`).
### 2. analýza
Pro analýzu dominového efektu bylo nutné transformovat izolované záznamy o letech na chronologické řetězce jednotlivých strojů v čase (UTC):

* **Identifikace flotily:** Sledováno celkem **4 817 unikátních letadel**, přičemž maximální denní vytížení na sledovaných hubech dosáhlo **6 navazujících odletů** na jeden stroj.
* **Plánované rozestupy:** Průměrný časový odstup mezi dvěma odlety stejného letadla z hubu činil **370,5 minuty** (~6,2 h), což zahrnuje jak standardní pozemní obrat, tak mezilehlé rotace na regionální letiště mimo auditované huby.

#### Případová ukázka absorpce skluzu (Letoun N101DQ | 2. července 2023)
Typický příklad provozní absorpce zpoždění v rámci denní rotace:

| Leg | Trasa | Plánovaný odlet (UTC) | DepDelay | Rozestup (Buffer) | Výsledný stav |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **Leg 1** | ATL $\rightarrow$ IAH | 2023-07-02 11:05 | **0 min** | — | Výchozí let dne – odlet přesně načas |
| **Leg 2** | ATL $\rightarrow$ LGA | 2023-07-02 17:30 | **+17 min** | 385 min | Vznik provozního zpoždění (> 15 min) |
| **Leg 3** | ATL $\rightarrow$ SAN | 2023-07-03 01:16 | **-1 min** | 466 min | **Absorpce:** Dostatečný buffer skluz zcela vymazal |

> **Klíčové zjištění 2:** Tento krok potvrdil nutnost zavedení **absorpčního cutoffu** pro Krok 3 – kaskáda indukovaného zpoždění nepokračuje donekonečna, ale zaniká ve chvíli, kdy dostatečný pozemní buffer stlačí zpoždění pod 15 minut, případně po noční provozní pauze.

### 3. analýza
Pomocí řetězení v rámci denních rotací jednotlivých strojů (`Tail_Number`) byl měřen přenos primárního meteorologického zpoždění na navazující lety téhož dne:

| Metrika řetězení | Hodnota | Manažerský a metodický význam |
| :--- | :---: | :--- |
| **Primární spouštěcí lety** | **3 917** | Lety se zpožděním > 15 min v podmínkách IMC/bouřky |
| **Primární zpoždění na spouštěčích** | **344 091 min** | Přímý dopad nepříznivého počasí na hubu (~88 min / let) |
| **Indukované zpožděné lety** | **529** | Následné lety stroje infikované skluzem z předchozího počasí |
| **Indukované sekundární zpoždění** | **38 317 min** | Zpoždění vykazované jako *Late Aircraft* s meteorologickým původem |
| **Kaskádový index (Cascade Multiplier)** | **0,11×** | Na 1 minutu primárního počasí vzniká 0,11 min sekundárního zpoždění |
| **Finanční škoda (FAA benchmark $100/min)** | **$3 831 700** | Přímá finanční ztráta z indukovaných zpoždění za jediný měsíc |

> **Klíčové zjištění 3:**
> * **Neprávem penalizovaný provoz:** Minimálně **38 317 minut** zpoždění (ekvivalent **$3,83 mil. USD**), které interní controlling aerolinky vykazuje jako provozní selhání letištního personálu a pozemních rotací (*Late Aircraft Arrival*), vzniklo prokazatelně jako sekundární důsledek konvektivního počasí.

> **Klíčové zjištění 4:**
> * **Proč je index 0,11× konzervativní:** Analýza sleduje pouze odlety ze 4 páteřních hubů. Pokud stroj po bouřce na hubu rotoval přes regionální letiště (kde absorboval část skluzu a náš dataset ho neviděl), do výpočtu nevstoupil. Skutečný kaskádový násobitel napříč celou sítí je proto podstatně vyšší.

### 4. analýza
Srovnání schopnosti vstřebat nepříznivé počasí mezi 4 páteřními uzly a vyčíslení skrytých finančních škod (přímé provozní náklady dle FAA: 100 USD / minuta zpoždění):

| Hub (IATA) | Celkem odletů | Primární spouštěče | Indukované lety | Kaskádový index | Primární zpoždění (min) | Indukované zpoždění (min) | Finanční škoda (Indukovaná) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **ORD** (Chicago) | 21 231 | 1 508 | 172 | **0,094×** | 142 898 | 13 486 | **$1 348 600** |
| **ATL** (Atlanta) | 28 329 | 1 586 | 173 | **0,085×** | 110 361 | 9 341 | **$934 100** |
| **DFW** (Dallas) | 24 780 | 262 | 119 | **0,432×** | 20 435 | 8 823 | **$882 300** |
| **JFK** (New York) | 9 892 | 561 | 65 | **0,095×** | 70 397 | 6 667 | **$666 700** |
| **Celkem / Průměr** | **84 148** | **3 917** | **529** | **0,111×** | **344 091** | **38 317** | **$3 831 700** |

#### Časový profil eskalace dominového efektu
Rozpad sekundárního zpoždění podle plánované hodiny odletu odhaluje kritické provozní okno:

* **Špička přenosu indukovaného zpoždění (22:00–02:00 UTC / 17:00–21:00 lokálně):** V tomto pětihodinovém okně vzniká přes **23 300 minut** (více než 60 %) veškerých indukovaných zpoždění celého dne.
* **Absolutní vrchol v 01:00 UTC (20:00 CDT):** Samotná 1. hodina ranní v UTC kumuluje **6 354 minut** indukovaného skluzu, což je přímý následek návratu zpožděných letadel z odpoledních bouřkových linek na centrální báze.

> **Klíčové zjištění 5:**  
> Zranitelnost Dallasu (DFW) s indexem **0,432×** ukazuje na poddimenzované turnround buffery v odpoledních vlnách American Airlines. Právě sem a do večerního časového okna 17:00–21:00 lokálního času by mělo řízení provozu firmy přednostně směrovat záložní letadla.


### Shrnutí analýzy
1. **Skrytá stopa počasí (Poměr 6,8 : 1):**
   * Vykázaná zpoždění z čekání na letadlo (`LateAircraftDelay`: 808,5 tis. min) převyšují oficiální zpoždění způsobená počasím (`WeatherDelay`: 119,4 tis. min) téměř sedminásobně.
   * I při optimálních vizuálních podmínkách (VFR) bylo zaznamenáno přes 742 tisíc minut zpoždění letadel, která si přivezla skluz z předchozích letů.

2. **Kvantifikace dominového efektu a finanční dopad:**
   * **3 917 primárních letů** zasažených nepříznivým počasím (IMC/bouřka) vygenerovalo **529 sekundárně infikovaných rotací**.
   * Identifikováno **38 317 minut indukovaného zpoždění**, které interní reporting neoprávněně připisuje provoznímu selhání pozemního personálu.
   * Při standardním FAA benchmarku (100 $/min) činí přímá finanční škoda z tohoto dominového efektu **3,83 mil. USD** (konzervativní odhad v rámci sledovaných hubů).

3. **Zranitelnost uzlů a časová okna (DFW vs. ORD):**
   * **Dallas/Fort Worth (DFW)** vykázal nejvyšší **Kaskádový index (0,432×)** – těsný letový řád v odpoledních špičkách nedokáže absorbovat zpoždění z konvektivních bouří.
   * **Chicago (ORD)** utrpělo nejvyšší absolutní škodu (**$1,35 mil. USD**).
   * **Kritické okno (22:00–02:00 UTC / 17:00–21:00 lokálně):** V této podvečerní vlně vzniká přes 60 % veškerých kaskádových zpoždění dne, kdy se zpožděné stroje vracejí na základny.

### Datové výstupy pro fázi SHARE
Veškeré analytické transformace proběhly v notebooku `notebooks/03_analyze_rotations_and_cascades.ipynb`. Pro vizualizaci v Tableau byly vygenerovány 3 agregované datasety v `data/processed/`:
* `tableau_hub_resilience.csv` – Srovnání odolnosti hubů, kaskádových indexů a finančních ztrát.
* `tableau_delay_breakdown.csv` – Bilanční srovnání oficiálního reportingu BTS vs. auditovaného stavu.
* `tableau_sample_rotations_gantt.csv` – Časové podklady pro Ganttův diagram rotací infikovaných letadel.
