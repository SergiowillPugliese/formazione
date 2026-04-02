# 06 — Record e immutabilità

## Il problema con le classi mutabili

Con una normale classe C#, puoi modificare i dati dopo la creazione:

```csharp
public class ReportRow
{
    public string Title { get; set; }
    public string Status { get; set; }
}

var row = new ReportRow { Title = "Test", Status = "Draft" };
row.Title = "Modificato dopo"; // possibile! e potenzialmente pericoloso
```

Se passi `row` a 5 metodi diversi, uno di loro potrebbe modificarlo
e gli altri riceverebbero dati diversi da quelli attesi.
Questo tipo di bug è difficile da trovare.

---

## La soluzione: `record`

Un `record` è un tipo speciale introdotto in C# 9 pensato per dati immutabili.
Una volta creato, non puoi modificarlo.

```csharp
public record CustomizationRequestReportRow
{
    public required string Title { get; init; }
    public required string Status { get; init; }
    public DateTime? DeliveryDate { get; init; }
}
```

---

## `init` — assegnabile solo alla costruzione

`init` è come `set` ma funziona solo nell'object initializer.
Dopo la creazione, la property è in sola lettura:

```csharp
var row = new CustomizationRequestReportRow
{
    Title = "Test",   // OK — siamo nell'object initializer
    Status = "Draft"  // OK
};

row.Title = "Cambiato"; // ERRORE DI COMPILAZIONE — init non lo permette
```

In TypeScript il corrispettivo è `readonly`:
```typescript
interface ReportRow {
    readonly title: string;
    readonly status: string;
}
```

---

## `required` — obbligatorio alla costruzione

`required` dice al compilatore: "questa property deve essere valorizzata
quando crei l'oggetto. Se la dimentichi, è un errore di compilazione."

```csharp
public record CustomizationRequestReportRow
{
    public required string Title { get; init; }   // obbligatorio
    public DateTime? DeliveryDate { get; init; }  // opzionale (nullable)
}

// ERRORE — Title è required ma manca
var row = new CustomizationRequestReportRow
{
    // Title non c'è!
    DeliveryDate = DateTime.UtcNow
};

// OK — tutti i required sono valorizzati
var row = new CustomizationRequestReportRow
{
    Title = "Test",
    DeliveryDate = DateTime.UtcNow
};
```

In TypeScript è come `Required<T>` o semplicemente dichiarare una property
senza `?`:
```typescript
interface ReportRow {
    title: string;           // obbligatorio
    deliveryDate?: Date;     // opzionale
}
```

---

## Perché usare `record` invece di `class`?

Nel progetto, `CustomizationRequestReportRow` è un record perché:

1. **È solo un contenitore di dati** — non ha logica, non ha metodi
2. **Non deve cambiare dopo la creazione** — viene costruito in `BuildRow`
   e poi passato a `GenerateExcelWorkbook` che lo legge
3. **Il compilatore garantisce la correttezza** — `required` assicura
   che ogni riga abbia tutti i dati necessari

---

## Differenza tra `record` e `class`

| Caratteristica | `class` | `record` |
|---|---|---|
| Mutabile di default | Sì | No (con `init`) |
| Uguaglianza | Per riferimento | Per valore |
| Pensato per | Logica e comportamento | Dati immutabili |
| Clonazione con `with` | No | Sì |

**Uguaglianza per valore** — due record con gli stessi dati sono considerati uguali:

```csharp
var row1 = new CustomizationRequestReportRow { Title = "Test", Status = "Draft" /* ... */ };
var row2 = new CustomizationRequestReportRow { Title = "Test", Status = "Draft" /* ... */ };

// Con class: false (sono oggetti diversi in memoria)
// Con record: true (hanno gli stessi valori)
bool uguali = row1 == row2; // true per i record
```

---

## `with` — creare una copia modificata

Con i record puoi creare una copia con alcune property cambiate
senza modificare l'originale:

```csharp
var rowOriginale = new CustomizationRequestReportRow
{
    Title = "Test",
    Status = "Draft",
    // ...
};

// Crea una nuova copia con solo Status cambiato
var rowAggiornata = rowOriginale with { Status = "Approved" };

// rowOriginale è ancora immutata
// rowAggiornata è una nuova istanza con Status = "Approved"
```

È come lo spread operator in TypeScript:
```typescript
const rowAggiornata = { ...rowOriginale, status: "Approved" };
```

---

## Nel progetto

```csharp
// In BuildRow — costruiamo il record una volta, poi non cambia più
return new CustomizationRequestReportRow {
    Id = cr.Id,
    Title = cr.Title,
    Status = cr.Status.ToString("G"),
    AgencyCode = agencyInfo?.Code,        // nullable: può essere null
    AgencyName = agencyInfo?.Name,        // nullable: può essere null
    TotalQuantity = totalQuantity,
    // ...
};
// Dopo questa riga, nessuno può modificare i dati di questa riga
```
