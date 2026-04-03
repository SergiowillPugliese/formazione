# 5.6 — Bulk Operations: aggiornare molti record in un colpo

## Cos'è una bulk operation

Una **bulk operation** è un'operazione che lavora su un insieme di record
in una singola chiamata, invece di fare tante chiamate una per record.

**Esempio concreto**: il frontend vuole impostare il posizionamento di 20 assortimenti.

```
❌ Approccio non-bulk: 20 chiamate separate
PUT /v1/assortments/1  { position: 1 }
PUT /v1/assortments/5  { position: 2 }
PUT /v1/assortments/3  { position: 3 }
... (17 altre chiamate)

✅ Approccio bulk: 1 sola chiamata
PUT /v1/assortments/bulk-positions
{
  "storeChannelId": 5,
  "positions": [
    { "assortmentId": 1, "position": 1 },
    { "assortmentId": 5, "position": 2 },
    { "assortmentId": 3, "position": 3 },
    ...
  ]
}
```

---

## Partial update vs Replace-all: due strategie diverse

Quando si progetta un'operazione bulk di aggiornamento, esistono due approcci:

### Partial update (aggiornamento parziale)

Il payload modifica solo i record presenti nella lista. Gli altri rimangono invariati.

```
Stato iniziale: A=pos1, B=pos2, C=pos3
Payload: [{ B, pos5 }]
Stato finale:  A=pos1, B=pos5, C=pos3
```

**Pro**: flessibile, si può aggiornare un sottoinsieme
**Contro**: più complesso da gestire lato backend; può creare incoerenze
se il frontend non ha una visione aggiornata dello stato

### Replace-all (sostituzione completa)

Il payload **sostituisce completamente** le posizioni per un dato scope.
Prima azzera tutto, poi assegna solo quello che è nella lista.

```
Stato iniziale: A=pos1, B=pos2, C=pos3
Payload: [{ A, pos3 }, { C, pos1 }]   (B non è nel payload)
Stato finale:  A=pos3, B=null, C=pos1
```

**Pro**: semplice da implementare; il frontend ha sempre controllo totale
**Contro**: il frontend deve mandare SEMPRE la lista completa degli assortimenti
che devono avere una posizione

**Nel progetto abbiamo scelto replace-all** — è la scelta giusta per il
posizionamento: il frontend mostra una lista ordinata e salva l'ordine completo.

---

## Implementazione del replace-all

```csharp
// Endpoint: PUT /v1/assortments/bulk-positions
public async Task BulkUpdatePositionsAsync(BulkPositionsInputDto input)
{
    // 1. Validazione del payload stesso (prima di toccare il DB)
    ValidateNoDuplicatePositions(input.Positions);
    await ValidateAllAssortmentsBelongToStoreChannelAsync(input);

    using (var uow = _unitOfWorkManager.Begin())
    {
        // 2. Azzera tutte le posizioni per questo StoreChannel
        var existing = await _assortmentRepo.GetListAsync(
            a => a.StoreChannelId == input.StoreChannelId && a.Position != null);

        foreach (var assortment in existing)
        {
            assortment.SetPosition(null);  // ← setter privato nel Domain
        }

        // 3. Assegna le nuove posizioni
        foreach (var item in input.Positions)
        {
            var assortment = await _assortmentRepo.GetAsync(item.AssortmentId);
            assortment.SetPosition(item.Position);
        }

        // 4. Commit — se qualcosa è andato storto prima, rollback automatico
        await uow.CompleteAsync();
    }
}
```

---

## Validazione del payload: trovare duplicati nell'input

Prima di toccare il database, devi verificare che il payload stesso non abbia
due righe con la stessa posizione:

```csharp
private static void ValidateNoDuplicatePositions(IEnumerable<AssortmentPositionDto> positions)
{
    var seen = new HashSet<int>();
    foreach (var item in positions)
    {
        if (!seen.Add(item.Position))  // Add ritorna false se già presente
        {
            throw new UserFriendlyException(
                $"La posizione {item.Position} è presente più volte nel payload");
        }
    }
}
```

`HashSet<T>` è la struttura dati giusta qui: inserimento e lookup in O(1),
e `.Add()` ritorna `false` se l'elemento esiste già — perfetto per rilevare duplicati.

---

## Perché il bulk deve essere atomico

Senza atomicità, se il bulk fallisce a metà:

```
Step 1: azzera tutte le posizioni  → ✅ eseguito
Step 2: assegna pos 1 ad A         → ✅ eseguito
Step 3: assegna pos 2 a B          → ❌ ERRORE
                                      Stato: A ha pos 1, tutti gli altri null
                                      Le posizioni di B, C, D... sono andate perse!
```

Con la Unit of Work (vedi modulo 5.5):
```
Step 1: azzera  → in memoria
Step 2: assegna → in memoria
Step 3: assegna → ❌ ERRORE → ROLLBACK
                   Il DB torna allo stato iniziale.
                   Nessuna perdita di dati.
```

---

## Il DTO per il bulk

```csharp
// Input per il bulk endpoint
public class BulkPositionsInputDto
{
    [Required]
    public int StoreChannelId { get; set; }

    [Required]
    public List<AssortmentPositionDto> Positions { get; set; } = new();
}

public class AssortmentPositionDto
{
    public int AssortmentId { get; set; }
    public int Position { get; set; }  // non nullable: se sei in lista, hai una posizione
}
```

`Position` nel DTO non è nullable: se un assortimento è nella lista del bulk,
per forza ha una posizione. Gli assortimenti che non devono avere posizione
semplicemente non vengono inclusi nel payload.

---

## Riepilogo

| Concetto | Spiegazione |
|----------|-------------|
| **Bulk operation** | Una sola chiamata che aggiorna molti record |
| **Partial update** | Modifica solo i record nel payload, altri invariati |
| **Replace-all** | Azzera tutto, poi riassegna solo quelli nel payload |
| **Validazione payload** | Controlla duplicati nell'input prima di toccare il DB |
| **Atomicità** | Obbligatoria per il bulk — usa Unit of Work |
| **HashSet** | Struttura dati per rilevare duplicati in O(1) |
