# 5.3 — Repository Pattern

## Cos'è

Il **Repository Pattern** è un pattern architetturale che nasconde
i dettagli dell'accesso ai dati dietro un'interfaccia.

Il Domain dice "ho bisogno di questi dati" — ma non sa **come** sono
recuperati (database, API, cache, file...). È il Repository che sa come.

---

## Il problema senza Repository

```csharp
// SENZA Repository — il Domain dipende direttamente da EF Core
public class CustomizationRequestManager
{
    private readonly CatalogDbContext _dbContext; // ← dipendenza diretta al DB

    public async Task ApproveAsync(long id)
    {
        // Logica business mescolata con logica di accesso ai dati
        var cr = await _dbContext.CustomizationRequests
            .Include(cr => cr.CustomizedProducts)
            .FirstAsync(cr => cr.Id == id);

        cr.Approve();
        await _dbContext.SaveChangesAsync();
    }
}
```

Problemi:
- Il Domain conosce EF Core (non dovrebbe)
- Non puoi testare `ApproveAsync` senza un database reale
- Se cambi ORM, devi modificare il Domain

---

## La soluzione: interfaccia nel Domain, implementazione nell'Infrastructure

```
Domain
┌──────────────────────────────────────────┐
│ ICustomizationRequestRepository          │
│ + GetAsync(id)                           │
│ + GetForReportAsync(request)             │
│ (solo interfaccia — nessuna implementaz.)│
└──────────────────────────────────────────┘
                    ↑ implementa
Infrastructure (EF Core)
┌──────────────────────────────────────────┐
│ CustomizationRequestRepository           │
│ + GetAsync(id) → query EF Core           │
│ + GetForReportAsync(request) → query     │
└──────────────────────────────────────────┘
```

---

## L'interfaccia nel Domain

```csharp
// In Kustom.Domain — solo l'interfaccia
public interface ICustomizationRequestRepository
    : IFourShopEfCoreRepository<CustomizationRequest, long>
{
    // Metodi generici (GetAsync, GetListAsync, DeleteAsync...) ereditati
    // Metodi specifici del dominio:
    Task<List<CustomizationRequest>> GetForReportAsync(
        ReportExportRequestModel request,
        CancellationToken ct);
}
```

Il Domain dichiara **cosa** gli serve. Non sa come è implementato.
`IFourShopEfCoreRepository<TEntity, TKey>` è la base generica —
fornisce le operazioni CRUD standard per qualsiasi entità.

---

## L'implementazione nell'Infrastructure

```csharp
// In Kustom.Application (o Infrastructure) — l'implementazione concreta
public class CustomizationRequestRepository
    : FourShopEfCoreRepository<CatalogDbContext, CustomizationRequest, long>,
      ICustomizationRequestRepository
{
    public async Task<List<CustomizationRequest>> GetForReportAsync(
        ReportExportRequestModel request,
        CancellationToken ct)
    {
        return await (await GetQueryableAsync())
            .AsNoTracking()
            .Where(cr => cr.StoreId == request.StoreId)
            .Include(cr => cr.CustomizedProducts)
                .ThenInclude(cp => cp.Sizes)
            .OrderByDescending(cr => cr.CreationTime)
            .ToListAsync(ct);
    }
}
```

Solo qui c'è EF Core. Il Domain non sa nulla di questo.

---

## Come viene usato nel Domain

```csharp
public class CustomizationRequestReportExcelExporter : FourShopDomainServiceBase
{
    private readonly ICustomizationRequestRepository _repo; // ← interfaccia

    public CustomizationRequestReportExcelExporter(
        ICustomizationRequestRepository repo)  // DI inietta l'implementazione concreta
    {
        _repo = repo;
    }

    public async Task<byte[]> ExportAsync(ReportExportRequestModel request, ...)
    {
        // Il Domain parla con l'interfaccia — non sa che sotto c'è EF Core
        var crs = await _repo.GetForReportAsync(request, ct);
        // ...
    }
}
```

---

## Analogia Angular

In Angular, un componente non chiama `HttpClient` direttamente —
chiama un service:

```typescript
// Componente — usa l'interfaccia (il service)
@Component({ ... })
export class OrderListComponent {
    constructor(private orderService: OrderService) {}

    loadOrders() {
        this.orderService.getOrders().subscribe(orders => ...);
        // Non sa se OrderService usa HttpClient, localStorage, o mock
    }
}

// Service — l'implementazione concreta
@Injectable({ providedIn: 'root' })
export class OrderService {
    constructor(private http: HttpClient) {}

    getOrders(): Observable<Order[]> {
        return this.http.get<Order[]>('/api/orders');
    }
}
```

`OrderService` in Angular = `IOrderRepository` + `OrderRepository` in C#.
La differenza: in C# l'interfaccia è separata dall'implementazione,
in Angular di solito no (ma il concetto è identico).

---

## Perché è importante

1. **Testabilità**: nei test puoi sostituire il repository con un fake
   che restituisce dati statici, senza bisogno di un database
2. **Separazione dei layer**: il Domain rimane puro, senza dipendenze tecniche
3. **Sostituibilità**: potresti cambiare EF Core con Dapper o MongoDB
   modificando solo il repository, non il Domain

---

## Riepilogo

| Concetto | Dove vive | Cosa fa |
|---|---|---|
| `ICustomizationRequestRepository` | Domain | Dichiara cosa serve |
| `CustomizationRequestRepository` | Infrastructure | Implementa con EF Core |
| `IFourShopEfCoreRepository<T, K>` | Shared | CRUD generici per qualsiasi entità |
| DI container | Host | Collega interfaccia → implementazione |
