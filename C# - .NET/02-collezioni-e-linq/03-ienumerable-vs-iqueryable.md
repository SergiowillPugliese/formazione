# 2.3 — IEnumerable vs IQueryable

## Il concetto più importante per le performance con EF Core

Questa distinzione è **critica**. Sbagliare qui significa caricare
migliaia di righe dal database quando ne servono 10.

---

## `IEnumerable<T>` — filtraggio in memoria

`IEnumerable<T>` rappresenta una sequenza di elementi **già in memoria**.
Quando ci applichi LINQ (`.Where()`, `.Select()`...), il filtro
avviene **dopo** che i dati sono stati caricati dal DB.

```csharp
// PERICOLOSO con EF Core
IEnumerable<CustomizationRequest> tutte = await _repo.GetListAsync();
// ↑ carica TUTTO dal database in memoria (es. 50.000 righe)

var approvate = tutte.Where(cr => cr.Status == CustomizationRequestStatus.CommerciallyApproved);
// ↑ filtra in memoria — ma i dati erano già tutti caricati
```

**Analogia**: è come scaricare tutto il catalogo Amazon sul tuo PC
e poi usare Ctrl+F per cercare scarpe rosse taglia 42.

---

## `IQueryable<T>` — filtraggio nel database

`IQueryable<T>` rappresenta una **query non ancora eseguita**.
Quando ci applichi LINQ, costruisce l'SQL ma non lo esegue ancora.
La query parte solo quando materializzi i risultati (`.ToList()`, `.FirstOrDefault()`...).

```csharp
// CORRETTO con EF Core
IQueryable<CustomizationRequest> query = _dbContext.CustomizationRequests;
// ↑ nessuna query al DB ancora — solo una "ricetta"

query = query.Where(cr => cr.Status == CustomizationRequestStatus.CommerciallyApproved);
// ↑ aggiunge WHERE alla ricetta — ancora nessuna query

var approvate = await query.ToListAsync();
// ↑ SOLO ORA parte la query SQL:
// SELECT * FROM CustomizationRequest WHERE Status = 5
```

**Analogia**: è come costruire un filtro di ricerca su Amazon
(categoria: scarpe, colore: rosso, taglia: 42) e premere "Cerca"
solo alla fine. Amazon esegue la ricerca sul server, non ti manda
tutto il catalogo.

---

## Come EF Core li usa

Quando usi EF Core correttamente, parti sempre da `IQueryable`:

```csharp
// GetQueryableAsync() restituisce IQueryable — ancora nessuna query
var queryable = await _customizationRequestRepo.GetQueryableAsync();

// Ogni metodo LINQ aggiunge condizioni alla query SQL
var results = await queryable
    .Where(cr => cr.TenantId == currentTenantId)        // WHERE TenantId = ...
    .Where(cr => cr.StoreId == request.StoreId)         // AND StoreId = ...
    .Include(cr => cr.CustomizedProducts)               // LEFT JOIN CustomizedProduct
    .OrderByDescending(cr => cr.CreationTime)           // ORDER BY CreationTime DESC
    .Take(50)                                           // TOP 50
    .ToListAsync(ct);                                   // ESEGUI ORA
```

SQL generato (approssimativamente):
```sql
SELECT TOP 50 * FROM CustomizationRequest cr
LEFT JOIN CustomizedProduct cp ON cp.CustomizationRequestId = cr.Id
WHERE cr.TenantId = @tenantId AND cr.StoreId = @storeId
ORDER BY cr.CreationTime DESC
```

---

## Il bug più comune: perdere l'IQueryable

Quando materializzi troppo presto, perdi il vantaggio:

```csharp
// BUG: .ToList() troppo presto
var tutte = await queryable.ToListAsync();      // carica TUTTO
var filtrate = tutte.Where(cr => cr.StoreId == 1); // filtra in memoria

// CORRETTO: filtra prima, poi materializza
var filtrate = await queryable
    .Where(cr => cr.StoreId == 1)
    .ToListAsync();                             // carica solo quelle di StoreId=1
```

---

## Come riconoscerli nel codice

| Tipo | Dove lo vedi | Quando filtra |
|---|---|---|
| `IQueryable<T>` | Da `GetQueryableAsync()`, DbSet | Nel database (SQL) |
| `IEnumerable<T>` | Da `GetListAsync()`, `.ToList()`, array | In memoria (C#) |
| `List<T>` | Dopo `.ToList()` / `.ToListAsync()` | Già in memoria |

---

## Riepilogo con analogia finale

Pensa a `IQueryable` come a un **Angular HttpParams builder**:

```typescript
// Angular — costruisci la query prima di fare la chiamata HTTP
let params = new HttpParams();
params = params.set('storeId', '1');
params = params.set('status', 'approved');
// La chiamata HTTP parte solo quando fai .pipe(take(1))
this.http.get('/api/orders', { params }).subscribe(...)
```

`IQueryable` in C# è lo stesso concetto: costruisci la "richiesta"
pezzo per pezzo, e la "mandi" (al database) solo alla fine con
`.ToListAsync()` o `.FirstOrDefaultAsync()`.
