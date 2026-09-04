# airline-delay-metar-analysis
The Hidden Cost of Weather | Datový audit odhalující indukovaná meteorologická zpoždění v rotacích letadel aerolinek v USA pomocí BTS TranStats, METAR (IEM), Pythonu a Tableau.

## 1. ASK: Definice business problému a cílů projektu

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
