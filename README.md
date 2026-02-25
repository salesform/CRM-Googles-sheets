# CRM-Googles-sheets
Teljeskörű CRM több munkatárssal - Google App script + sheets

# Webshop Retention & Upsell Rendszer – Google Apps Script Dokumentáció

Ez a rendszer két Google Apps Script fájlból áll, amelyek két különböző Google Spreadsheet között szinkronizálnak értékesítési adatokat. A cél, hogy az értékesítők egy saját munkafüzetben lássák a megmenthető ügyfeleket, és ott kezeljék a státuszokat és jutalékokat.

---

## Rendszer áttekintés

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│     FORRÁS Spreadsheet      │         │      CÉL Spreadsheet        │
│  (webshop rendelési adatok) │──────►  │  (értékesítői munkafüzet)   │
│                             │         │                             │
│  Sikeres rendelések_DÁTUM   │         │  Sikeres rendelések         │
│  Sikeres előfizetések_DÁTUM │         │  Lejárt előfizetések        │
│  Sikertelen rendelések_DÁTUM│         │  Törölt előfizetések        │
│  Lejárt előfizetések_DÁTUM  │         │  Sikertelen rendelések      │
│  Törölt előfizetések_DÁTUM  │         │  Upsell                     │
└─────────────────────────────┘         │  Termék_szűrő               │
                                        └─────────────────────────────┘
```

Több értékesítő is dolgozhat egyszerre, mindenki saját cél munkafüzettel. A rendszer automatikusan kezeli, hogy egy ügyfelet csak egy értékesítő „foglalhat le" egyszerre.

---

## 1. Forrás Script (`forrás_script.gs`)

### Feladata
CSV fájlokat tölt le három webshop platformról és feldolgozza őket a forrás Spreadsheetbe. Ez a script fut először – az adatokat ez tölti fel, amelyekből a cél script dolgozik.

### Beállítások

A script tetején a `processCSVsWithSubscriptionLogic()` függvényen belül található `config` objektumban adhatók meg
Ha több Salesform szoftvert használsz, akkor mindegyiket külön sorba

```javascript
var config = {
  orders: [
    "https://elso.salesform.hu/index/csv?key=...",   // 1. SalesForm rendelések
    "https://masodik.salesform.hu/index/csv?key=...",       // 2. SalesForm rendelések
    "https://harmadik.salesform.hu/index/csv?key=..."          // 3. SalesForm rendelések
  ],
  subscriptions: [
    "https://f.bartfaibalazs.hu/index/csv?rec&key=...", // 1. SalesForm előfizetések
    "https://masodik.salesform.hu/index/csv?key=...",       // 2. SalesForm előfizetések
    "https://harmadik.salesform.hu/index/csv?key=..."          // 3. SalesForm előfizetések
    ...
  ],
  chunkSize: 500  // egyszerre írandó sorok száma
};
```

### Hogyan indítsd el

1. Nyisd meg a forrás Spreadsheedet
2. Kattints: **Bővítmények → Apps Script**
3. Illeszd be a `forrás_script.gs` tartalmát
4. Futtasd a `processCSVsWithSubscriptionLogic` függvényt

### Mit csinál futáskor

- Letölti a CSV fájlokat az összes URL-ről
- Szűri a sorokat státusz alapján:
  - `"sikeres"` státuszú rendelések → **Sikeres rendelések_DÁTUM** munkalap
  - `"sikertelen"` státuszú rendelések → **Sikertelen rendelések_DÁTUM** munkalap
- Az előfizetéseknél:
  - Aktív, érvényes dátumú sorok → **Sikeres előfizetések_DÁTUM**
  - Lejárt dátumú sorok → **Lejárt előfizetések_DÁTUM**
  - `"Lemondott"` értékű sorok → **Törölt előfizetések_DÁTUM**
- A munkalapok neve minden futáskor az **aktuális dátummal** frissül (pl. `Sikeres rendelések_2026-02-25`)
- Csak új azonosítókat ír be – a már meglévő sorokat nem írja felül
- Minden sorhoz hozzáad egy üres **Tulajdonos** oszlopot (Z oszlop), amelyet a cél script tölt ki

### Forrás munkalapok oszlopszerkezete

| Munkalap | Dátum oszlop | Termék oszlop |
|---|---|---|
| Sikeres rendelések | L (12.) | M (13.) |
| Sikeres előfizetések | M (13.) | N (14.) |
| Sikertelen rendelések | K (11.) | L (12.) |
| Lejárt előfizetések | M (13.) | N (14.) |
| Törölt előfizetések | M (13.) | N (14.) |

> **Megjegyzés:** A Tulajdonos oszlop (Z, 26.) automatikusan kerül kitöltésre a cél script által, amikor egy értékesítő státuszt állít be. Ez jelzi, hogy melyik értékesítői munkafüzet „foglalta le" az adott ügyfelet.

---

## 2. Cél Script (`cel_script.gs`)

### Feladata
A forrás Spreadsheetből szinkronizál adatokat a saját cél munkafüzetbe, ahol az értékesítők dolgoznak. Kezeli a státuszokat, jutalékokat, és megakadályozza, hogy két értékesítő egyszerre dolgozzon ugyanazon az ügyfélen.

### Első beállítás

A script tetején két dolgot kell kötelezően beállítani:

```javascript
function getSpreadsheetIds() {
  return {
    sourceSpreadsheetId: "FORRÁS_SPREADSHEET_ID",  // ← forrás fájl ID-ja
    targetSpreadsheetId: "CÉL_SPREADSHEET_ID"       // ← saját cél fájl ID-ja
  };
}
```

> Az ID a Google Sheets URL-jéből olvasható ki:
> `https://docs.google.com/spreadsheets/d/`**`EZ_AZ_ID`**`/edit`

