# 01 — Tipi fondamentali in C#

## Il concetto

C# è un linguaggio **fortemente tipizzato** — ogni variabile deve avere un tipo dichiarato
e il compilatore ti impedisce di assegnare il tipo sbagliato.

TypeScript funziona allo stesso modo, ma C# è ancora più rigido: non esiste `any`,
non esiste coercizione implicita tra tipi diversi.

---

## I tipi base

### Numeri interi

```csharp
int numero = 42;         // 32 bit, da -2 miliardi a +2 miliardi
long numeroGrande = 42;  // 64 bit, da -9.2 trilioni a +9.2 trilioni
```

In TypeScript hai un solo `number` per tutto. In C# la distinzione esiste perché
occupa memoria diversa e ha limiti diversi.

**Quando usare `long` invece di `int`?**
Quando il valore può superare i 2 miliardi — tipicamente per chiavi primarie
del database (`Id`), contatori, timestamp. Nel progetto, `CustomizationRequest.Id`
è `long` perché nel tempo il numero di record può crescere enormemente.

```typescript
// TypeScript — un solo tipo
const id: number = 42;
```

```csharp
// C# — due tipi distinti, con significati diversi
int paginaAttuale = 3;     // piccolo, non supererà mai 2 miliardi
long customizationRequestId = 123456789012; // potenzialmente enorme
```

**Bug reale del progetto**: avevamo dichiarato `Id` come `int` in `CustomizationRequestReportRow`
ma l'entità aveva `Id` come `long`. Il compilatore ha segnalato errore perché non può
convertire automaticamente — potrebbe perdere dati.

---

### Numeri decimali

```csharp
decimal prezzo = 19.99m;   // precisione esatta, usato per soldi
double altezza = 1.75;     // virgola mobile, può avere errori di arrotondamento
float peso = 70.5f;        // come double ma meno preciso
```

**Regola pratica**: per soldi usa sempre `decimal`. Per calcoli scientifici usa `double`.
Nel progetto, `PersonalizationCost`, `ShippingCost`, `DiscountPercentage` sono `decimal`.

La `m` finale in `19.99m` dice al compilatore "questo è un decimal, non un double".

---

### Testo

```csharp
string nome = "Mario";
string vuoto = "";
string nullo = null;       // attenzione: può causare NullReferenceException
```

```typescript
// TypeScript
const nome: string = "Mario";
```

In C# moderno (dalla versione 8), puoi abilitare i **nullable reference types**
che distinguono `string` (non può essere null) da `string?` (può essere null).
Lo vedremo nel modulo 02.

---

### Booleani

```csharp
bool attivo = true;
bool disattivo = false;
```

Identico a TypeScript. In C# si scrive `bool`, non `boolean`.

---

### Date

```csharp
DateTime adesso = DateTime.UtcNow;
DateTime data = new DateTime(2024, 12, 25);
DateTime? dataOpzionale = null;  // può non esserci
```

Non esiste un tipo `Date` come in JavaScript. `DateTime` include sia la data che l'ora.
Nel progetto usiamo sempre `DateTime?` (nullable) per le date di approvazione perché
una CR non ancora approvata non ha data.

---

### Confronto TypeScript → C#

| TypeScript | C# | Note |
|---|---|---|
| `number` | `int` | per numeri piccoli |
| `number` | `long` | per ID e numeri grandi |
| `number` | `decimal` | per soldi |
| `string` | `string` | identico |
| `boolean` | `bool` | nome diverso |
| `Date` | `DateTime` | include ora, non solo data |
| `null` / `undefined` | `null` | solo `null` in C# |

---

## Nel progetto

In `CustomizationRequestReportRow` abbiamo usato tutti questi tipi:

```csharp
public record CustomizationRequestReportRow
{
    public required long Id { get; init; }          // long: chiave primaria
    public required string Title { get; init; }     // string: testo
    public required string Status { get; init; }    // string: enum convertito
    public required int TotalQuantity { get; init; } // int: quantità, non supera miliardi
    public DateTime? DeliveryDate { get; init; }    // DateTime?: opzionale
    public decimal? PersonalizationCost { get; init; } // decimal?: soldi, opzionale
}
```
