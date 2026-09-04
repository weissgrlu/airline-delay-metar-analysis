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
