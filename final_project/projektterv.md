# Késni fog-e a járat? — záró projekt terve

**Adat:** 2015 Flight Delays and Cancellations (amerikai közlekedési minisztérium, Kaggle `usdot/flight-delays`, CC0), a **Delta** járatai, 2015 egész éve,
plusz repülőtéri óránkénti időjárás (Iowa Environmental Mesonet, ASOS/METAR).
**Módszerek:** minden, amit az órákon tanultunk: EDA és vizualizáció, hipotézisvizsgálat, klasszifikáció, regresszió, időrendi kiértékelés, diagnosztika, visszacsatolási kör, előrejelzés, üzleti döntés.

---

## 1. A kérdés

**Fő kérdés:** az indulás előtt 2 órával meg tudjuk-e mondani, hogy egy Delta-járat **15 percnél többet késik-e** érkezéskor?

- **Ki használná?** A légitársaság műveleti központja (tartalék gép vagy személyzet, a csatlakozó utasok előzetes átfoglalása), illetve az utas (értesítés, másik csatlakozás).
- **Mikor jósolunk?** **Az indulás előtt 2 órával.** Csak az lehet jellemző, ami ekkor már ismert. Ez a projekt legfontosabb szabálya (ugyanaz, mint az 5. órán a `lag1` kérdése).
- **Miért 15 perc?** Ez az amerikai hivatalos késés-definíció, és az adatban a késés okai is csak 15 perc fölött vannak kitöltve.
- **Melyik hiba drágább?** A **kihagyott késés** (az utas lekési a csatlakozást, a légitársaság kártérítést fizet, a következő járat is késik). A téves riasztás is pénzbe kerül (felesleges átfoglalás), de kevesebbe. A pontos költségarányt feltételezzük, és érzékenységvizsgálattal nézzük meg.

**Mellékkérdések** (ha marad idő):
1. **Hány percet késik?** (regresszió, log-transzformációval)
2. **Mennyit ér az időjárás-előrejelzés?** A „2 órával előtte ismert” és a „tökéletesen ismert” időjárással tanított modell különbsége.
3. **Továbbgyűrűzik-e a késés?** Ugyanannak a gépnek az előző késése mennyire viszi át a késést a következő járatra?

---

## 2. Az adat (ellenőrizve a Kaggle-oldalon)

| Fájl | Tartalom | Méret |
|---|---|---|
| `flights.csv` | **5 819 079 járat, 31 oszlop**, 2015. jan.–dec. | 592 MB |
| `airports.csv` | repülőtér-kód, név, város, állam, ország, koordináták | kicsi |
| `airlines.csv` | 14 légitársaság kódja és neve | kicsi |

**A Delta-részhalmaz:** a járatok kb. 15%-a, **~870 ezer járat** (a pontos számot az előkészítés után írjuk be).
Miért a Delta? Kezelhető méret, és **megmarad a gépek teljes útja**, ami a késés-tovagyűrűzéshez kell. Az atlantai központ miatt az időjárás hatása is jól látszik.

