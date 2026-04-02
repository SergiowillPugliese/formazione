# 12 — Architettura DDD e layer del progetto

## Cos'è DDD (Domain-Driven Design)

DDD è un approccio all'architettura software che dice:
"il codice deve rispecchiare il **dominio del business**, non la struttura tecnica."

Il "dominio" è il problema che l'applicazione risolve.
Per 4Shop, il dominio è: catalogo prodotti, ordini, personalizzazioni (kustom), agenzie.

La conseguenza pratica: invece di organizzare il codice per "tecnologia"
(tutti i controller insieme, tutti i repository insieme), lo organizzi per "concetto di business".

---

## I layer del progetto

Il progetto ha 4 layer, dal più "puro" al più "sporco" (tecnico):

```
┌─────────────────────────────────────────────────────┐
│  HOST / API  (Catalog.Host)                         │
│  Controller, middleware, routing HTTP                │
├─────────────────────────────────────────────────────┤
│  APPLICATION  (Catalog.Application)                 │
│  Orchestrazione, DTO, background jobs               │
├─────────────────────────────────────────────────────┤
│  DOMAIN  (Catalog.Domain)                           │
│  Entità, regole business, interfacce repository     │
├─────────────────────────────────────────────────────┤
│  INFRASTRUCTURE  (implicito in EF Core, repository) │
│  Database, file system, servizi esterni             │
└─────────────────────────────────────────────────────┘
```

**Regola fondamentale**: le dipendenze vanno solo verso il basso.
Domain non conosce Application. Application non conosce Host.

---

## Domain — il cuore puro

**Cosa contiene:**
- Entità (le classi che rappresentano i concetti del business)
- Domain Services (logica business che non appartiene a una singola entità)
- Repository interfaces (definisce "cosa" serve dal DB, ma non "come")

**Non contiene:**
- Nessuna chiamata al database (solo interfacce)
- Nessuna chiamata HTTP
- Nessun DTO (Data Transfer Object — il formato esposto all'esterno)

```csharp
// In Domain — CustomizationRequest è un'entità del dominio
public class CustomizationRequest : FullAuditedEntity4Shop<long>
{
    public string Title { get; private set; }
    public CustomizationRequestStatus Status { get; private set; }
    public long? AgencyRequestingUserId { get; private set; }
    // ...

    // La logica business vive qui
    public void Approve(long approverId)
    {
        if (Status != CustomizationRequestStatus.PendingApproval)
            throw new BusinessException("Non puoi approvare in questo stato");
        Status = CustomizationRequestStatus.Approved;
    }
}
```

```csharp
// In Domain — ICustomizationRequestRepository è solo un'interfaccia
public interface ICustomizationRequestRepository
    : IFourShopEfCoreRepository<CustomizationRequest, long>
{
    Task<List<CustomizationRequest>> GetForReportAsync(
        ReportExportRequestModel request,
        CancellationToken ct);
}
// Dice "ho bisogno di questo metodo" ma NON sa come è implementato
```

---

## Application — l'orchestratore

**Cosa contiene:**
- AppService (punti di ingresso, coordinano il lavoro)
- DTO (Data Transfer Objects — il formato dei dati per l'API)
- Background jobs
- Mapping profiles (AutoMapper)

**Regola**: nessuna logica di business qui.
Application coordina, Domain decide.

```csharp
// In Application — CustomizationRequestsAppService
public class CustomizationRequestsAppService : CatalogBaseAppService, ...
{
    private readonly ICustomizationRequestReportExcelExporter _excelExporter;

    public async Task<byte[]> ExportReportAsync(ReportExportRequestModel request)
    {
        // L'AppService NON sa come funziona l'export
        // Delega tutto all'exporter (che è nel Domain)
        return await _excelExporter.ExportAsync(request, CurrentUser.Id!.Value, ...);
    }
}
```

---

## Host — il punto di ingresso HTTP

**Cosa contiene:**
- Controller (ricevono le richieste HTTP, restituiscono risposte)
- Middleware (autenticazione, logging, gestione errori)
- Configurazione DI, routing

```csharp
// In Host — CustomizationRequestsController
[ApiController]
[Route("v1/customization-requests")]
public class CustomizationRequestsController : CatalogBaseController
{
    private readonly ICustomizationRequestsAppService _appService;

    [HttpGet("export-report")]
    public async Task<IActionResult> ExportReport([FromQuery] ReportExportRequestModel request)
    {
        // Il controller non sa nulla di business — delega all'AppService
        var bytes = await _appService.ExportReportAsync(request);
        return File(bytes, "application/vnd.openxmlformats-officedocument...", "report.xlsx");
    }
}
```

---

## Perché questa separazione?

### Senza DDD (spaghetti tipici)
```csharp
// Controller con tutto dentro — tipico anti-pattern
[HttpGet("export")]
public async Task<IActionResult> Export()
{
    // Query al database direttamente nel controller
    var crs = await _dbContext.CustomizationRequests.ToListAsync();

    // Logica business nel controller
    if (crs.Any(cr => cr.Status == "Pending"))
        throw new Exception("non puoi esportare");

    // Formattazione direttamente qui
    var excel = new ExcelPackage();
    // ... 300 righe di codice Excel nel controller
}
```

### Con DDD
- Il controller fa solo routing + risposta HTTP
- L'AppService coordina
- Il Domain contiene la logica business
- Ogni layer è testabile indipendentemente

---

## Il modulo Kustom

Nel progetto, Kustom è un **modulo ABP separato** — come un'applicazione
mini-autonoma dentro il progetto principale.

```
modules/Kustom/
    src/
        Kustom.Domain/          ← le entità e la logica business di Kustom
        Kustom.Application/     ← gli AppService di Kustom
```

Kustom.Domain **non può importare** classi da `Catalog.Application`.
Può solo importare da `Catalog.Application.Contracts` (il layer contratti/interfacce).

Questa separazione è importante: il Domain deve restare puro e indipendente.

---

## Il flusso di una richiesta HTTP

```
Browser / Client
    ↓  HTTP GET /v1/customization-requests/export-report
Host: CustomizationRequestsController.ExportReport()
    ↓  delega
Application: CustomizationRequestsAppService.ExportReportAsync()
    ↓  delega
Domain: CustomizationRequestReportExcelExporter.ExportAsync()
    ↓  ha bisogno di dati dal DB
Domain: ICustomizationRequestRepository.GetForReportAsync()  [interfaccia]
    ↓  implementata da
Infrastructure: CustomizationRequestRepository (EF Core)  [implementazione]
    ↓  query SQL al database
Database
```

Il Domain parla solo con interfacce. Non sa che sotto c'è EF Core e SQL Server.
Domani potresti sostituire EF Core con MongoDB — il Domain non cambia.

---

## Riepilogo: chi fa cosa

| Layer | Responsabilità | Esempio |
|---|---|---|
| **Host** | Routing HTTP, risposta | `CustomizationRequestsController` |
| **Application** | Coordinazione, DTO | `CustomizationRequestsAppService` |
| **Domain** | Regole business, logica | `CustomizationRequest`, `CustomizationRequestReportExcelExporter` |
| **Infrastructure** | DB, servizi esterni | `CustomizationRequestRepository` (EF Core) |

| Regola | Motivo |
|---|---|
| Domain non conosce Application | Domain è il nucleo — non dipende da nessuno |
| Application non conosce Host | Testabile senza HTTP |
| Dipendenze verso il basso | Sostituibilità degli strati |
