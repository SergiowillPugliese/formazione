# 5.5 — Unit of Work: atomicità e transazioni

## Il problema che ci ha fatto scoprire questo concetto

Nel progetto del posizionamento bulk degli assortimenti, la logica è:
1. Azzera tutte le posizioni esistenti per un determinato Store + Canale
2. Assegna le nuove posizioni dalla lista del frontend

Se il passo 2 fallisce a metà (es. errore al terzo assortimento su dieci),
cosa succede al passo 1 già eseguito? Le posizioni sono state azzerate,
ma le nuove non sono state assegnate completamente. Lo stato del database
è incoerente.

La soluzione si chiama **Unit of Work**.

---

## Cos'è una Unit of Work

Una **Unit of Work** (letteralmente: "unità di lavoro") raggruppa più operazioni
sul database in un'unica transazione. Il risultato è binario:

- Tutte le operazioni vanno a buon fine → **commit** (le modifiche vengono salvate)
- Una qualsiasi operazione fallisce → **rollback** (il database torna allo stato iniziale)

**Analogia dalla vita reale**: pensa al pagamento con carta di credito.
Ci sono più passaggi (autorizzazione, addebito, accredito). Se uno fallisce,
tutti gli altri vengono annullati. Non puoi avere "solo l'addebito senza l'accredito".

**Analogia Angular**: non esiste un equivalente diretto, perché il frontend
non scrive mai direttamente nel DB. Ma pensa a `Promise.all()` — solo che invece
di "tutte le promise si risolvono", qui è "tutte le scritture DB vengono committate".

---

## Come funziona in ABP Framework

ABP gestisce automaticamente la Unit of Work per le richieste HTTP normali.
Ogni chiamata API ha già una transazione implicita: se l'AppService lancia
un'eccezione, il rollback è automatico.

Per i **background jobs** (Hangfire) e per operazioni particolari come il bulk,
devi aprirla esplicitamente:

```csharp
// Pattern standard per background jobs e bulk operations
using (var uow = _unitOfWorkManager.Begin())
{
    // Tutto quello che scrivi qui è dentro la stessa transazione

    // Step 1: azzera le posizioni esistenti
    await AzzeraPosizioniAsync(storeChannelId);

    // Step 2: assegna le nuove posizioni
    foreach (var item in input.Positions)
    {
        await AssegnaPosizioneAsync(item.AssortmentId, item.Position);
    }

    // Se siamo arrivati qui senza eccezioni: salva tutto
    await uow.CompleteAsync();  // ← COMMIT
}
// Se un'eccezione è stata lanciata prima di CompleteAsync: ROLLBACK automatico
```

`using` garantisce che la UoW venga sempre chiusa (anche in caso di eccezione)
grazie al pattern `IDisposable` — come `using (var stream = File.Open(...))` che
chiude sempre il file.

---

## Cosa significa "atomico"

**Atomico** vuol dire indivisibile — o succede tutto, o non succede niente.

```
SENZA Unit of Work:
─────────────────────────────
Azzera posizioni  → ✅ salvato subito nel DB
Assegna pos 1     → ✅ salvato subito nel DB
Assegna pos 2     → ✅ salvato subito nel DB
Assegna pos 3     → ❌ ERRORE
                     Il DB è in uno stato parziale:
                     posizioni azzerate, solo 2 riassegnate su 10

CON Unit of Work:
─────────────────────────────
Azzera posizioni  → in memoria (non ancora nel DB)
Assegna pos 1     → in memoria
Assegna pos 2     → in memoria
Assegna pos 3     → ❌ ERRORE → ROLLBACK
                     Il DB torna allo stato iniziale.
                     Nessuna modifica è stata applicata.
```

---

## Unit of Work nei background jobs

Come vedi in `CLAUDE.md`, **tutti i background job Hangfire devono wrappare
le query DB in una Unit of Work esplicita**:

```csharp
public class MyJobProcessor
{
    private readonly IUnitOfWorkManager _unitOfWorkManager;

    public async Task ProcessAsync(CancellationToken ct)
    {
        using (_unitOfWorkManager.Begin())
        {
            // Tutte le operazioni DB qui
            var assortments = await _repo.GetListAsync();
            foreach (var a in assortments)
            {
                a.SetPosition(null);
            }
        } // ← CompleteAsync implicito (il using chiude la UoW)
    }
}
```

Questo è necessario perché i job Hangfire non passano attraverso il ciclo
normale di una richiesta HTTP — ABP non apre automaticamente la UoW per loro.

---

## La differenza tra UoW e SaveChanges

In EF Core puro, chiami `SaveChangesAsync()` per salvare. In ABP, `SaveChangesAsync()`
esiste ancora ma la UoW aggiunge uno strato sopra:

| | EF Core puro | ABP con Unit of Work |
|---|---|---|
| Come si salva | `_dbContext.SaveChangesAsync()` | `uow.CompleteAsync()` |
| Transazione implicita? | No (devi aprirla tu) | Sì (per richieste HTTP) |
| Rollback automatico | No | Sì (se non chiami `CompleteAsync`) |

---

## Riepilogo

| Concetto | Spiegazione |
|----------|-------------|
| **Unit of Work** | Raggruppa più operazioni DB in una transazione unica |
| **Commit** | Tutte le operazioni riuscite → le modifiche vengono salvate |
| **Rollback** | Una operazione fallisce → il DB torna allo stato precedente |
| **Atomico** | O tutto o niente — nessuno stato intermedio |
| **`uow.CompleteAsync()`** | Esegue il commit esplicitamente |
| **`using (_unitOfWorkManager.Begin())`** | Apre la UoW — rollback automatico se non si chiama `CompleteAsync` |
| **Background jobs** | Richiedono UoW esplicita (non è aperta automaticamente come nelle richieste HTTP) |
