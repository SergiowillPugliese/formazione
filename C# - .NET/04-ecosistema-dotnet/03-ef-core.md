# 4.3 — EF Core

## Cos'è EF Core

**Entity Framework Core** (EF Core) è l'ORM (Object-Relational Mapper)
standard di .NET. Traduce tra il mondo degli oggetti C# e il mondo
delle tabelle SQL.

**ORM** significa: scrivi codice C# invece di SQL, e EF Core genera
il SQL per te.

**Analogia Angular**: pensa a EF Core come a un service Angular che
fa chiamate HTTP al database invece che a un'API REST. Tu chiami
metodi C#, lui parla SQL.

---

## Il `DbContext` — il cuore di EF Core

`DbContext` è la classe che rappresenta una sessione con il database.
Contiene i `DbSet<T>` — uno per ogni tabella.

```csharp
// In Catalog.Domain/EntityFrameworkCore/CatalogDbContext.cs
public class CatalogDbContext : AbpDbContext<CatalogDbContext>
{
    // Ogni DbSet<T> = una tabella nel database
    public DbSet<CustomizationRequest> CustomizationRequests { get; set; }
    public DbSet<Agency> Agencies { get; set; }
    public DbSet<AgencyStore> AgencyStores { get; set; }
    public DbSet<Order> Orders { get; set; }
}
```

**Analogia**: `DbContext` è come un `HttpClient` configurato — sai
già a quale "server" (database) parla. `DbSet<T>` è come un endpoint
specifico — `/api/customization-requests`, `/api/agencies`, ecc.

---

## Leggere dati: query base

```csharp
// Tutti i record (ATTENZIONE: carica tutto in memoria)
var tutte = await _dbContext.CustomizationRequests.ToListAsync();

// Con filtro WHERE
var approvate = await _dbContext.CustomizationRequests
    .Where(cr => cr.Status == CustomizationRequestStatus.CommerciallyApproved)
    .ToListAsync();

// Singolo record per ID
var cr = await _dbContext.CustomizationRequests
    .FirstOrDefaultAsync(cr => cr.Id == id);
// cr può essere null se non trovato

// Come sopra ma lancia eccezione se null
var cr = await _dbContext.CustomizationRequests
    .FirstAsync(cr => cr.Id == id);
```

---

## `Include` — caricare le relazioni (JOIN)

Le entità hanno relazioni tra loro. Per default EF Core non carica
le entità collegate — devi esplicitarlo con `Include`.

```csharp
// Senza Include: cr.CustomizedProducts è null o vuoto
var cr = await _dbContext.CustomizationRequests
    .FirstAsync(cr => cr.Id == id);

// Con Include: carica anche i prodotti personalizzati
var cr = await _dbContext.CustomizationRequests
    .Include(cr => cr.CustomizedProducts)        // JOIN con CustomizedProduct
    .FirstAsync(cr => cr.Id == id);

// ThenInclude: include annidata (JOIN su JOIN)
var cr = await _dbContext.CustomizationRequests
    .Include(cr => cr.CustomizedProducts)
        .ThenInclude(cp => cp.Sizes)            // JOIN anche con Size
    .FirstAsync(cr => cr.Id == id);
```

SQL generato approssimativamente:
```sql
SELECT * FROM CustomizationRequest cr
LEFT JOIN CustomizedProduct cp ON cp.CustomizationRequestId = cr.Id
LEFT JOIN CustomizedProductSize s ON s.CustomizedProductId = cp.Id
WHERE cr.Id = @id
```

---

## `AsNoTracking` — per le letture

Per default, EF Core "traccia" gli oggetti che carica — tiene in
memoria le modifiche per salvare poi con `SaveChanges()`.

Per le query di sola lettura (report, API GET), questo tracking
è inutile e spreca memoria. Disabilitalo con `AsNoTracking()`:

```csharp
var crs = await _dbContext.CustomizationRequests
    .AsNoTracking()           // ← disabilita il tracking
    .Where(cr => cr.StoreId == storeId)
    .ToListAsync();
```

**Regola nel progetto**: tutte le query in `GetQueryableAsync()` usano
`AsNoTracking()` per le letture. Lo vedi nella convenzione documentata
in `CLAUDE.md`.

---

## `Select` — proiezione (evita di caricare tutto)

A volte non ti servono tutti i campi di un'entità. `.Select()` permette
di caricare solo quello che ti serve — EF Core genera un SELECT
con solo quelle colonne.

```csharp
// Carica TUTTE le colonne di Agency (spreco se ne vuoi solo 2)
var agenzie = await _dbContext.Agencies
    .Select(a => new { a.Id, a.Name })  // solo Id e Name
    .ToListAsync();
```

SQL generato:
```sql
SELECT Id, Name FROM Agency   -- non SELECT *
```

---

## Scrivere dati

```csharp
// INSERT
var nuovaCr = new CustomizationRequest { Title = "Test" };
await _dbContext.CustomizationRequests.AddAsync(nuovaCr);
await _dbContext.SaveChangesAsync();  // ← esegue l'INSERT

// UPDATE (il tracking serve qui)
var cr = await _dbContext.CustomizationRequests.FirstAsync(cr => cr.Id == id);
cr.Title = "Nuovo titolo";  // modifica l'oggetto
await _dbContext.SaveChangesAsync();  // ← esegue l'UPDATE

// DELETE (soft delete nel progetto — non elimina davvero)
_dbContext.CustomizationRequests.Remove(cr);
await _dbContext.SaveChangesAsync();
```

**Nota sul progetto**: le entità usano **soft delete** — non vengono
mai eliminate fisicamente dal DB. EF Core aggiunge automaticamente
`WHERE IsDeleted = 0` a tutte le query. Lo vedi in `CLAUDE.md`.

---

## Migrations — versionare lo schema del DB

Una **migration** è un file C# che descrive una modifica allo schema
del database (aggiungere una tabella, una colonna, un indice...).

```
# Workflow migrations nel progetto
./scripts/ef.ps1 -create -project catalog -migration "AddWholesalePriceToOrder"
# ↑ genera un file C# con le istruzioni Up() e Down()

./scripts/ef.ps1 -update -project catalog -migration ""
# ↑ applica le migrations pendenti al DB
```

`Up()` = applica la migrazione (aggiunge la colonna)
`Down()` = fa il rollback (rimuove la colonna)

**Analogia**: le migrations sono come i commit di git per lo schema
del database — ogni cambiamento è tracciato e reversibile.

---

## Riepilogo

| Concetto | Cosa fa |
|---|---|
| `DbContext` | Sessione con il database |
| `DbSet<T>` | Rappresenta una tabella |
| `.Where()` | Aggiunge WHERE (genera SQL) |
| `.Include()` | Carica relazioni (genera JOIN) |
| `.AsNoTracking()` | Lettura senza tracking — più veloce |
| `.Select()` | Proiezione — carica solo alcune colonne |
| `.ToListAsync()` | Esegue la query e ritorna lista |
| `.FirstOrDefaultAsync()` | Esegue e ritorna il primo (o null) |
| `SaveChangesAsync()` | Salva INSERT/UPDATE/DELETE |
| Migration | File C# che descrive una modifica allo schema |
