# 10 — Dependency Injection

## Cosa già sai da Angular

In Angular la DI è ovunque. Ogni servizio ha `@Injectable()` e viene
iniettato nel costruttore:

```typescript
// Angular
@Injectable({ providedIn: 'root' })
export class ReportExporter {
    constructor(
        private catalogDataProxy: CatalogDataProxy,
        private logger: Logger
    ) {}
}
```

Angular si occupa di creare le istanze e di fornirle a chi le richiede.
In C# / .NET funziona esattamente lo stesso — senza i decorator Angular,
ma con lo stesso principio.

---

## Come funziona in C#

```csharp
// .NET — nessun decorator, ma stessa logica
public class CustomizationRequestReportExcelExporter
{
    private readonly ICatalogDataProxy _catalogDataProxy;
    private readonly ICustomizationRequestRepository _customizationRequestRepository;
    private readonly ILogger<CustomizationRequestReportExcelExporter> _logger;

    // Il costruttore dichiara le dipendenze
    public CustomizationRequestReportExcelExporter(
        ICustomizationRequestRepository customizationRequestRepository,
        ICatalogDataProxy catalogDataProxy,
        ILogger<CustomizationRequestReportExcelExporter> logger)
    {
        _customizationRequestRepository = customizationRequestRepository;
        _catalogDataProxy = catalogDataProxy;
        _logger = logger;
    }
}
```

Il framework .NET (o ABP) vede il costruttore, capisce cosa serve,
e fornisce automaticamente le istanze giuste a runtime.

---

## Il container DI

Il cuore della DI è il **container**: un registro che sa "se qualcuno chiede
`ICatalogDataProxy`, dagli un'istanza di `CatalogDataProxy`".

In Angular registri i servizi con `providers` o `providedIn: 'root'`.
In .NET li registri normalmente nel file di startup del modulo:

```csharp
// In un modulo ABP — registrazione del servizio
services.AddTransient<ICatalogDataProxy, CatalogDataProxy>();
// "ogni volta che qualcuno chiede ICatalogDataProxy, crea una nuova CatalogDataProxy"
```

In questo progetto ABP gestisce molte registrazioni automaticamente —
basta implementare l'interfaccia giusta o aggiungere un'attributo come
`ITransientDependency`.

---

## `ITransientDependency` — registrazione automatica in ABP

Nel progetto vedi spesso:

```csharp
public class CustomizationRequestReportExcelExporter
    : ICustomizationRequestReportExcelExporter, ITransientDependency
```

`ITransientDependency` è un'interfaccia marker di ABP — dice al framework:
"registra automaticamente questa classe nel container DI come transient".

Non devi fare niente altro — ABP la trova, la registra, e la inietta.

---

## I lifecycle delle dipendenze

Quando registri un servizio, devi scegliere per quanto "vive" l'istanza:

| Lifecycle | Significato | Analogo Angular |
|---|---|---|
| **Transient** | Nuova istanza ogni volta che viene richiesta | nessun diretto analogo |
| **Scoped** | Una istanza per richiesta HTTP | simile a un service senza `providedIn: 'root'` |
| **Singleton** | Una sola istanza per tutta la vita dell'applicazione | `providedIn: 'root'` |

Nel progetto:
- `ITransientDependency` → Transient (nuova istanza ogni volta)
- Molti AppService → Scoped (vivono per la durata della richiesta HTTP)

---

## Perché dipendere dall'interfaccia e non dalla classe?

```csharp
// MAL FATTO — dipende dalla classe concreta
public class ReportExporter
{
    private readonly CatalogDataProxy _proxy; // classe concreta

    public ReportExporter(CatalogDataProxy proxy)
    {
        _proxy = proxy;
    }
}

// BEN FATTO — dipende dall'interfaccia
public class ReportExporter
{
    private readonly ICatalogDataProxy _proxy; // interfaccia

    public ReportExporter(ICatalogDataProxy proxy)
    {
        _proxy = proxy;
    }
}
```

Con l'interfaccia:
1. **Testabilità**: nei test puoi iniettare una versione fake di `ICatalogDataProxy`
   che ritorna dati statici senza toccare il database
2. **Separazione moduli**: `ReportExporter` vive in `Kustom.Domain`
   e non può dipendere da `CatalogDataProxy` che è in `Kustom.Application`.
   Ma può dipendere da `ICatalogDataProxy` che è definita in `Kustom.Domain` stesso.

---

## Il flusso nel progetto

```
HTTP Request
    ↓
CustomizationRequestsController
    (iniettato: CustomizationRequestsAppService)
    ↓
CustomizationRequestsAppService
    (iniettato: ICustomizationRequestReportExcelExporter)
    ↓
CustomizationRequestReportExcelExporter
    (iniettato: ICatalogDataProxy, ICustomizationRequestRepository)
    ↓
CatalogDataProxy  (implementazione concreta di ICatalogDataProxy)
    (iniettato: IAgenciesAppService, IUsersAppService, ...)
    ↓
AgenciesAppService  (implementazione concreta di IAgenciesAppService)
    (iniettato: IAgencyRepository, AgencyDomainService, ...)
```

Ogni classe dichiara solo le proprie dipendenze immediate.
Il container DI costruisce tutta la catena automaticamente.
