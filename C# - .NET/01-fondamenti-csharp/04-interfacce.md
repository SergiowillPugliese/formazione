# 04 — Interfacce

## Il problema che risolvono

Immagina di avere un metodo che invia notifiche agli utenti.
Oggi le mandi via email, domani via SMS, dopodomani via push notification.
Se il metodo dipende direttamente da `EmailService`, ogni volta che cambi
il canale devi riscrivere il metodo.

La soluzione: il metodo non dipende da `EmailService` ma da
`INotificationService` — un contratto che dice "ho bisogno di qualcosa
che sa inviare notifiche, non mi interessa come lo fa".

---

## Cos'è un'interfaccia in C#

Un'interfaccia è un **contratto**: definisce quali metodi e property
un tipo deve avere, senza dire come li implementa.

```csharp
// Interfaccia — solo la firma, nessuna implementazione
public interface INotificationService
{
    Task SendAsync(string recipient, string message, CancellationToken ct);
}

// Implementazione 1 — via email
public class EmailNotificationService : INotificationService
{
    public async Task SendAsync(string recipient, string message, CancellationToken ct)
    {
        // logica per inviare email
    }
}

// Implementazione 2 — via SMS
public class SmsNotificationService : INotificationService
{
    public async Task SendAsync(string recipient, string message, CancellationToken ct)
    {
        // logica per inviare SMS
    }
}
```

Chi usa `INotificationService` non sa e non gli interessa
se sta usando email o SMS — entrambi rispettano il contratto.

---

## L'analogo in Angular

In Angular hai usato i servizi con `InjectionToken` o abstract class per fare
la stessa cosa, ma non è obbligatorio come in C#.

La cosa più vicina che probabilmente hai visto è questo pattern:

```typescript
// TypeScript — abstract class come interfaccia
abstract class NotificationService {
    abstract send(recipient: string, message: string): Observable<void>;
}

// Implementazione concreta
class EmailNotificationService extends NotificationService {
    send(recipient: string, message: string): Observable<void> {
        // ...
    }
}
```

In C# si usa `interface` direttamente (senza `abstract class`) per i contratti puri.
È più semplice e più esplicito.

---

## Interfacce nel progetto

### `ICatalogDataProxy`

```csharp
// Definita in Kustom.Domain
public interface ICatalogDataProxy
{
    Task<AgencyInfoModel> GetAgencyOfUserOrThrowAsync(long userId, CancellationToken ct);
    Task<List<UserInfoModel>> GetUsersInfoAsync(HashSet<long> userIds, long storeDefaultDesignerId, CancellationToken ct);
    Task<List<CustomerInfoModel>> GetCustomersInfoAsync(List<long> customerIds, CancellationToken ct);
    // ... altri metodi
}
```

`CustomizationRequestReportExcelExporter` dipende da `ICatalogDataProxy`,
non da `CatalogDataProxy`. Questo significa:

1. L'exporter non sa che i dati vengono da Catalog — non gli interessa
2. Nei test, puoi iniettare un `FakeCatalogDataProxy` che restituisce dati statici
3. Se domani cambi come Catalog restituisce i dati, l'exporter non cambia

### `IAgenciesAppService`

```csharp
// Definita in Catalog.Application.Contracts
public interface IAgenciesAppService
{
    Task<AgencyInfoDto> GetAgencyOfUserOrThrowAsync(long userId, CancellationToken ct);
    Task<List<long>> GetManagedPointOfSaleIdsAsync(long agencyId, int storeId, CancellationToken ct);
}
```

`CatalogDataProxy` dipende da `IAgenciesAppService`,
non da `AgenciesAppService` direttamente.

---

## Convenzione di nomenclatura

In C# le interfacce hanno sempre il prefisso `I` (maiuscola):

```
INotificationService   ← interfaccia
NotificationService    ← implementazione concreta

ICatalogDataProxy      ← interfaccia
CatalogDataProxy       ← implementazione concreta

IAgenciesAppService    ← interfaccia
AgenciesAppService     ← implementazione concreta
```

Quando vedi una `I` all'inizio del nome, stai leggendo un'interfaccia.

---

## Come si "usa" un'interfaccia — Dependency Injection

In C# non scrivi mai `new CatalogDataProxy(...)` nel costruttore di un'altra classe.
Il sistema di Dependency Injection (DI) — che vedremo nel modulo 10 —
si occupa di fornire l'implementazione giusta quando la classe viene creata.

```csharp
// L'exporter dice "ho bisogno di qualcosa che implementa ICatalogDataProxy"
public class CustomizationRequestReportExcelExporter
{
    private readonly ICatalogDataProxy _catalogDataProxy;  // interfaccia, non classe concreta

    public CustomizationRequestReportExcelExporter(ICatalogDataProxy catalogDataProxy)
    {
        _catalogDataProxy = catalogDataProxy;
        // A runtime, .NET inietta automaticamente CatalogDataProxy
    }
}
```

---

## Una classe può implementare più interfacce

In C# non c'è limite al numero di interfacce che una classe può implementare:

```csharp
public class AgenciesAppService : IAgenciesAppService, IPointOfSaleService
{
    // deve implementare tutti i metodi di entrambe le interfacce
}
```

In TypeScript è lo stesso con `implements`:
```typescript
class AgenciesService implements IAgenciesService, IPointOfSaleService {
    // ...
}
```

---

## Perché non usare direttamente la classe concreta?

Tre motivi pratici:

**1. Testabilità**: nei test puoi sostituire con un'implementazione fake
**2. Disaccoppiamento**: cambi l'implementazione senza toccare chi la usa
**3. Separazione moduli**: nel progetto, Kustom non può dipendere da `CatalogDataProxy`
   (è in un altro modulo), ma può dipendere da `ICatalogDataProxy` (definita in Kustom stesso)
