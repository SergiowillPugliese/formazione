# 3.3 — Race Conditions: quando due richieste arrivano insieme

## Il problema che ci ha fatto scoprire questo concetto

Durante la progettazione del posizionamento degli assortimenti, abbiamo detto:

> "Dobbiamo validare che non ci siano due assortimenti con la stessa posizione per lo stesso Store + Canale."

La prima idea istintiva è: prima di salvare, controlla se la posizione è già occupata.

```csharp
// Idea iniziale — sembra logica, ma ha un problema
public async Task<AssortmentGetDto> CreateAsync(AssortmentInsertDto input)
{
    if (input.Position.HasValue)
    {
        var alreadyTaken = await repo.AnyAsync(a =>
            a.StoreChannelId == input.StoreChannelId &&
            a.Position == input.Position);

        if (alreadyTaken)
            throw new UserFriendlyException("Posizione già occupata");
    }

    // ... salva
}
```

**Il problema**: questa logica ha una falla nascosta chiamata **race condition**.

---

## Cos'è una race condition

Una race condition è un bug che si verifica quando **due operazioni concorrenti**
producono un risultato diverso da quello che produrrebbero se eseguite in sequenza.

**Analogia dalla vita reale**: immagina due persone che guardano contemporaneamente
il numero disponibile in biglietteria. Entrambe vedono "1 biglietto rimasto",
entrambe comprano. Risultato: over-booking.

Nel nostro caso:

```
Richiesta A                    Richiesta B
─────────────────────          ─────────────────────
Controlla: posizione 1 libera? Controlla: posizione 1 libera?
→ SÌ (il check passa)          → SÌ (il check passa)
Salva assortimento con pos 1   Salva assortimento con pos 1
                               ← Due record con posizione 1!
```

Tra il controllo e il salvataggio c'è un istante di tempo. In quel momento,
un'altra richiesta può fare lo stesso controllo — e passarlo — prima che il
primo salvataggio sia completato.

---

## Perché succede in un'API REST

Le API REST gestiscono migliaia di richieste in parallelo. Il web server
(ASP.NET Core) esegue ogni richiesta in un thread separato, contemporaneamente.

**Analogia Angular**: pensa a `Promise.all([chiamataA, chiamataB])` — entrambe
partono contemporaneamente. Se entrambe scrivono sullo stesso dato, non sai
quale finirà per ultima.

---

## La soluzione: due livelli di protezione

Per gestire le race conditions sui dati, si usano **due livelli**:

### Livello 1 — Validazione applicativa (user-friendly)

```csharp
// Nel AppService — prima di salvare
if (input.Position.HasValue)
{
    var alreadyTaken = await repo.AnyAsync(a =>
        a.StoreChannelId == storeChannelId &&
        a.Position == input.Position &&
        a.Id != currentId);  // escludi se stesso in caso di update

    if (alreadyTaken)
        throw new UserFriendlyException("Posizione già occupata per questo Store + Canale");
}
```

Questo gestisce il **caso normale**: se un utente prova a impostare una posizione
già usata, riceve un messaggio chiaro. Funziona nel 99% dei casi.

### Livello 2 — Constraint sul database (sicurezza assoluta)

```sql
-- Unique index con filtered index (solo dove Position non è null)
CREATE UNIQUE INDEX IX_Assortment_StoreChannelId_Position
    ON Assortment (StoreChannelId, Position)
    WHERE Position IS NOT NULL;
```

In EF Core (nel `DbContext` o nella migration):
```csharp
entity.HasIndex(a => new { a.StoreChannelId, a.Position })
      .IsUnique()
      .HasFilter("[Position] IS NOT NULL");
```

Il DB **garantisce fisicamente** che non possano esistere due righe con lo stesso
`StoreChannelId + Position`. Se due richieste arrivano simultaneamente e passano
entrambe il controllo applicativo, solo una riuscirà a salvare. L'altra riceverà
un'eccezione dal database.

---

## Come gestire l'eccezione del DB constraint

Quando il DB rifiuta l'insert per violazione del constraint, lancia un'eccezione.
Devi catturarla e trasformarla in un messaggio leggibile:

```csharp
try
{
    await _unitOfWorkManager.Current.SaveChangesAsync();
}
catch (DbUpdateException ex) when (ex.InnerException?.Message.Contains("IX_Assortment_StoreChannelId_Position") == true)
{
    throw new UserFriendlyException("Posizione già occupata: un altro assortimento ha preso questa posizione nel frattempo.");
}
```

Questo messaggio all'utente è diverso da quello del Livello 1: suggerisce che
il conflitto è avvenuto "nel frattempo" — cioè in modo concorrente.

---

## Perché non basta solo il constraint DB

Senza la validazione applicativa, l'utente riceverebbe un errore generico del
database invece di un messaggio comprensibile. I due livelli si completano:

| Livello | Gestisce | Messaggio all'utente |
|---------|----------|----------------------|
| Validazione app | Caso normale (99%) | "Posizione già occupata" — chiaro |
| DB constraint | Caso concorrente (1%) | "Qualcuno ha preso la posizione nel frattempo" |
| Solo validazione app | ❌ Non protegge dalla concorrenza | Dati inconsistenti nel DB |
| Solo DB constraint | ❌ Messaggio tecnico incomprensibile | Errore del database grezzo |

---

## Riepilogo

| Concetto | Spiegazione |
|----------|-------------|
| **Race condition** | Due richieste concorrenti producono un risultato incoerente |
| **Validazione applicativa** | Controllo nel codice — user-friendly, ma non atomico |
| **DB constraint (unique index)** | Garanzia fisica del database — sempre affidabile |
| **Pattern corretto** | Entrambi insieme: validazione app per UX, DB constraint per correttezza |