**Időjárás:** IEM ASOS/METAR (https://mesonet.agron.iastate.edu/request/download.phtml), ingyenes, bejelentkezés nélkül. Óránkénti mérés repülőterenként,
ugyanazzal a hárombetűs kóddal, mint a járatadat. Kipróbálva: `ATL`, 2015-01-01, óránkénti sorok, UTC-ben.

**Ismert buktatók:**
- Az időpontok **helyi időben** vannak, ÓÓPP formátumban (`0545`); a `2400` érték éjfélt jelöl.
- Az **októberi** járatoknál a repülőtér-kódok (tudomásunk szerint) számok, nem IATA-kódok → az október a tanító és a teszt közötti résbe kerül, a modellezésből kimarad.
- A törölt és elterelt járatoknak nincs érkezési késésük → a fő elemzésből kimaradnak, de az EDA-ban külön megnézzük őket.

---

## 3. A változók feltérképezése

**A kulcskérdés minden oszlopnál: ismerném-e az indulás előtt 2 órával?**

### Az eredeti oszlopok

| Oszlop | Jelentés | Típus | Ismert 2 órával előtte? | Szerep |
|---|---|---|---|---|
| `YEAR`, `MONTH`, `DAY`, `DAY_OF_WEEK` | a járat napja | dátum | ✅ | jellemző (hónap, a hét napja, ünnep közelsége) |
| `SCHEDULED_DEPARTURE` | tervezett indulás (helyi, ÓÓPP) | idő | ✅ | jellemző (az indulás órája) |
| `SCHEDULED_ARRIVAL`, `SCHEDULED_TIME` | tervezett érkezés, tervezett menetidő | idő, perc | ✅ | jellemző |
| `AIRLINE` | légitársaság | kategória | ✅ | szűrés (csak `DL`) |
| `FLIGHT_NUMBER` | járatszám | kategória | ✅ | csak azonosításra (túl sok szint) |
| `TAIL_NUMBER` | a gép lajstromjele | kategória | ✅ | **a gép útjának követése** (a jellemzőkhöz) |
| `ORIGIN_AIRPORT`, `DESTINATION_AIRPORT` | induló és érkező repülőtér | kategória | ✅ | jellemző (és az időjárás kulcsa) |
| `DISTANCE` | távolság (mérföld) | szám | ✅ | jellemző |
| `DEPARTURE_TIME`, `DEPARTURE_DELAY` | tényleges indulás, indulási késés | idő, perc | ❌ | **szivárgás** (csak az előző járatoknál használható) |
| `TAXI_OUT`, `WHEELS_OFF`, `WHEELS_ON`, `TAXI_IN` | gurulás, fel- és leszállás | perc, idő | ❌ | szivárgás |
| `ELAPSED_TIME`, `AIR_TIME`, `ARRIVAL_TIME` | tényleges menetidő, repülési idő, érkezés | perc, idő | ❌ | szivárgás |
| **`ARRIVAL_DELAY`** | érkezési késés (perc; negatív = korai) | szám | — | **CÉLVÁLTOZÓ** (> 15 perc: 0/1) |
| `DIVERTED`, `CANCELLED`, `CANCELLATION_REASON` | elterelés, törlés és oka | 0/1, kód | ❌ | kizárás + külön EDA |
| `AIR_SYSTEM_DELAY`, `SECURITY_DELAY`, `AIRLINE_DELAY`, `LATE_AIRCRAFT_DELAY`, `WEATHER_DELAY` | a késés okai (perc) | szám | ❌ | **csak EDA és ellenőrzés**, soha nem jellemző |

### Származtatott jellemzők (mind ismert 2 órával előtte)

| Jellemző | Honnan | Miért |
|---|---|---|
| az indulás órája, a hét napja, hónap | menetrend | a késés napközben felhalmozódik; szezonalitás |
| napok a legközelebbi ünnepig | naptár | hálaadás, karácsony, július 4. |
| útvonal (induló + érkező) | menetrend | egyes útvonalak rendszeresen késnek |
| **forgalom:** tervezett indulások száma az adott órában a reptéren | menetrend (az egész adat, nem csak a Delta) | zsúfoltság |
| **a gép előző járata:** a legutolsó ismert késése a cutoffig | `TAIL_NUMBER` + időrend | a késés továbbgyűrűzése |
| **fordulóidő:** az előző járat tervezett érkezése és ennek tervezett indulása között | menetrend | rövid fordulóidőnél nincs tartalék |
| **időjárás az induló és az érkező reptéren** a cutoff pillanatában | IEM | zivatar, hó, köd, ónos eső, széllökés, látótávolság |
| **hálózati időjárás:** hány nagy repülőtéren van zivatar | IEM | egy atlantai vihar Chicagóban is késést okoz |
| történeti késési arány (útvonal, reptér, óra) | **csak a tanító adatból, foldon belül** | célkódolás, ezért szivárgásveszélyes |

**A gép előző járatánál vigyázni kell:** az előző járat késése csak akkor ismert, ha az a cutoff (indulás − 2 óra) előtt már leszállt.
Ha még a levegőben volt, csak az indulási késése ismert; ha még el sem indult, akkor semmi. A jellemző mindig **a cutoff pillanatában ismert legutolsó érték**.

---

## 4. Mutatók (az eredmények előtt rögzítve)

**Fő kérdés (klasszifikáció):**
- **Fő mutató: PR-AUC**, mert a késő járatok kisebbségben vannak (az arányt az EDA-ban mérjük), és a pozitív osztály a fontos.
- Mellette: ROC-AUC, **kalibráció** (decilisek, Brier-pontszám), és **találati arány a legkockázatosabb 10%-ban** (a műveleti központ kapacitása).
- **Döntés:** költségalapú küszöb (egy kihagyott késés X, egy téves riasztás Y dollár; érzékenység több arányra).
- A pontosságot (accuracy) nem használjuk: a „semmi sem késik” modell is magas értéket érne el.

**Mellékkérdés (regresszió):** MAE (fő), RMSE, és arányos hiba a késő járatokon.

**Elfogadási szabály** (ugyanaz, mint az 5. házinál): egy változtatás akkor számít javulásnak, ha **minden foldon** javít, és az átlagos javulás **nagyobb, mint a foldok közti szórás**.

---

## 5. Split és kiértékelés

| Rész | Időszak | Mire |
|---|---|---|
| Tanító + validáció | 2015. jan.–szept. | EDA, modellezés, minden döntés |
| Rés | október | kimarad (számkódok, és egy hónapnyi távolság a teszttől) |
| **Zárolt teszt** | **2015. nov.–dec.** | egyszer, a végén; ünnepi szezon: nehéz, de valósághű |

- **Foldok:** kiterjesztett ablak, havi validációs blokkokkal (júl., aug., szept.); a tanító rész mindig a blokk előtti összes hónap.
- **Bootstrap a teszten:** **napi blokkokkal**, mert az egy napon belüli járatok hibája együtt mozog (közös időjárás, közös késés-lánc).
- **Az EDA is csak a tanító időszakon** készül.

---

## 6. EDA — kérdés → döntés

Minden ábra egy döntést készít elő.

| # | Kérdés | Ábra | Döntés |
|---|---|---|---|
| 1 | Hogyan oszlik el az érkezési késés? Hány százalék késik 15 percnél többet? | hisztogram (levágva) | a cél definíciója; az osztályarány → PR-AUC |
| 2 | Mi okozza a késéseket? | a késett percek megoszlása okonként, havonta | mennyit adhat az időjárás és a gép előző járata |
| 3 | Napszak: nő-e a késés a nap folyamán? | késési arány az indulás órája szerint | az óra jellemző |
| 4 | Szezon és ünnepek | napi késési arány idősorként | split; ünnep-jellemző |
| 5 | A hét napjai | késési arány naponként | jellemző |
| 6 | Mely repülőterek a legrosszabbak? | térkép: pont mérete = forgalom, színe = késési arány | reptér-jellemző, célkódolás foldon belül |
| 7 | Zsúfoltság | késési arány a reptér óránkénti forgalma szerint | forgalom-jellemző |
| 8 | **Továbbgyűrűzés:** viszi-e a gép a késést? | az előző járat késése vs. a mostani | **a legerősebb jellemző** |
| 9 | Fordulóidő | késés-átvitel a fordulóidő szerint | fordulóidő-jellemző |
| 10 | Törlések és elterelések | arány és ok havonta | kizárás a fő elemzésből, külön említés |
| 11 | Adatminőség | hiányzó lajstromjel, októberi kódok, `2400` | tisztítási szabályok |
| 12 | Időjárás (ha összekapcsoltuk) | késési arány zivatarral, hóval, köddel és nélkülük | időjárás-jellemzők; hipotézisvizsgálat |

---

## 7. Modell-létra

1. **Alapráta:** mindenkinek ugyanaz a valószínűség (a tanító időszak késési aránya).
2. **Naiv szabályok:** (a) az útvonal történeti késési aránya; (b) „késik, ha a gép előző járata késett”.
3. **Logisztikus regresszió**, tisztességesen beállítva: kategóriák dummyként, skálázva, foldon belül.
4. **Random Forest** és **XGBoost**, ugyanazokkal a jellemzőkkel.
5. **+ a gép előző járata** (továbbgyűrűzés).
6. **+ időjárás** (cutoff-kori) → és a „tökéletes” időjárás mint felső korlát.

**Szándékosan hibás kontroll:** egy modell a `DEPARTURE_DELAY`-jel. A „túl jó” szám (szivárgás) megmutatja, miért nem szabad használni.

---

## 8. Diagnosztika, visszacsatolás, teszt

- **Diagnosztika** a legjobb modell out-of-fold jóslatain: kalibráció (decilisek), hiba napszak, reptér és hónap szerint, a hibák időbeli autokorrelációja (napi szinten), mely késéseket hagyja ki (ok szerint, a `*_DELAY` oszlopokkal).
- **Visszacsatolási kör:** a diagnosztikából egy hipotézis (például „a késés-láncot rosszul becsli”) → egy új jellemző → páros különbség foldonként ± szórás.
- **Záró teszt egyszer:** nov.–dec., napi blokk-bootstrap konfidenciaintervallummal, összevetve az alapmodellel és a naiv szabállyal.
- **Üzleti zárás:** a költségalapú küszöbnél hány késést fogunk meg, és mennyi téves riasztás árán; mennyit ér az időjárás-előrejelzés.

---

## 9. Kockázatok

| Kockázat | Kezelés |
|---|---|
| Lassú tanítás (~600 ezer tanító sor) | a kísérletek mintán futnak (~150 ezer sor), korai leállítás; a végső modell a teljes adaton |
| Memória | darabos beolvasás, csak a szükséges oszlopok, kisebb adattípusok, mentés pickle-be |
| Időzónák, nyári időszámítás | koordinátákból `timezonefinder`; a nyári időszámítás váltásainál óvatosan |
| Gyenge eredmény (a késés nehezen jósolható) | nem kudarc: az érvelés számít, és a „mennyit lehet kihozni, és miért” a jó válasz |
| Csak egy év adat | a szezonalitás egyszer látszik → korlát, kimondva |

---

## 10. Fájlok és ütemezés

Repó: https://github.com/csudtkamate-png/data-science-course-all (a `final_project` mappa)

```
final_project/
├── .gitignore              ← a data/ mappa és a gyorsítótárak nem kerülnek a repóba
├── projektterv.md          ← ez a fájl
├── 01_eda.ipynb            ← adat-előkészítés + EDA (elkészült, az adatra vár)
├── 02_idojaras.ipynb       ← időjárás letöltése és összekapcsolása
├── 03_modell.ipynb         ← split, modell-létra, diagnosztika, visszacsatolás, teszt
└── data/                   ← flights.csv, airports.csv, airlines.csv (a Kaggle-ről)
```

| Lépés | Állapot |
|---|---|
| Terv | ✅ |
| EDA notebook | ✅ megírva, kitalált mintán tesztelve; az igazi adatra vár |
| Adat letöltése a Kaggle-ről | ⏳ bejelentkezés kell |
| Időjárás | ⏳ |
| Modellezés | ⏳ |
