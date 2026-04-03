# 2.4 — Ordinamento: OrderBy, ThenBy e casi con nullable

## Il problema di oggi

Abbiamo costruito un sistema di posizionamento per gli assortimenti.
La regola era:
1. Prima gli assortimenti **con posizione**, in ordine crescente (1, 2, 3…)
2. Poi quelli **senza posizione**, prima per `IsFeatured`, poi per `CreationTime`

Per farlo ci servivano `OrderBy`, `ThenBy`, e un trucco per gestire i valori `null`.

---

## `OrderBy` e `OrderByDescending` — la base

```csharp
// Ordine crescente (A → Z, 1 → 100)
var assortimenti = query.OrderBy(a => a.CreationTime);

// Ordine decrescente (Z → A, 100 → 1)
var assortimenti = query.OrderByDescending(a => a.CreationTime);
```

**Analogia Angular**: è come `array.sort((a, b) => a.value - b.value)` — ma eseguito
direttamente nel database, non in memoria JavaScript.

---

## `ThenBy` e `ThenByDescending` — il secondo criterio

Quando il primo criterio è uguale tra due elementi, entra in gioco il secondo.

```csharp
// Prima per IsFeatured (true prima), poi per CreationTime (più recente prima)
var assortimenti = query
    .OrderByDescending(a => a.IsFeatured)
    .ThenByDescending(a => a.CreationTime);
```

In SQL diventa:
```sql
ORDER BY IsFeatured DESC, CreationTime DESC
```

Puoi concatenare quanti `.ThenBy()` vuoi — come `ORDER BY` in SQL ha più colonne.

**Errore comune**: usare due `.OrderBy()` invece di `.OrderBy()` + `.ThenBy()`:
```csharp
// SBAGLIATO — il secondo OrderBy annulla il primo
query.OrderByDescending(a => a.IsFeatured)
     .OrderByDescending(a => a.CreationTime);  // ← sovrascrive tutto

// CORRETTO
query.OrderByDescending(a => a.IsFeatured)
     .ThenByDescending(a => a.CreationTime);
```

---

## Il problema dei valori nullable nell'ordinamento

`Position` è `int?` — può essere `null`. Se ordini direttamente per un campo nullable,
SQL Server mette i `NULL` **prima** degli altri valori (in ordine crescente).
Non è quello che vogliamo: gli assortimenti senza posizione devono venire **dopo**.

### Il trucco: campo calcolato per separare null dai non-null

```csharp
query
    .OrderBy(a => a.Position == null ? 1 : 0)   // 0 = ha posizione (va prima), 1 = null (va dopo)
    .ThenBy(a => a.Position)                      // tra quelli con posizione, ordina per valore
    .ThenByDescending(a => a.IsFeatured)          // poi featured
    .ThenByDescending(a => a.CreationTime);       // poi data
```

Cosa fa EF Core con questa espressione:
```sql
ORDER BY
    CASE WHEN Position IS NULL THEN 1 ELSE 0 END ASC,  -- nulls last
    Position ASC,
    IsFeatured DESC,
    CreationTime DESC
```

Il `CASE WHEN` è una colonna virtuale calcolata al momento della query — non esiste
nel database, viene generata da EF Core sulla base della lambda C#.

---

## Dynamic LINQ — ordinamento come stringa

Nel progetto, il frontend può mandare un parametro `sorting` come stringa:
```
GET /v1/assortments?sorting=isFeatured%20desc,%20lastModificationTime%20desc
```

Il backend lo processa così:
```csharp
// Se il frontend manda Sorting = "isFeatured desc, lastModificationTime desc"
return query.OrderBy(sortInput.Sorting);  // Dynamic LINQ
```

`OrderBy(string)` non è LINQ standard — viene da **System.Linq.Dynamic.Core**,
una libreria che interpreta le stringhe come espressioni di ordinamento.

**Quando usarlo**: quando l'ordinamento è deciso dall'utente a runtime.
**Quando NON usarlo**: quando l'ordinamento è fisso nel codice — usa le lambda tipate,
sono più sicure (errori a compile-time, non a runtime).

---

## Il pattern usato nel progetto

```csharp
// In AssortmentsAppService.cs
protected override IQueryable<Assortment> ApplySorting(
    IQueryable<Assortment> query,
    AssortmentRequestDto input)
{
    // Se il frontend chiede l'ordinamento per posizione
    if (input.SortByPosition == true)
    {
        return query
            .OrderBy(a => a.Position == null ? 1 : 0)   // positioned first
            .ThenBy(a => a.Position)
            .ThenByDescending(a => a.IsFeatured)
            .ThenByDescending(a => a.CreationTime);
    }

    // Altrimenti: logica standard (usa il parametro sorting del frontend)
    return base.ApplySorting(query, input);
}
```

`ApplySorting` è un metodo virtual della classe base — lo override per aggiungere
comportamento specifico agli assortimenti senza toccare la logica generale.

---

## Riepilogo

| Metodo | SQL generato | Quando usarlo |
|--------|-------------|---------------|
| `.OrderBy(x => x.Campo)` | `ORDER BY Campo ASC` | Primo criterio, crescente |
| `.OrderByDescending(x => x.Campo)` | `ORDER BY Campo DESC` | Primo criterio, decrescente |
| `.ThenBy(x => x.Campo)` | `, Campo ASC` | Secondo criterio (e oltre) |
| `.ThenByDescending(x => x.Campo)` | `, Campo DESC` | Secondo criterio decrescente |
| `.OrderBy(x => x.Campo == null ? 1 : 0)` | `CASE WHEN Campo IS NULL THEN 1 ELSE 0 END` | Metti i null alla fine |
| `.OrderBy(stringaSorting)` | Dipende dalla stringa | Sorting dinamico da frontend |
