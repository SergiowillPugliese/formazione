# 02 — Nullable types

## Il problema

In JavaScript/TypeScript, ogni variabile può essere `null` o `undefined` quasi sempre.
C# è più rigido: un tipo dichiarato come `string` NON può essere null.
Se vuoi permettere null, devi dichiararlo esplicitamente con `?`.

---

## Il simbolo `?` dopo il tipo

```csharp
string nome = "Mario";    // non può essere null — il compilatore ti avverte
string? nomeOpzionale = null; // può essere null — esplicitamente dichiarato

int eta = 30;             // non può essere null
int? etaOpzionale = null; // può essere null
```

In TypeScript esiste lo stesso concetto:
```typescript
let nome: string = "Mario";      // non nullable
let nomeOpzionale: string | null = null; // nullable
// oppure con strictNullChecks
let nomeOpzionale?: string;
```

---

## Come usare un valore nullable

Quando hai un tipo nullable, non puoi usarlo direttamente senza prima verificare
che non sia null. Il compilatore ti obbliga a farlo.

### `.HasValue` e `.Value` — per tipi di valore (int?, long?, DateTime?, decimal?)

```csharp
long? agencyRequestingUserId = cr.AgencyRequestingUserId;

// Prima controlla se c'è un valore
if (agencyRequestingUserId.HasValue)
{
    // .Value ti dà il valore reale — sicuro perché hai già verificato
    long userId = agencyRequestingUserId.Value;
    Console.WriteLine(userId);
}
```

`.HasValue` è equivalente al controllo `!= null` in TypeScript:
```typescript
if (agencyRequestingUserId !== null && agencyRequestingUserId !== undefined) {
    console.log(agencyRequestingUserId);
}
```

**Nota**: `.HasValue` e `.Value` funzionano solo su tipi di valore nullable
(`int?`, `long?`, `DateTime?`, `decimal?`). NON funzionano su `string?`.
Per `string?` usi direttamente `!= null`.

---

## L'operatore `?.` — null conditional

Permette di accedere a una property o chiamare un metodo solo se l'oggetto non è null.
Se è null, il risultato è null invece di un'eccezione.

```csharp
AgencyInfoModel? agencyInfo = null;

// SENZA ?. — potrebbe esplodere se agencyInfo è null
string codice = agencyInfo.Code; // NullReferenceException!

// CON ?. — sicuro: se agencyInfo è null, restituisce null
string? codice = agencyInfo?.Code; // null, nessuna eccezione
```

Nel progetto lo abbiamo usato in `BuildRow`:
```csharp
var agencyInfo = cr.AgencyRequestingUserId.HasValue
    ? agencyInfoByUserId.GetValueOrDefault(cr.AgencyRequestingUserId.Value)
    : null;

// Se agencyInfo è null, AgencyCode e AgencyName diventano null
AgencyCode = agencyInfo?.Code,
AgencyName = agencyInfo?.Name,
```

In TypeScript è identico:
```typescript
const codice = agencyInfo?.code; // optional chaining
```

---

## L'operatore `??` — null coalescing

Fornisce un valore di default se l'espressione di sinistra è null.

```csharp
string? nomeOpzionale = null;
string nome = nomeOpzionale ?? "Sconosciuto"; // "Sconosciuto"

string? altroNome = "Mario";
string altroNomeReale = altroNome ?? "Sconosciuto"; // "Mario"
```

In TypeScript è identico:
```typescript
const nome = nomeOpzionale ?? "Sconosciuto";
```

Nel progetto lo usiamo in `SetStringCell`:
```csharp
cell.SetCellValue(value ?? string.Empty); // se null, scrivi stringa vuota
```

---

## Il formato dei DateTime nullable

Quando formattiamo una data nullable per l'Excel, usiamo `?.ToString(...)`:

```csharp
// Se CommercialApprovalAt è null, l'espressione intera diventa null
// Se non è null, formatta la data come "2024-12-25 14:30"
row.CommercialApprovalAt?.ToString("yyyy-MM-dd HH:mm")
```

Il formato `"yyyy-MM-dd HH:mm"` è il pattern di formattazione:
- `yyyy` = anno a 4 cifre
- `MM` = mese a 2 cifre
- `dd` = giorno a 2 cifre
- `HH` = ore in formato 24h
- `mm` = minuti

---

## Pattern comune nel progetto

Quasi ogni property dei campi di approvazione è nullable,
perché una CR appena creata non ha ancora nessuna approvazione:

```csharp
// Sull'entità
public DateTime? CommercialApprovalAt { get; set; }   // null se non approvata
public long? CommercialApprovalByUserId { get; set; } // null se non approvata

// Nel BuildRow
CommercialApprovalAt = cr.CommercialApprovalAt,  // passa null direttamente, ok
CommercialApproverName = cr.CommercialApprovalByUserId.HasValue
    ? usersLookup.GetValueOrDefault(cr.CommercialApprovalByUserId.Value)
    : null,
// "se CommercialApprovalByUserId ha un valore, cercalo nel lookup, altrimenti null"
```
