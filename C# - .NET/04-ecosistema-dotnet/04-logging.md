# 4.4 — Logging

## Cos'è e perché serve

Il **logging** è il modo in cui un'applicazione backend "parla" con
chi la gestisce. In un'app Angular mostri errori in console o in
un toast UI. In un backend non c'è UI — scrivi su un log.

I log servono per:
- Capire cosa è successo quando qualcosa va storto in produzione
- Tracciare il flusso di una richiesta
- Monitorare le performance
- Fare debug senza attaccare un debugger

---

## `ILogger<T>` — l'interfaccia standard

In .NET usi sempre `ILogger<T>` — mai `Console.WriteLine` in produzione.
La `T` è la classe che sta loggando (serve per filtrare i log per sorgente).

```csharp
public class CustomizationRequestReportExcelExporter
{
    private readonly ILogger<CustomizationRequestReportExcelExporter> _logger;

    public CustomizationRequestReportExcelExporter(
        ILogger<CustomizationRequestReportExcelExporter> logger)
    {
        _logger = logger;  // iniettato via DI come tutto il resto
    }
}
```

**Analogia Angular**: è come il servizio `console` ma con livelli,
filtri, e destinazioni configurabili (file, Azure, Elastic...).

---

## I livelli di log

I log hanno livelli di severità — dal più verbose al più critico:

```
Trace       → dettagli minutissimi (loop interni, ogni singolo step)
Debug       → informazioni di debug (valori di variabili, branch presi)
Information → eventi normali importanti (richiesta ricevuta, job completato)
Warning     → qualcosa di insolito ma non bloccante
Error       → errore gestito (eccezione catturata, operazione fallita)
Critical    → errore fatale (il sistema non può continuare)
```

In produzione si configurano tipicamente solo `Warning` e superiori —
`Debug` e `Trace` sono troppo verbosi.

---

## Sintassi

```csharp
_logger.LogTrace("Entro nel loop per userId={UserId}", userId);
_logger.LogDebug("Agency trovata: {AgencyId}", agency.Id);
_logger.LogInformation("Report generato per store {StoreId}, {Count} righe", storeId, rows.Count);
_logger.LogWarning("Agency non trovata per userId={UserId}, colonne vuote", userId);
_logger.LogError(ex, "Errore durante export report per storeId={StoreId}", storeId);
_logger.LogCritical("Database irraggiungibile");
```

---

## Structured logging — non usare interpolazione

Nota la sintassi `{NomeParametro}` invece di `$"..."`:

```csharp
// SBAGLIATO — string interpolation
_logger.LogInformation($"Report per store {storeId} con {rows.Count} righe");

// CORRETTO — structured logging
_logger.LogInformation("Report per store {StoreId} con {Count} righe", storeId, rows.Count);
```

**Perché?** Con `{NomeParametro}`, i sistemi di log (Azure, Elastic, Seq)
possono indicizzare i parametri come campi separati — puoi cercare
`StoreId = 5` invece di fare grep su testo libero.

---

## Logging delle eccezioni

Quando catturi un'eccezione, passala come **primo parametro** a `LogError`:

```csharp
try
{
    var agency = await GetAgencyAsync(userId);
}
catch (Exception ex)
{
    // ex come primo parametro — include lo stack trace nel log
    _logger.LogError(ex, "Errore nel recupero agency per userId={UserId}", userId);
    throw;  // rilancia l'eccezione dopo averla loggata
}
```

---

## Nel progetto: `FourShopDomainServiceBase`

Le classi che estendono `FourShopDomainServiceBase` o `CatalogBaseAppService`
hanno già `Logger` disponibile come property — non serve iniettare `ILogger`:

```csharp
public class CustomizationRequestReportExcelExporter : FourShopDomainServiceBase
{
    public async Task<byte[]> ExportAsync(...)
    {
        Logger.LogInformation("Inizio export report per storeId={StoreId}", request.StoreId);
        // Logger è già disponibile dalla classe base
    }
}
```

Questo è un pattern ABP — la classe base gestisce l'iniezione di `ILogger`.

---

## Riepilogo

| Livello | Quando usarlo |
|---|---|
| `LogTrace` | Loop interni, ogni micro-step |
| `LogDebug` | Valori di variabili, branch decisionali |
| `LogInformation` | Eventi normali importanti |
| `LogWarning` | Anomalie non bloccanti |
| `LogError(ex, ...)` | Eccezioni catturate, operazioni fallite |
| `LogCritical` | Errori fatali |

**Regola d'oro**: usa `{NomeParametro}` invece di `$"..."` per
lo structured logging — rende i log cercabili e analizzabili.
