# 3.2 — Task.WhenAll e parallelismo

## Il problema

Hai 3 operazioni async indipendenti da fare. Se le fai in sequenza,
aspetti la somma dei loro tempi:

```csharp
// LENTO — sequenziale (300ms + 200ms + 150ms = 650ms)
var customers = await _catalogDataProxy.GetCustomersInfoAsync(customerIds, ct);
var posInfo   = await _catalogDataProxy.GetPointsOfSaleInfoAsync(posIds, ct);
var users     = await _catalogDataProxy.GetUsersLightInfoAsync(userIds, ct);
```

Se le operazioni sono indipendenti (non usa il risultato dell'una
per fare l'altra), puoi farle **in parallelo**:

```csharp
// VELOCE — parallelo (max(300ms, 200ms, 150ms) = 300ms)
var customersTask = _catalogDataProxy.GetCustomersInfoAsync(customerIds, ct);
var posTask       = _catalogDataProxy.GetPointsOfSaleInfoAsync(posIds, ct);
var usersTask     = _catalogDataProxy.GetUsersLightInfoAsync(userIds, ct);

await Task.WhenAll(customersTask, posTask, usersTask);

var customers = await customersTask;
var posInfo   = await posTask;
var users     = await usersTask;
```

---

## Analogia TypeScript

Hai esattamente lo stesso concetto con `Promise.all`:

```typescript
// TypeScript — Promise.all
const [customers, posInfo, users] = await Promise.all([
    getCustomers(customerIds),
    getPointsOfSale(posIds),
    getUsers(userIds)
]);
```

`Task.WhenAll` in C# = `Promise.all` in TypeScript. Stessa idea,
sintassi leggermente diversa.

---

## Come funziona `Task.WhenAll`

Quando chiami un metodo `async` **senza** `await`, ottieni un `Task`
che rappresenta l'operazione in corso — ma non aspetti il risultato.

```csharp
// Avvia tutte e tre le operazioni contemporaneamente
var t1 = GetCustomersAsync();   // parte subito, non aspetta
var t2 = GetPosAsync();         // parte subito, non aspetta
var t3 = GetUsersAsync();       // parte subito, non aspetta

// Ora aspetta che TUTTE siano finite
await Task.WhenAll(t1, t2, t3);

// Qui tutte e tre sono completate
var customers = await t1;  // risultato già pronto, nessuna attesa
```

---

## Raccogliere i risultati in modo compatto

C# permette di destrutturare con tuple o usare direttamente i task:

```csharp
// Metodo 1: await Task.WhenAll poi leggi i singoli task
var customersTask = GetCustomersAsync();
var posTask = GetPosAsync();
await Task.WhenAll(customersTask, posTask);
var customers = await customersTask;
var pos = await posTask;

// Metodo 2 (se i tipi sono uguali): array di task
var tasks = userIds.Select(id => GetUserAsync(id)).ToList();
await Task.WhenAll(tasks);
var users = tasks.Select(t => t.Result).ToList();
// .Result è sicuro SOLO dopo WhenAll — altrimenti blocca il thread
```

---

## `Task.WhenAny` — il primo che finisce

`WhenAny` completa appena **uno qualsiasi** dei task finisce:

```csharp
var fastTask = GetFromCacheAsync();
var slowTask = GetFromDatabaseAsync();

var firstCompleted = await Task.WhenAny(fastTask, slowTask);
var result = await firstCompleted;
// Usa il risultato di chi ha risposto prima (probabilmente la cache)
```

Utile per pattern cache-con-fallback o timeout custom.

---

## Gestione errori con `Task.WhenAll`

Se uno dei task lancia un'eccezione, `WhenAll` la propaga:

```csharp
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    // Cattura la prima eccezione — le altre potrebbero essere perse
    // Per vedere tutte le eccezioni usa task.Exception dopo WhenAll
}
```

**Attenzione**: `WhenAll` aspetta che **tutti** i task finiscano
prima di lanciare l'eccezione, anche se uno fallisce subito.
Questo è diverso da "si ferma al primo errore".

---

## Quando usarlo (e quando no)

**Usa `Task.WhenAll` quando:**
- Le operazioni sono indipendenti (non usi il risultato di A per fare B)
- Stai facendo più chiamate a servizi/database in parallelo
- Vuoi ridurre il tempo di risposta

**Non usarlo quando:**
- L'operazione B dipende dal risultato di A
- Stai iterando su una lista e ogni elemento dipende dal precedente
- Le operazioni modificano lo stesso stato (rischio race condition)

---

## Nel progetto

Il report Excel ha esattamente questo pattern — più lookup indipendenti
che potrebbero essere parallelizzati:

```csharp
// Come è ora (sequenziale)
var storeChannelInfo = await _catalogDataProxy.GetStoreChannelInfoOrThrowAsync(...);
var customersInfo = await _catalogDataProxy.GetCustomersInfoAsync(...);
var posInfo = await _catalogDataProxy.GetPointsOfSaleInfoAsync(...);
var usersInfo = await _catalogDataProxy.GetUsersLightInfoAsync(...);

// Come potrebbe essere (parallelo)
var storeTask    = _catalogDataProxy.GetStoreChannelInfoOrThrowAsync(...);
var customersTask = _catalogDataProxy.GetCustomersInfoAsync(...);
var posTask      = _catalogDataProxy.GetPointsOfSaleInfoAsync(...);
var usersTask    = _catalogDataProxy.GetUsersLightInfoAsync(...);

await Task.WhenAll(storeTask, customersTask, posTask, usersTask);
```

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| `var t = MetodoAsync()` | Avvia senza aspettare — ottieni un Task |
| `await Task.WhenAll(t1, t2)` | Aspetta che TUTTI finiscano |
| `await Task.WhenAny(t1, t2)` | Aspetta che UNO finisca |
| `await t` dopo WhenAll | Legge il risultato già pronto |
| `Promise.all` in TypeScript | Equivalente di `Task.WhenAll` |