### Jutalék beállítása

```javascript
// Típus: "fixed" = fix összeg | "percentage" = százalék
function getCommissionType() { return "percentage"; }

// Fix összeg (csak ha getCommissionType = "fixed")
function getCommissionAmount() { return 10000; } // 10 000 Ft

// Százalék (csak ha getCommissionType = "percentage")
function getCommissionRate() { return 15; } // 15%
```

### Hogyan indítsd el

1. Nyisd meg a saját cél Spreadsheedet
2. Kattints: **Bővítmények → Apps Script**
3. Illeszd be a `cel_script.gs` tartalmát
4. Állítsd be a két Spreadsheet ID-t (lásd fent)
5. Futtasd egyszer a `runAll` függvényt manuálisan az első szinkronizáláshoz
6. Állíts be **időalapú triggert** a `runAll` függvényre (ajánlott: naponta egyszer reggel)
7. Állíts be **telepített triggert** az `onEditInstallable` függvényre (lásd lent)

### `runAll()` – Szinkronizálás

A fő szinkronizáló függvény, amely az összes munkalapot egyszerre frissíti:

```
runAll()
  ├── syncSuccessfulOrders()       → Sikeres rendelések munkalap
  ├── syncExpiredSubscriptions()   → Lejárt előfizetések munkalap
  ├── syncCanceledSubscriptions()  → Törölt előfizetések munkalap
  ├── syncFailedOrders()           → Sikertelen rendelések munkalap (csak 60 napos)
  └── syncProductsToUpsell()       → Upsell munkalap + Termék_szűrő frissítése
```

**Fontos:** Minden szinkronizáló függvény csak az **új azonosítókat** adja hozzá – a már meglévő sorokat (és az ott beállított státuszokat, jutalékokat) soha nem írja felül.

Az adatok minden munkalapon **dátum szerint csökkenő sorrendben** jelennek meg – a legfrissebb sor mindig az első.

### Munkalaponkénti részletek

#### Sikeres rendelések
Az összes sikeres rendelést tartalmazza. Nincs időbeli szűrés – minden sor átkerül.

#### Lejárt előfizetések
Azok az előfizetők, akiknek az előfizetése lejárt. Nincs időbeli szűrés.

#### Törölt előfizetések
Azok az előfizetők, akik lemondták az előfizetésüket.

#### Sikertelen rendelések
Csak az **elmúlt 60 napból** szinkronizál – ennél régebbi sikertelen rendelések nem kerülnek át, mivel valószínűleg már nem menthetők meg.

#### Termék_szűrő munkalap
Automatikusan feltöltődik az összes forrás munkalapból összegyűjtött egyedi terméknévvel. Oszlopok:

| Termék | Db | Kiválasztva |
|---|---|---|
| Termék neve | Összes rendelés száma | ☐ checkbox |

- A **Db** oszlop minden futáskor frissül
- A **Kiválasztva** oszlopban pipálhatod be, mely termékek vásárlói kerüljenek az Upsell munkalapra
- A pipák megmaradnak futások között

#### Upsell munkalap
Csak azok a vásárlók kerülnek ide, akik a Termék_szűrőben bepipált termékeket vásárolták, és a vásárlás az **elmúlt 60 napban** történt.

