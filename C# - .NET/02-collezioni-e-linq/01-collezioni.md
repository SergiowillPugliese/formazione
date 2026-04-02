# 07 — Collezioni: List, HashSet, Dictionary

## Il confronto con JavaScript/TypeScript

In JavaScript hai `Array` per quasi tutto. In C# esistono strutture dati
specializzate, ognuna ottimizzata per un uso specifico.

---

## `List<T>` — la lista ordinata

È come un `Array` in JavaScript: ordinata, può avere duplicati,
accesso per indice.

```csharp
// Creare una lista
var nomi = new List<string>();
var ids = new List<long>();

// Aggiungere elementi
nomi.Add("Mario");
nomi.Add("Luigi");
nomi.Add("Mario"); // i duplicati sono permessi

// Accesso per indice
string primo = nomi[0]; // "Mario"

// Lunghezza
int quanti = nomi.Count; // non .length come in JS, ma .Count
```

```typescript
// TypeScript equivalente
const nomi: string[] = [];
nomi.push("Mario");
const primo = nomi[0];
const quanti = nomi.length;
```

La `T` in `List<T>` si chiama **tipo generico** — dice al compilatore
che tipo di elementi contiene la lista. `List<string>` = lista di stringhe,
`List<long>` = lista di interi, `List<CustomizationRequest>` = lista di CR.

---

## `HashSet<T>` — la collezione senza duplicati

Un `HashSet` è una collezione non ordinata che **non ammette duplicati**.
Aggiungere lo stesso valore due volte non ha effetto.

```csharp
var userIds = new HashSet<long>();

userIds.Add(42);    // aggiunto
userIds.Add(99);    // aggiunto
userIds.Add(42);    // IGNORATO — già presente

// userIds contiene solo: { 42, 99 }
// Count = 2
```

```typescript
// TypeScript equivalente
const userIds = new Set<number>();
userIds.add(42);
userIds.add(99);
userIds.add(42); // ignorato
```

**Quando usarlo?** Quando vuoi raccogliere ID unici da una lista
senza dover controllare manualmente i duplicati.

### Nel progetto

```csharp
// Prima passata sulle CR — raccolta ID unici
var userIds = new HashSet<long>();
var agencyRequestingUserIds = new HashSet<long>();

foreach (var cr in customizationRequests)
{
    userIds.Add(cr.CreatorId);      // se già presente, ignorato
    userIds.Add(cr.DesignerUserId); // se già presente, ignorato

    if (cr.AgencyRequestingUserId.HasValue)
    {
        agencyRequestingUserIds.Add(cr.AgencyRequestingUserId.Value);
    }
}

// Risultato: tutti gli ID utente distinti presenti nelle CR
// Nessun duplicato — possiamo fare una singola query batch
```

---

## `Dictionary<TKey, TValue>` — la mappa chiave-valore

Un `Dictionary` è una collezione di coppie chiave-valore.
Corrisponde a `Map` in JavaScript o a un oggetto usato come mappa.

```csharp
// Creare un Dictionary
var usersLookup = new Dictionary<long, string>();

// Aggiungere
usersLookup[42] = "Mario Rossi";
usersLookup[99] = "Luigi Bianchi";

// Leggere
string nome = usersLookup[42];        // "Mario Rossi"
string? nome = usersLookup[999];      // KeyNotFoundException! — la chiave non esiste

// Leggere in modo sicuro
string? nome = usersLookup.GetValueOrDefault(999);  // null — nessuna eccezione
string nome = usersLookup.GetValueOrDefault(999, "Sconosciuto"); // fallback
```

```typescript
// TypeScript equivalente
const usersLookup = new Map<number, string>();
usersLookup.set(42, "Mario Rossi");
const nome = usersLookup.get(42);      // "Mario Rossi" | undefined
const nome = usersLookup.get(999);     // undefined — nessuna eccezione
```

**`GetValueOrDefault`** è fondamentale: evita eccezioni quando una chiave
potrebbe non esistere. Nel progetto lo usiamo ovunque nel `BuildRow`:

```csharp
CustomerName = customersLookup.GetValueOrDefault(cr.CustomerId, cr.CustomerId.ToString()),
// "dammi il nome del cliente, se non c'è usa l'ID come stringa di fallback"
```

### Creare un Dictionary da una lista — `ToDictionary`

Il metodo LINQ `ToDictionary` trasforma una lista in un Dictionary:

```csharp
var usersInfo = new List<UserInfoModel> { /* ... */ };

// Costruisce Dictionary<long, string>: { userId → userName }
var usersLookup = usersInfo.ToDictionary(
    u => u.Id,    // keySelector: cosa diventa la chiave
    u => u.Name   // valueSelector: cosa diventa il valore
);
```

```typescript
// TypeScript equivalente
const usersLookup = new Map(usersInfo.map(u => [u.id, u.name]));
```

---

## `.ToList()` — convertire in lista

Molte operazioni LINQ restituiscono `IEnumerable<T>` (una sequenza lazy).
`.ToList()` la materializza in una `List<T>` effettiva in memoria:

```csharp
var customerIds = new HashSet<long> { 1, 2, 3 };

// GetCustomersInfoAsync vuole List<long>, non HashSet<long>
var customersInfo = await _catalogDataProxy.GetCustomersInfoAsync(
    customerIds.ToList(), // converte HashSet → List
    ct
);
```

---

## Riepilogo: quando usare quale?

| Struttura | Uso | Analogo JS |
|---|---|---|
| `List<T>` | Lista ordinata, con duplicati, accesso per indice | `Array` |
| `HashSet<T>` | Insieme di valori unici, senza ordine | `Set` |
| `Dictionary<K,V>` | Lookup chiave → valore | `Map` / oggetto `{}` |
