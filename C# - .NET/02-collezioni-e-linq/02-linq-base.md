# 08 — LINQ: lavorare con le collezioni

## Cos'è LINQ

LINQ (Language Integrated Query) è un insieme di metodi per trasformare,
filtrare e aggregare collezioni. È l'equivalente dei metodi `Array` in JavaScript:
`.map()`, `.filter()`, `.reduce()`, `.find()`, ecc.

La differenza: in C# questi metodi sono su qualsiasi collezione (`List`, array,
risultati di query EF Core), non solo sugli array.

---

## `.Select()` — trasformare elementi (come `.map()`)

```csharp
var nomi = new List<string> { "mario", "luigi" };

// Trasforma ogni elemento
var nomiMaiuscoli = nomi.Select(n => n.ToUpper()).ToList();
// ["MARIO", "LUIGI"]
```

```javascript
// JavaScript equivalente
const nomiMaiuscoli = nomi.map(n => n.toUpperCase());
```

### Nel progetto

```csharp
// Trasforma ogni CR in una riga del report
var rows = customizationRequests
    .Select(cr => BuildRow(cr, usersLookup, customersLookup, posLookup, agencyInfoByUserId, storeChannelInfoByChannelId[cr.ChannelId]))
    .ToList();
```

---

## `.Where()` — filtrare (come `.filter()`)

```csharp
var numeri = new List<int> { 1, 2, 3, 4, 5 };

var pari = numeri.Where(n => n % 2 == 0).ToList();
// [2, 4]
```

```javascript
const pari = numeri.filter(n => n % 2 === 0);
```

---

## `.ToDictionary()` — costruire una mappa

Trasforma una lista in un Dictionary (Map) usando una funzione per la chiave
e una per il valore.

```csharp
var utenti = new List<UserInfoModel> { /* ... */ };

// chiave: u.Id, valore: u.Name
var usersLookup = utenti.ToDictionary(u => u.Id, u => u.Name);

// chiave: u.Id, valore: l'intero oggetto
var usersByIdLookup = utenti.ToDictionary(u => u.Id);
```

```typescript
// TypeScript
const usersLookup = new Map(utenti.map(u => [u.id, u.name]));
```

---

## `.Any()` — esiste almeno un elemento? (come `.some()`)

```csharp
var lista = new List<int> { 1, 2, 3 };

bool haElementi = lista.Any();              // true — ha almeno un elemento
bool haPari = lista.Any(n => n % 2 == 0);  // true — ha almeno un pari
bool haVuoto = new List<int>().Any();       // false — lista vuota
```

```javascript
const haElementi = lista.length > 0;
const haPari = lista.some(n => n % 2 === 0);
```

### Nel progetto

```csharp
if (!customizationRequests.Any())
{
    // nessuna CR trovata — ritorna Excel vuoto
    return GenerateExcelWorkbook(new List<CustomizationRequestReportRow>());
}
```

---

## `.First()` e `.FirstOrDefault()` — primo elemento

```csharp
var lista = new List<int> { 10, 20, 30 };

int primo = lista.First();                        // 10
int primoMaggioreDi15 = lista.First(n => n > 15); // 20

// Se non trova nulla: First() lancia eccezione, FirstOrDefault() ritorna il default
int? nonEsiste = lista.FirstOrDefault(n => n > 100); // 0 (default di int)
// per tipi reference come string: null
```

```javascript
const primo = lista[0];
const primoMaggioreDi15 = lista.find(n => n > 15);
```

---

## `.GroupBy()` — raggruppare per chiave

Raggruppa gli elementi per un criterio comune, restituendo gruppi.

```csharp
var results = new List<(long PosId, string AgencyName)>
{
    (1, "Agenzia A"),
    (2, "Agenzia B"),
    (1, "Agenzia C"), // stesso PosId di prima
};

// Raggruppa per PosId
var gruppi = results.GroupBy(r => r.PosId);
// Gruppo PosId=1: [(1, "Agenzia A"), (1, "Agenzia C")]
// Gruppo PosId=2: [(2, "Agenzia B")]

// Prendi solo il primo elemento di ogni gruppo
var uniPerPos = results
    .GroupBy(r => r.PosId)
    .Select(g => g.First())
    .ToList();
// [(1, "Agenzia A"), (2, "Agenzia B")]
```

```javascript
// JavaScript — non c'è GroupBy nativo, si fa manualmente
const grouped = results.reduce((acc, r) => {
    if (!acc[r.posId]) acc[r.posId] = [];
    acc[r.posId].push(r);
    return acc;
}, {});
```

---

## `.Sum()` — somma (come `.reduce()` per somme)

```csharp
var numeri = new List<int> { 1, 2, 3, 4 };
int totale = numeri.Sum(); // 10

// Con selettore
var prodotti = new List<CustomizedProduct> { /* ... */ };
int totalePezzi = prodotti.Sum(p => p.Sizes.Sum(s => s.RequestedQuantity));
// somma le quantità di tutte le taglie di tutti i prodotti
```

### Nel progetto

```csharp
var totalQuantity = cr.CustomizedProducts
    .Sum(cp => cp.Sizes
        .Sum(s => s.RequestedQuantity));
// per ogni prodotto personalizzato, somma le quantità delle taglie
```

---

## `.Distinct()` — rimuovere duplicati

```csharp
var ids = new List<long> { 1, 2, 1, 3, 2 };
var unici = ids.Distinct().ToList(); // [1, 2, 3]
```

---

## Concatenare operazioni — method chaining

Il vero potere di LINQ è concatenare operazioni, come in JavaScript:

```csharp
var result = customizationRequests
    .Where(cr => cr.Status == CustomizationRequestStatus.Draft)  // filtra
    .Select(cr => cr.Title)                                       // trasforma
    .Distinct()                                                   // rimuovi duplicati
    .OrderBy(title => title)                                      // ordina
    .ToList();                                                    // materializza
```

```javascript
const result = customizationRequests
    .filter(cr => cr.status === "Draft")
    .map(cr => cr.title)
    .filter((title, index, arr) => arr.indexOf(title) === index) // distinct
    .sort()
```

---

## Lazy evaluation — attenzione al `.ToList()`

Le operazioni LINQ sono **lazy**: non vengono eseguite finché non le "materializzi".
`.ToList()` forza l'esecuzione e crea la lista in memoria.

```csharp
// Questa riga NON esegue ancora nulla — è solo una "ricetta"
var query = customizationRequests.Where(cr => cr.Status == CustomizationRequestStatus.Draft);

// Questo esegue la ricetta e crea la lista
var lista = query.ToList();
```

Se non aggiungi `.ToList()`, la query viene rieseguita ogni volta che itero.
In EF Core (database), questo ha implicazioni importanti — ma per ora
la regola pratica è: **aggiungi sempre `.ToList()` alla fine**.
