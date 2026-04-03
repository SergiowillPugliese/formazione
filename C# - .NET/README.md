# Corso C# / .NET — Da Angular Developer a Full Stack

Corso costruito sui concetti reali incontrati durante lo sviluppo del modulo Kustom
nel progetto 4Shop CatalogService. Ogni lezione parte da ciò che già conosci (Angular, TypeScript)
e costruisce il ponte verso C# e .NET.

## A chi è rivolto

- Hai esperienza in Angular e TypeScript
- Non hai mai fatto OOP "vera" (classi, interfacce, ereditarietà)
- Sei stato catapultato su un progetto .NET e vuoi capire cosa stai leggendo

## Come usare questo corso

Leggi i moduli nell'ordine della sezione. Ogni modulo ha:
- La spiegazione del concetto
- L'analogia con Angular/TypeScript che già conosci
- Il codice reale preso dal progetto

---

## Sezione 1 — Fondamenti C#
> Il linguaggio puro: tipi, classi, oggetti, interfacce.

| # | Modulo | Concetti chiave |
|---|--------|-----------------|
| 1.1 | [Tipi fondamentali](./01-fondamenti-csharp/01-tipi-fondamentali.md) | int, long, string, bool, decimal, DateTime |
| 1.2 | [Nullable types](./01-fondamenti-csharp/02-nullable.md) | ?, .HasValue, .Value, ?., ?? |
| 1.3 | [Classi e oggetti](./01-fondamenti-csharp/03-classi-e-oggetti.md) | class, costruttore, property, istanziazione |
| 1.4 | [Interfacce](./01-fondamenti-csharp/04-interfacce.md) | interface, implementazione, perché esistono |
| 1.5 | [Ereditarietà](./01-fondamenti-csharp/05-ereditarietà.md) | base class, override, abstract, sealed |
| 1.6 | [Record e immutabilità](./01-fondamenti-csharp/06-record-e-immutabilità.md) | record, required, init |
| 1.7 | [Enum](./01-fondamenti-csharp/07-enum.md) | enum, cast, switch, flag |
| 1.8 | [Generics](./01-fondamenti-csharp/08-generics.md) | T, vincoli, List\<T\>, Dictionary\<K,V\> |
| 1.9 | [Exception handling](./01-fondamenti-csharp/09-exception-handling.md) | try/catch/finally, throw, eccezioni custom |

## Sezione 2 — Collezioni e LINQ
> Come lavorare con insiemi di dati in memoria e verso il database.

| # | Modulo | Concetti chiave |
|---|--------|-----------------|
| 2.1 | [Collezioni](./02-collezioni-e-linq/01-collezioni.md) | List, HashSet, Dictionary |
| 2.2 | [LINQ](./02-collezioni-e-linq/02-linq-base.md) | Select, Where, ToDictionary, Any, GroupBy, First |
| 2.3 | [IEnumerable vs IQueryable](./02-collezioni-e-linq/03-ienumerable-vs-iqueryable.md) | Differenza in memoria vs database, EF Core |
| 2.4 | [Ordinamento](./02-collezioni-e-linq/04-ordinamento.md) | OrderBy, ThenBy, nullable sorting, Dynamic LINQ |

## Sezione 3 — Async e Concorrenza
> Come funziona il codice asincrono in .NET.

| # | Modulo | Concetti chiave |
|---|--------|-----------------|
| 3.1 | [Async e await](./03-async-e-concorrenza/01-async-await.md) | Task, async, await, CancellationToken |
| 3.2 | [Task.WhenAll](./03-async-e-concorrenza/02-task-whenall.md) | Parallelismo, WhenAll, WhenAny |
| 3.3 | [Race Conditions](./03-async-e-concorrenza/03-race-conditions.md) | Concorrenza, validazione app vs DB constraint |

## Sezione 4 — Ecosistema .NET
> Le librerie e i pattern standard che trovi in ogni progetto .NET.

| # | Modulo | Concetti chiave |
|---|--------|-----------------|
| 4.1 | [Dependency Injection](./04-ecosistema-dotnet/01-dependency-injection.md) | DI, costruttori, interfacce come dipendenze |
| 4.2 | [AutoMapper](./04-ecosistema-dotnet/02-automapper.md) | CreateMap, ForMember, profili di mapping |
| 4.3 | [EF Core](./04-ecosistema-dotnet/03-ef-core.md) | DbContext, Include, AsNoTracking, migrations |
| 4.4 | [Logging](./04-ecosistema-dotnet/04-logging.md) | ILogger, livelli di log, structured logging |
| 4.5 | [Configuration](./04-ecosistema-dotnet/05-configuration.md) | appsettings.json, IOptions\<T\> |
| 4.6 | [Unique Index e Constraint](./04-ecosistema-dotnet/06-unique-index-e-constraint.md) | HasIndex, IsUnique, filtered index, DbUpdateException |

## Sezione 5 — Architettura
> Come è strutturato un progetto .NET professionale.

| # | Modulo | Concetti chiave |
|---|--------|-----------------|
| 5.1 | [Architettura DDD](./05-architettura/01-architettura-ddd.md) | Layer, Domain, Application, Infrastructure |
| 5.2 | [Pattern Proxy cross-modulo](./05-architettura/02-pattern-proxy.md) | Isolamento moduli, proxy, anti-corruption layer |
| 5.3 | [Repository Pattern](./05-architettura/03-repository-pattern.md) | Interfacce repository, astrazione dal DB |
| 5.4 | [Domain Events](./05-architettura/04-domain-events.md) | Eventi di dominio, event-driven architecture |
| 5.5 | [Unit of Work](./05-architettura/05-unit-of-work.md) | Atomicità, transazioni, commit, rollback |
| 5.6 | [Bulk Operations](./05-architettura/06-bulk-operations.md) | Partial update vs replace-all, validazione payload, atomicità |