### Célmunkalapok oszlopszerkezete

Minden munkalapon egységes fejléc:

| # | Oszlop | Leírás |
|---|---|---|
| 1 | Azonosító | Egyedi rendelésazonosító |
| 2 | Név | Vásárló neve |
| 3 | E-mail | Vásárló e-mail címe |
| 4 | Telefonszám | Vásárló telefonszáma |
| 5 | Dátum | Rendelés/előfizetés dátuma |
| 6 | Végösszeg | Rendelés összege |
| 7 | Termék | Vásárolt termék neve |
| 8 | BUMP | Order bump termék (ha volt) |
| 9 | Státusz | Legördülő lista (értékesítő tölti ki) |
| 10 | Jutalék | Automatikusan számított jutalék |

### Státusz lehetőségek

A 9. oszlopban legördülő listából választható:

| Státusz | Sor színe | Jutalék |
|---|---|---|
| Válassz | Fehér | – |
| Időpont küldve | Sárga | – |
| Időpont emlékeztető | Narancssárga | – |
| Foglalt | Kék | – |
| **Megmentve** | **Zöld** | **✓ Számítódik** |
| Elvesztett | Piros | – |

### `onEditInstallable` – Automatikus státuszkezelés

> ⚠️ Ezt **telepített triggerként** kell beállítani, nem egyszerű `onEdit`-ként, különben nincs jogosultsága más Spreadsheetet írni.

**Beállítás:**
1. Apps Script szerkesztőben: **Triggerek → Trigger hozzáadása**
2. Függvény: `onEditInstallable`
3. Esemény: **Spreadsheet módosításakor**

**Mit csinál automatikusan, ha a Státusz oszlopot módosítod:**

1. **Sor kiszínezése** – az egész sor a státusznak megfelelő színt kap
2. **Jutalék számítása** – ha „Megmentve" státuszt állítasz be:
   - `percentage` módban: `Végösszeg × jutalék%`
   - `fixed` módban: fix összeg
   - Bármilyen más státusznál a jutalék törlődik
3. **Tulajdonos visszaírása** – a forrás Spreadsheet Z oszlopába beírja a cél munkafüzet URL-jét, jelezve hogy ez az értékesítő „foglalta le" az ügyfelet
4. **Ütközés-kezelés** – lefuttatja a `cleanTargetSpreadsheet()` függvényt, amely törli azokat a sorokat a cél munkafüzetből, ahol a forrásban már más értékesítő van tulajdonosként megjelölve

### Időalapú trigger beállítása

A `runAll` függvényt érdemes automatikusan futtatni, hogy mindig friss adatok legyenek:

1. Apps Script szerkesztőben: **Triggerek → Trigger hozzáadása**
2. Függvény: `runAll`
3. Eseményforrás: **Időalapú**
4. Típus: **Nap timer** → pl. reggel 7–8 között

---

## Több értékesítő esetén

Ha több értékesítő dolgozik a rendszerrel, **mindenki saját cél Spreadsheetel** rendelkezik. A lépések:

1. Másold le a cél Spreadsheet sablont minden értékesítőnek
2. Minden másolatban állítsd be a `targetSpreadsheetId`-t az adott fájl ID-jára
3. A `sourceSpreadsheetId` mindenhol ugyanaz marad
4. Minden értékesítő beállítja a saját triggereit

A rendszer automatikusan kezeli az ütközéseket: ha egy értékesítő „Megmentve"-re állít egy ügyfelet, az a forrásban foglaltnak jelölődik, és a többi értékesítő munkafüzetéből automatikusan törlődik.

---

## Hibaelhárítás

**Nem kerülnek át az adatok:**
- Ellenőrizd, hogy a forrás munkalap neve tartalmazza-e a mai dátumot (pl. `Sikeres rendelések_2026-02-25`)
- A script prefix alapján keres – ha a munkalap neve nem stimmel, nem találja meg

**Az `onEditInstallable` nem működik:**
- Egyszerű `onEdit`-ként nem fut le más Spreadsheet írásához – kötelező telepített triggerként beállítani

**Jutalék nem számolódik:**
- Ellenőrizd, hogy a Végösszeg cellában szám van-e (ne legyen üres vagy 0)
- A jutalék csak „Megmentve" státusznál számítódik

**Termék_szűrőben hiányoznak termékek:**
- Futtasd le manuálisan a `syncProductsToUpsell()` függvényt – ez frissíti a szűrőlapot az összes forrás munkalapból
