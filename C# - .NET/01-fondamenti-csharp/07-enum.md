# 1.7 — Enum

## Il problema che risolve

Nel progetto, una `CustomizationRequest` può trovarsi in molti stati:
`Draft`, `PendingGraphicalApproval`, `GraphicallyApproved`, `CommerciallyApproved`...

Senza enum, useresti delle stringhe:
```csharp
// SBAGLIATO — stringhe libere
public string Status { get; set; } = "draft";
// Niente impedisce a qualcuno di scrivere "Draft", "DRAFT", "drft"...
```

Un **enum** (enumeration — "enumerazione") è un tipo che rappresenta
un insieme fisso di valori nominati. Il compilatore ti impedisce di usare
valori non validi.

---

## Sintassi base

```csharp
public enum CustomizationRequestStatus
{
    Draft,
    PendingRequesterApproval,
    RequesterApproved,
    PendingGraphicalApproval,
    GraphicallyApproved,
    PendingCommercialApproval,
    CommerciallyApproved,
    Rejected
}
```

Uso:
```csharp
var status = CustomizationRequestStatus.Draft;

if (status == CustomizationRequestStatus.Rejected)
{
    Console.WriteLine("La CR è stata rifiutata");
}
```

---

## Cosa c'è sotto: i numeri

Internamente, ogni valore di un enum è un intero.
Per default, il primo vale `0`, il secondo `1`, e così via:

```csharp
public enum CustomizationRequestStatus
{
    Draft = 0,
    PendingRequesterApproval = 1,
    RequesterApproved = 2,
    // ...
    Rejected = 7
}
```

Puoi convertire da e verso `int`:
```csharp
int n = (int)CustomizationRequestStatus.Draft;  // → 0
var status = (CustomizationRequestStatus)2;      // → RequesterApproved
```

---

## Analogia TypeScript

In TypeScript hai due equivalenti:

```typescript
// Opzione 1: union type (più comune in Angular moderno)
type Status = 'Draft' | 'Rejected' | 'Approved';

// Opzione 2: enum (simile a C#)
enum CustomizationRequestStatus {
    Draft = 0,
    Rejected = 7
}
```

In C# gli enum sono molto più usati dei union type di TypeScript,
perché il type system di C# è nominale (non strutturale).

---

## Come si usa nel progetto

```csharp
// Sull'entità CustomizationRequest
public CustomizationRequestStatus Status { get; private set; }

// Nel BuildRow dell'exporter (lo hai visto tu stesso!)
Status = cr.Status.ToString("G"),
// "G" = General format → restituisce il nome: "GraphicallyApproved"
```

`.ToString("G")` converte l'enum nel suo nome come stringa.
Senza `"G"` avresti il numero (`"5"` invece di `"GraphicallyApproved"`).

---

## Switch su enum

Il caso più comune: fare cose diverse in base allo stato.

```csharp
// Sintassi classica
switch (cr.Status)
{
    case CustomizationRequestStatus.Draft:
        Console.WriteLine("Bozza");
        break;
    case CustomizationRequestStatus.Rejected:
        Console.WriteLine("Rifiutata");
        break;
    default:
        Console.WriteLine("Altro stato");
        break;
}

// Sintassi moderna C# (switch expression)
var label = cr.Status switch
{
    CustomizationRequestStatus.Draft    => "Bozza",
    CustomizationRequestStatus.Rejected => "Rifiutata",
    _                                   => "Altro"   // _ = default
};
```

La switch expression è più compatta — funziona come un ternario
ma per più casi.

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| `enum NomeEnum { Val1, Val2 }` | Dichiara un enum |
| `NomeEnum.Val1` | Usa un valore |
| `(int)NomeEnum.Val1` | Converte a numero |
| `(NomeEnum)0` | Converte da numero |
| `.ToString("G")` | Converte al nome stringa |
| `switch` / `switch expression` | Gestisce i diversi valori |
