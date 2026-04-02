# 13 — Il pattern Proxy (cross-module)

## Il problema

Kustom.Domain ha bisogno di dati sulle agenzie — per il report Excel.
Ma i dati delle agenzie vivono in `AgenciesAppService`, che è in `Catalog.Application`.

Kustom.Domain non può importare direttamente `AgenciesAppService`.
Perché? Perché romperebbe la regola fondamentale dell'architettura DDD:
**il Domain non può dipendere dall'Application**.

```
Kustom.Domain
    ↓ NON PUÒ importare direttamente
Catalog.Application  ← qui vive AgenciesAppService
```

Come fa Kustom.Domain a ottenere i dati di cui ha bisogno?

---

## La soluzione: il Proxy

Definiamo un'interfaccia nel Domain, e spostiamo l'implementazione nell'Application.

```
Kustom.Domain                    Kustom.Application
┌─────────────────┐              ┌─────────────────────────┐
│ ICatalogDataProxy│◄────────────│ CatalogDataProxy        │
│ (interfaccia)   │  implementa  │ (classe concreta)       │
│                 │              │ inietta AgenciesAppService│
└─────────────────┘              └─────────────────────────┘
```

- `ICatalogDataProxy` vive in `Kustom.Domain` — il Domain conosce solo questa interfaccia
- `CatalogDataProxy` vive in `Kustom.Application` — qui può usare `AgenciesAppService`

---

## L'interfaccia nel Domain

```csharp
// In Kustom.Domain/DomainServices/CatalogData/ICatalogDataProxy.cs
public interface ICatalogDataProxy
{
    Task<UserInfoWithStoreAccessModel> GetUserInfoWithStoreAccessOrThrowAsync(
        long userId, long storeId, CancellationToken ct);

    Task<List<UserInfoModel>> GetUsersInfoAsync(
        IEnumerable<long> userIds, long? designerId, CancellationToken ct);

    Task<AgencyInfoModel> GetAgencyOfUserOrThrowAsync(long userId, CancellationToken ct);

    // ... altri metodi
}
```

Il Domain dichiara: "ho bisogno di questi dati". Non sa chi li fornirà.
È come un `interface` in TypeScript — definisce il contratto, non l'implementazione.

---

## L'implementazione nell'Application

```csharp
// In Kustom.Application/DomainServices/CatalogData/CatalogDataProxy.cs
public class CatalogDataProxy : ICatalogDataProxy, ITransientDependency
{
    private readonly IAgenciesAppService _agenciesAppService;
    private readonly IUsersAppService _usersAppService;
    private readonly IObjectMapper _objectMapper;

    public CatalogDataProxy(
        IAgenciesAppService agenciesAppService,
        IUsersAppService usersAppService,
        IObjectMapper objectMapper)
    {
        _agenciesAppService = agenciesAppService;
        _usersAppService = usersAppService;
        _objectMapper = objectMapper;
    }

    public async Task<AgencyInfoModel> GetAgencyOfUserOrThrowAsync(long userId, CancellationToken ct)
    {
        // Qui possiamo chiamare AgenciesAppService perché siamo in Application
        var dto = await _agenciesAppService.GetAgencyOfUserOrThrowAsync(userId);

        // Converte AgencyDto (Application) → AgencyInfoModel (Domain)
        return _objectMapper.Map<AgencyDto, AgencyInfoModel>(dto);
    }
}
```

`CatalogDataProxy` implementa `ICatalogDataProxy` e inietta i veri AppService.
Il Domain non vede nulla di questo — vede solo `ICatalogDataProxy`.

---

## Il flusso completo

```
CustomizationRequestReportExcelExporter   (Kustom.Domain)
    ↓  usa
ICatalogDataProxy                         (interfaccia — Kustom.Domain)
    ↓  implementata da
CatalogDataProxy                          (Kustom.Application)
    ↓  inietta
IAgenciesAppService                       (interfaccia — Catalog.Application.Contracts)
    ↓  implementata da
AgenciesAppService                        (Catalog.Application)
    ↓  inietta
IAgencyRepository, AgencyDomainService    (Catalog.Domain)
    ↓  database
SQL Server
```

Ogni livello conosce solo il livello immediatamente sotto, tramite interfacce.
Il container DI costruisce tutta questa catena automaticamente a runtime.

---

## Perché si chiama "Proxy"?

Il pattern Proxy (nel senso informatico generale) è un oggetto che
"sta al posto di" un altro oggetto — lo "rappresenta" agli occhi di chi lo usa.

`CatalogDataProxy` sta al posto di tutti i vari AppService del modulo Catalog
agli occhi di Kustom.Domain. Il Domain non sa quanti AppService ci sono dietro,
non sa come sono costruiti — vede solo una singola interfaccia pulita.

In più, il Proxy può fare trasformazioni sui dati: converte i DTO dell'Application
nei Model del Domain. Questo lo rende anche un **Anti-Corruption Layer** (ACL) —
un termine DDD che significa "strato che protegge il Domain dal mondo esterno".

---

## Confronto con Angular

In Angular, quando un componente ha bisogno di dati da un'API:

```typescript
// Il componente non chiama l'API direttamente
@Component({ ... })
export class CustomizationListComponent {
    constructor(private catalogProxy: CatalogDataService) {}
    // Usa CatalogDataService, non HttpClient direttamente
}

// CatalogDataService fa la chiamata HTTP e trasforma i dati
@Injectable({ providedIn: 'root' })
export class CatalogDataService {
    constructor(private http: HttpClient) {}

    getAgency(userId: number): Observable<AgencyModel> {
        return this.http
            .get<AgencyDto>(`/api/agencies/user/${userId}`)
            .pipe(map(dto => this.toModel(dto))); // trasformazione DTO → Model
    }
}
```

`CatalogDataService` in Angular gioca lo stesso ruolo di `CatalogDataProxy` in C#:
isola il componente dai dettagli dell'implementazione e trasforma i dati.

---

## Dove vivono i file nel progetto

```
modules/Kustom/
    src/
        Kustom.Domain/
            DomainServices/
                CatalogData/
                    ICatalogDataProxy.cs        ← interfaccia (Domain)
                    Models/
                        AgencyInfoModel.cs      ← modello Domain
                        UserInfoModel.cs
                        ...

        Kustom.Application/
            DomainServices/
                CatalogData/
                    CatalogDataProxy.cs                      ← implementazione (Application)
                    CatalogDataProxyAutoMapperProfile.cs     ← mapping DTO → Model
```

---

## Riepilogo

| Concetto | Ruolo |
|---|---|
| `ICatalogDataProxy` | Contratto — il Domain dichiara di cosa ha bisogno |
| `CatalogDataProxy` | Implementazione — l'Application fornisce i dati reali |
| `AutoMapper` | Converte i DTO dell'Application nei Model del Domain |
| `ITransientDependency` | ABP registra automaticamente `CatalogDataProxy` nel DI |

**Il principio chiave**: il Domain definisce i contratti, l'Application li soddisfa.
Il Domain non sa nulla dell'implementazione — solo dell'interfaccia.
Questo mantiene il Domain testabile, indipendente, e sostituibile.
