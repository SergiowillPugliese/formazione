# 09 — Async e await in C#

## Cosa già sai da Angular

In Angular hai usato `async/await` con le Promise o gli Observable:

```typescript
// TypeScript / Angular
async function getUser(id: number): Promise<User> {
    const user = await this.http.get<User>(`/api/users/${id}`).toPromise();
    return user;
}
```

In C# `async/await` esiste con la stessa logica, ma con alcune differenze importanti.

---

## `Task` — l'equivalente di `Promise`

In JavaScript/TypeScript una operazione asincrona restituisce una `Promise<T>`.
In C# restituisce un `Task<T>`.

```typescript
// TypeScript
async function getUser(): Promise<User> { ... }
async function doSomething(): Promise<void> { ... }
```

```csharp
// C#
async Task<User> GetUserAsync() { ... }
async Task DoSomethingAsync() { ... }
```

| TypeScript | C# |
|---|---|
| `Promise<User>` | `Task<User>` |
| `Promise<void>` | `Task` (senza tipo) |
| `async function` | `async Task` |
| `await` | `await` |

---

## Sintassi base

```csharp
// Metodo asincrono che restituisce un valore
public async Task<byte[]> ExportAsync(ReportExportRequestModel request, CancellationToken ct)
{
    // await "aspetta" il completamento della Task prima di proseguire
    var currentUserInfo = await _catalogDataProxy.GetUserInfoWithStoreAccessOrThrowAsync(
        currentUserId, request.StoreId, ct);

    var usersInfo = await _catalogDataProxy.GetUsersInfoAsync(userIds, designerId, ct);

    return GenerateExcelWorkbook(rows);
}
```

Identico a TypeScript nella logica: il metodo si "mette in pausa" sull'`await`,
lascia il thread libero per altro lavoro, e riprende quando l'operazione è completata.

---

## La convenzione `Async` nel nome

In C# è convenzione aggiungere `Async` al nome dei metodi asincroni:

```csharp
GetUserAsync()          // restituisce Task<User>
ExportAsync()           // restituisce Task<byte[]>
GetAgencyOfUserOrThrowAsync()  // restituisce Task<AgencyInfoModel>
```

In TypeScript questa convenzione non è obbligatoria ma in C# è fortemente
rispettata — se vedi `Async` nel nome, sai che devi usare `await`.

---

## `CancellationToken` — come fermare un'operazione

`CancellationToken` è un meccanismo per annullare operazioni asincrone in corso.
Immagina che un utente chiuda il browser mentre il report Excel sta girando —
senza `CancellationToken`, il server continuerebbe a lavorare inutilmente.

```csharp
public async Task<byte[]> ExportAsync(
    ReportExportRequestModel request,
    long currentUserId,
    CancellationToken ct = default)  // ct = cancellation token
{
    // Ogni operazione async riceve il token
    var userInfo = await _catalogDataProxy.GetUserInfoWithStoreAccessOrThrowAsync(
        currentUserId, request.StoreId, ct); // passa il token

    var usersInfo = await _catalogDataProxy.GetUsersInfoAsync(userIds, designerId, ct);
}
```

La regola pratica: **passa sempre `ct` a tutte le operazioni async**.
Il `= default` significa che se non passi nessun token, usa uno "neutro"
che non si annullerà mai. È solo un valore di default per comodità.

---

## `await` in un `foreach` — operazioni sequenziali

Nel progetto risolviamo le agenzie degli utenti in un foreach:

```csharp
var agencyInfoByUserId = new Dictionary<long, AgencyInfoModel>();

foreach (var agencyUserId in agencyRequestingUserIds)
{
    // Ogni iterazione aspetta il completamento prima di procedere
    var agencyInfo = await _catalogDataProxy.GetAgencyOfUserOrThrowAsync(agencyUserId, ct);
    agencyInfoByUserId[agencyUserId] = agencyInfo;
}
```

Questo è **sequenziale**: prima risolve userId=42, poi userId=99, ecc.
Se vuoi parallelismo, dovresti usare `Task.WhenAll()` — ma per semplicità,
nel contesto di un report, il sequenziale va benissimo.

---

## Differenza da Angular: non ci sono Observable

In Angular usi molto RxJS con `Observable`, `pipe`, `map`, `switchMap`, ecc.
In C# (in questo progetto) non esistono Observable — tutto è `Task` e `await`.

```typescript
// Angular — con Observable
this.agencyService.getAgency(id).pipe(
    map(agency => agency.name)
).subscribe(name => console.log(name));
```

```csharp
// C# — con Task
var agency = await _catalogDataProxy.GetAgencyOfUserOrThrowAsync(id, ct);
var name = agency.Name;
```

Molto più lineare da leggere.

---

## `async` e `static` insieme

Nel progetto, `BuildRow` è `private static`. Un metodo statico può essere async
se necessario:

```csharp
// NON async — fa solo calcoli in memoria, non operazioni IO
private static CustomizationRequestReportRow BuildRow(...)
{
    return new CustomizationRequestReportRow { ... };
}

// ASYNC — farebbe chiamate al database o API
private static async Task<CustomizationRequestReportRow> BuildRowAsync(...)
{
    var data = await SomeAsyncOperation();
    return new CustomizationRequestReportRow { ... };
}
```

`BuildRow` non è async perché non fa nessuna operazione IO (database, rete, file).
Fa solo trasformazioni in memoria — nessun bisogno di `await`.
