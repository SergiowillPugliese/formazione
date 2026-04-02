# 1.8 — Generics

## Il problema che risolve

Hai una funzione che trova il primo elemento di una lista.
Senza generics dovresti scriverla una volta per ogni tipo:

```csharp
// SBAGLIATO — codice duplicato per ogni tipo
public string TrovaPrimoStringa(List<string> lista) { ... }
public int TrovaPrimoInt(List<int> lista) { ... }
public Agency TrovaPrimaAgency(List<Agency> lista) { ... }
```

I **generics** ti permettono di scrivere codice che funziona con
qualsiasi tipo, decidendo il tipo concreto solo al momento dell'uso.

---

## La `T` — il segnaposto del tipo

`T` è una convenzione (potresti chiamarla `X` o `Tipo`, ma tutti usano `T`).
Significa: "questo tipo sarà specificato da chi usa questo codice".

```csharp
// Una sola funzione per tutti i tipi
public T TrovaPrimo<T>(List<T> lista)
{
    return lista[0];
}

// Uso
var primaStringa = TrovaPrimo<string>(new List<string> { "ciao", "mondo" }); // "ciao"
var primoNumero  = TrovaPrimo<int>(new List<int> { 42, 7, 3 });              // 42
```

---

## Analogia TypeScript

In TypeScript hai esattamente la stessa cosa:

```typescript
// TypeScript generics
function trovaPrimo<T>(lista: T[]): T {
    return lista[0];
}

const prima = trovaPrimo<string>(['ciao', 'mondo']); // 'ciao'
```

La sintassi è identica. Se hai usato generics in TypeScript, li conosci già.

---

## Dove li vedi nel progetto

Ovunque. I generics sono invisibili perché ci sei così abituato
che non li noti più:

```csharp
List<CustomizationRequest>          // lista di CR
Dictionary<long, AgencyInfoModel>   // dizionario con chiave long e valore AgencyInfoModel
Task<AgencyInfoDto>                  // operazione asincrona che restituisce un AgencyInfoDto
HashSet<long>                        // insieme di long
IEnumerable<string>                  // sequenza di stringhe
```

La `T` (o `TKey`, `TValue`, `TResult`) tra `< >` è sempre un parametro generico.

---

## Classi generiche

Non solo le funzioni possono essere generiche — anche le classi:

```csharp
// Dal progetto: IFourShopEfCoreRepository<TEntity, TKey>
public interface IFourShopEfCoreRepository<TEntity, TKey>
{
    Task<TEntity> GetAsync(TKey id);
    Task<List<TEntity>> GetListAsync();
    Task DeleteAsync(TKey id);
}

// Uso concreto
public interface ICustomizationRequestRepository
    : IFourShopEfCoreRepository<CustomizationRequest, long>
{
    // Aggiunge metodi specifici sopra ai generici
}
```

`IFourShopEfCoreRepository<TEntity, TKey>` è una classe generica con
due parametri: il tipo dell'entità e il tipo della sua chiave primaria.
Definisce le operazioni CRUD una volta sola — funziona per qualsiasi entità.

---

## Vincoli sui generics (`where T : ...`)

A volte vuoi che `T` sia solo un certo tipo di oggetto.
Puoi aggiungere vincoli con `where`:

```csharp
// T deve essere una classe (reference type)
public void Salva<T>(T entità) where T : class { ... }

// T deve implementare un'interfaccia
public void Processa<T>(T item) where T : IEntity { ... }

// T deve avere un costruttore senza parametri
public T Crea<T>() where T : new() { return new T(); }
```

Nel progetto vedi `where TEntity : class` sulle classi base dei repository —
significa "TEntity deve essere una classe, non un int o un bool".

---

## Riepilogo

| Sintassi | Significato |
|---|---|
| `Metodo<T>(T valore)` | Metodo generico con parametro di tipo T |
| `Classe<T>` | Classe generica |
| `List<string>` | Lista di stringhe (T = string) |
| `Task<AgencyDto>` | Task che restituisce AgencyDto (T = AgencyDto) |
| `where T : class` | Vincolo: T deve essere una classe |
| `where T : IEntity` | Vincolo: T deve implementare IEntity |
