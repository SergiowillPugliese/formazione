# 4.6 — Unique Index e Constraint in EF Core

## Cos'è un indice nel database

Un **indice** (index) in SQL Server è una struttura dati che permette al database
di trovare righe velocemente senza scorrere tutta la tabella.

**Analogia**: è come l'indice analitico di un libro. Invece di leggere tutte le pagine
per trovare "posizionamento", guardi l'indice → vai direttamente a pagina 47.

---

## Cos'è un Unique Index

Un **Unique Index** aggiunge un vincolo: i valori in quella colonna (o combinazione
di colonne) devono essere unici. Il database rifiuta fisicamente qualsiasi insert
o update che violerebbe questa regola.

**Nel nostro caso**: nessun assortimento può avere la stessa posizione nello stesso
Store + Canale (`StoreChannelId`).

---

## Come si dichiara in EF Core

Nella configurazione dell'entità (tipicamente nel `DbContext` o in una classe
`IEntityTypeConfiguration<T>`):

```csharp
// Unique index su una singola colonna
entity.HasIndex(a => a.Email)
      .IsUnique();

// Unique index su più colonne (combinazione deve essere unica)
entity.HasIndex(a => new { a.StoreChannelId, a.Position })
      .IsUnique();
```

EF Core genera nella migration:
```sql
CREATE UNIQUE INDEX IX_Assortment_StoreChannelId_Position
    ON Assortment (StoreChannelId, Position);
```

---

## Il Filtered Index — per i nullable

C'è un problema: se `Position` è nullable (`int?`), SQL Server considera
i `NULL` come valori che violano l'unicità. Due righe con `Position = NULL`
farebbero scattare il constraint — ma non è quello che vogliamo.
(Tutti gli assortiment senza posizione coesistono pacificamente.)

La soluzione è il **Filtered Index** (indice filtrato): un indice che si applica
solo alle righe che soddisfano una condizione.

```csharp
// In EF Core — HasFilter esclude i NULL dall'indice
entity.HasIndex(a => new { a.StoreChannelId, a.Position })
      .IsUnique()
      .HasFilter("[Position] IS NOT NULL");  // ← solo le righe con posizione
```

SQL generato:
```sql
CREATE UNIQUE INDEX IX_Assortment_StoreChannelId_Position
    ON Assortment (StoreChannelId, Position)
    WHERE Position IS NOT NULL;   -- ← filtered index
```

Risultato:
- Due assortimenti con `Position = NULL` → ✅ permesso (non entrano nell'indice)
- Due assortimenti con `Position = 1` nello stesso StoreChannel → ❌ bloccato dal DB

---

## Cosa succede quando il constraint viene violato

Se provi a salvare un assortimento con una posizione già occupata, EF Core
lancia una `DbUpdateException` con un'`InnerException` che contiene il nome
dell'indice violato.

```csharp
try
{
    await _dbContext.SaveChangesAsync();
}
catch (DbUpdateException ex)
{
    // Puoi ispezionare il messaggio per capire quale constraint è stato violato
    if (ex.InnerException?.Message.Contains("IX_Assortment_StoreChannelId_Position") == true)
    {
        throw new UserFriendlyException("Questa posizione è già occupata da un altro assortimento.");
    }
    throw; // rilancia se è un errore diverso
}
```

---

## Unique Index vs Primary Key

| | Primary Key | Unique Index |
|---|-------------|--------------|
| Può essere null? | No | Sì (con filtered index) |
| Quanti per tabella? | Uno solo | Molti |
| Identifica la riga? | Sì | No |
| Uso tipico | `Id` | Email, codici, combinazioni univoche |

---

## La migration generata

Quando aggiungi un `HasIndex().IsUnique()` e generi una migration, EF Core crea:

```csharp
// File migration auto-generato
public partial class AddPositionToAssortment : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // 1. Aggiunge la colonna
        migrationBuilder.AddColumn<int>(
            name: "Position",
            table: "Assortment",
            type: "int",
            nullable: true);

        // 2. Crea l'indice filtrato
        migrationBuilder.CreateIndex(
            name: "IX_Assortment_StoreChannelId_Position",
            table: "Assortment",
            columns: new[] { "StoreChannelId", "Position" },
            unique: true,
            filter: "[Position] IS NOT NULL");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Rollback: rimuove indice e poi colonna
        migrationBuilder.DropIndex(
            name: "IX_Assortment_StoreChannelId_Position",
            table: "Assortment");

        migrationBuilder.DropColumn(
            name: "Position",
            table: "Assortment");
    }
}
```

`Up()` applica la migration, `Down()` la annulla — come `git revert` per lo schema DB.

---

## Riepilogo

| Concetto | Cosa fa |
|----------|---------|
| **Index** | Velocizza le ricerche nel DB (come l'indice di un libro) |
| **Unique Index** | Garantisce unicità — il DB rifiuta i duplicati |
| **Filtered Index** | Unique Index che si applica solo a righe che soddisfano una condizione |
| **HasIndex().IsUnique()** | Come si dichiara in EF Core |
| **HasFilter()** | Aggiunge la condizione al filtered index |
| **DbUpdateException** | Eccezione lanciata quando il constraint viene violato |
