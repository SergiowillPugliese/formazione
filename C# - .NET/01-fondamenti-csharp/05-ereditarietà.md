# 05 — Ereditarietà

## Il concetto

L'ereditarietà permette a una classe di **ereditare** comportamenti e dati
da un'altra classe. La classe "figlia" ha tutto ciò che ha la classe "genitore",
più le proprie aggiunte.

In Angular hai già visto questo quando un componente estende un altro,
o quando usi classi base condivise:

```typescript
// TypeScript
class Animal {
    name: string;
    speak(): string { return "..."; }
}

class Dog extends Animal {
    speak(): string { return "Woof!"; }
}
```

In C# funziona allo stesso modo:

```csharp
// C#
public class Animal
{
    public string Name { get; set; }
    public virtual string Speak() { return "..."; }
}

public class Dog : Animal  // ":" invece di "extends"
{
    public override string Speak() { return "Woof!"; }
}
```

---

## Nel progetto: `FullAuditedEntity4ShopWithRequiredCreator<long>`

`CustomizationRequest` estende questa classe base:

```csharp
public sealed class CustomizationRequest
    : FullAuditedEntity4ShopWithRequiredCreator<long>, IMultiTenant
```

Significa che `CustomizationRequest` eredita automaticamente questi campi
senza doverli dichiarare:

```
Id                  ← la chiave primaria (il <long> specifica il tipo)
CreationTime        ← quando è stata creata
CreatorId           ← chi l'ha creata
LastModificationTime ← ultima modifica
LastModifierId      ← chi ha modificato l'ultima volta
IsDeleted           ← soft delete (eliminazione logica, non fisica)
DeletionTime        ← quando è stata eliminata
DeleterId           ← chi l'ha eliminata
```

Questo è il vantaggio dell'ereditarietà: scrive queste property una volta
nella classe base, e tutte le entità del progetto le hanno automaticamente.

---

## `virtual` e `override`

In C# per permettere a una classe figlia di sovrascrivere un metodo,
il metodo del genitore deve essere dichiarato `virtual`:

```csharp
public class CatalogBaseAppService
{
    // virtual = può essere sovrascritto dalle figlie
    public virtual async Task<AgencyGetDto> GetAsync(long id)
    {
        // implementazione base
    }
}

public class AgenciesAppService : CatalogBaseAppService
{
    // override = sto sostituendo l'implementazione del genitore
    public override async Task<AgencyGetDto> GetAsync(long id)
    {
        // implementazione specifica per le agenzie
    }
}
```

In TypeScript non hai bisogno di `virtual` — tutti i metodi sono sovrascrivibili
di default.

---

## `abstract` — metodi senza implementazione

Una classe `abstract` è una classe che non può essere istanziata direttamente
(non puoi fare `new AbstractClass()`). Esiste solo per essere ereditata.

Un metodo `abstract` è un metodo dichiarato ma senza corpo — le classi figlie
sono obbligate a implementarlo:

```csharp
public abstract class NotificationSender
{
    // abstract = nessuna implementazione qui, la figlia DEVE implementarlo
    public abstract Task SendAsync(string message, CancellationToken ct);

    // metodo concreto — tutte le figlie lo ereditano così com'è
    public async Task SendWithRetryAsync(string message, int maxRetries, CancellationToken ct)
    {
        for (var i = 0; i < maxRetries; i++)
        {
            await SendAsync(message, ct); // chiama il metodo astratto
        }
    }
}

public class EmailNotificationSender : NotificationSender
{
    public override async Task SendAsync(string message, CancellationToken ct)
    {
        // implementazione email
    }
}
```

---

## `sealed` — bloccare l'ereditarietà

`sealed` impedisce che una classe venga ulteriormente ereditata.

```csharp
// CustomizationRequest non può essere estesa da altre classi
public sealed class CustomizationRequest : FullAuditedEntity4ShopWithRequiredCreator<long>
{
    // ...
}
```

Si usa quando una classe ha una logica molto specifica che non ha senso estendere,
e per motivi di performance (il compilatore può ottimizzare meglio).

---

## `base` — chiamare il costruttore del genitore

Quando una classe figlia ha un costruttore, può chiamare il costruttore
del genitore con `base(...)`:

```csharp
public sealed class CustomizationRequest : FullAuditedEntity4ShopWithRequiredCreator<long>
{
    // Costruttore che chiama quello del genitore passando l'id
    internal CustomizationRequest(long id) : base(id) { }
}
```

In TypeScript è `super(id)`:
```typescript
class CustomizationRequest extends FullAuditedEntity {
    constructor(id: number) {
        super(id); // chiama il costruttore del genitore
    }
}
```

---

## `IMultiTenant` — interfacce come "etichette"

Nota che `CustomizationRequest` implementa anche `IMultiTenant`:

```csharp
public sealed class CustomizationRequest
    : FullAuditedEntity4ShopWithRequiredCreator<long>, IMultiTenant
```

In C# si può ereditare da **una sola classe** ma implementare **molte interfacce**.
`IMultiTenant` qui funziona come una "etichetta" — dice al framework ABP
"questa entità appartiene a un tenant specifico, applica automaticamente
il filtro per tenant in tutte le query".

---

## Riepilogo visivo

```
FullAuditedEntity4ShopWithRequiredCreator<long>   ← classe base
    Id, CreationTime, CreatorId, LastModificationTime...

        ↓ ereditata da

CustomizationRequest                               ← classe figlia
    Title, Status, PointOfSaleId...               ← campi aggiuntivi
    + tutti i campi ereditati dalla base
```
