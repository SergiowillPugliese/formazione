# 1.9 — Exception Handling

## Cos'è un'eccezione

Un'**eccezione** (exception) è un errore che interrompe il flusso normale
del programma. In C# ogni eccezione è un oggetto — una classe che
estende `Exception`.

Quando qualcosa va storto, il codice "lancia" (`throw`) un'eccezione.
Se nessuno la "cattura" (`catch`), l'applicazione crasha.

---

## Sintassi base: `try / catch / finally`

```csharp
try
{
    // Codice che potrebbe fallire
    var agency = await _agenciesAppService.GetAgencyOfUserOrThrowAsync(userId);
}
catch (UserFriendlyException ex)
{
    // Gestisci questo tipo specifico di eccezione
    Console.WriteLine($"Errore utente: {ex.Message}");
}
catch (Exception ex)
{
    // Cattura qualsiasi altra eccezione
    Console.WriteLine($"Errore generico: {ex.Message}");
}
finally
{
    // Eseguito SEMPRE — sia in caso di successo che di errore
    // Usato per liberare risorse (connessioni, file, ecc.)
    connection.Close();
}
```

**Analogia JavaScript/TypeScript:**
```typescript
try {
    const agency = await getAgency(userId);
} catch (error) {
    if (error instanceof UserFriendlyError) {
        console.log(error.message);
    }
} finally {
    connection.close();
}
```

Praticamente identico. La differenza principale: in C# puoi specificare
il **tipo** dell'eccezione nel `catch`, e il compilatore ti aiuta.

---

## Lanciare un'eccezione: `throw`

```csharp
public Agency GetAgencyOrThrow(long userId)
{
    var agency = FindAgency(userId);

    if (agency == null)
    {
        throw new UserFriendlyException($"no agency found for user {userId}");
        // Il codice dopo questo throw non viene mai eseguito
    }

    return agency;
}
```

`throw` interrompe immediatamente l'esecuzione del metodo e
"risale" lo stack fino al primo `catch` compatibile.

---

## `UserFriendlyException` — la convenzione ABP

Nel progetto usi ABP Framework, che ha una classe speciale:

```csharp
throw new UserFriendlyException("Messaggio che può vedere l'utente finale");
```

ABP intercetta questa eccezione e la trasforma in una risposta HTTP `400`
con il messaggio leggibile. Le altre eccezioni diventano `500` con
un messaggio generico (per non esporre dettagli interni).

Questo è il pattern nel progetto:
- **Errore di business** (es. "utente non trovato") → `UserFriendlyException`
- **Bug / errore inatteso** → lascia propagare, ABP gestisce il 500

---

## Gerarchia delle eccezioni

Le eccezioni sono classi con ereditarietà. Questo conta quando fai `catch`:

```
Exception                          ← base di tutto
├── SystemException
│   ├── NullReferenceException     ← hai chiamato .Value su un null
│   ├── InvalidOperationException  ← operazione non valida sullo stato
│   └── InvalidCastException       ← cast sbagliato
├── ApplicationException
└── UserFriendlyException          ← ABP — errore di business
```

Il `catch` cattura il tipo specificato **e tutti i suoi figli**:
```csharp
catch (Exception ex)  // cattura TUTTO — UserFriendlyException inclusa
catch (UserFriendlyException ex)  // cattura solo UserFriendlyException
```

**Ordine importante**: metti sempre i `catch` più specifici prima di quelli generici:
```csharp
catch (UserFriendlyException ex) { ... }  // ← prima
catch (Exception ex) { ... }              // ← dopo
```

---

## Come si usa nel progetto: `OrThrow` vs `OrNull`

Hai visto questa convenzione di naming nel progetto:

```csharp
// Lancia eccezione se non trovato
Task<AgencyInfoModel> GetAgencyOfUserOrThrowAsync(long userId);

// Restituisce null se non trovato
Task<AgencyInfoModel?> GetAgencyOfUserOrNullAsync(long userId);
```

L'implementazione di `OrNull` è esattamente un try/catch su `OrThrow`:

```csharp
public async Task<AgencyInfoModel?> GetAgencyOfUserOrNullAsync(long userId, ...)
{
    try
    {
        return await GetAgencyOfUserOrThrowAsync(userId, ...);
    }
    catch (UserFriendlyException)
    {
        return null;  // Non trovato → null invece di crash
    }
}
```

Questa convenzione è molto comune nei codebase .NET professionali.

---

## Eccezioni custom

Puoi creare le tue eccezioni estendendo `Exception`:

```csharp
public class AgencyNotFoundException : Exception
{
    public long UserId { get; }

    public AgencyNotFoundException(long userId)
        : base($"No agency found for user {userId}")
    {
        UserId = userId;
    }
}

// Uso
throw new AgencyNotFoundException(userId);

// Catch specifico
catch (AgencyNotFoundException ex)
{
    Logger.LogWarning("Agency missing for user {UserId}", ex.UserId);
}
```

Nel progetto usi `UserFriendlyException` di ABP invece di eccezioni custom —
è più pratico perché ABP gestisce automaticamente la risposta HTTP.

---

## Riepilogo

| Concetto | Sintassi |
|---|---|
| Blocco protetto | `try { ... }` |
| Cattura specifica | `catch (UserFriendlyException ex) { ... }` |
| Cattura generica | `catch (Exception ex) { ... }` |
| Sempre eseguito | `finally { ... }` |
| Lancia eccezione | `throw new Exception("messaggio")` |
| Eccezione business ABP | `throw new UserFriendlyException("messaggio")` |
| Rilancia stessa eccezione | `throw;` (senza `new`) |
