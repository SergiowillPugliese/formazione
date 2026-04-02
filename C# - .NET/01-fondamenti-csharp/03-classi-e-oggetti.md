# 03 — Classi e oggetti

## Il concetto base

In TypeScript hai probabilmente usato le classi nei servizi Angular:

```typescript
// TypeScript / Angular
@Injectable()
export class AgencyService {
    private http: HttpClient;

    constructor(http: HttpClient) {
        this.http = http;
    }

    getAgency(id: number): Observable<Agency> {
        return this.http.get<Agency>(`/api/agencies/${id}`);
    }
}
```

In C# le classi funzionano esattamente nello stesso modo, senza i decorator Angular:

```csharp
// C#
public class AgencyService
{
    private readonly HttpClient _http;

    public AgencyService(HttpClient http)
    {
        _http = http;
    }

    public async Task<Agency> GetAgencyAsync(long id)
    {
        return await _http.GetFromJsonAsync<Agency>($"/api/agencies/{id}");
    }
}
```

---

## Anatomia di una classe C#

```csharp
// Modificatore di accesso + parola chiave + nome
public class CustomizationRequestReportExcelExporter
{
    // Campi privati — per convenzione, prefisso _
    private readonly ICatalogDataProxy _catalogDataProxy;
    private readonly ILogger<CustomizationRequestReportExcelExporter> _logger;

    // Costruttore — stesso nome della classe, nessun tipo di ritorno
    public CustomizationRequestReportExcelExporter(
        ICatalogDataProxy catalogDataProxy,
        ILogger<CustomizationRequestReportExcelExporter> logger)
    {
        _catalogDataProxy = catalogDataProxy;
        _logger = logger;
    }

    // Metodo pubblico
    public async Task<byte[]> ExportAsync(ReportExportRequestModel request)
    {
        // logica...
    }

    // Metodo privato — solo usabile dentro questa classe
    private static CustomizationRequestReportRow BuildRow(...)
    {
        // logica...
    }
}
```

---

## Modificatori di accesso

| Modificatore | Significato | Analogo TypeScript |
|---|---|---|
| `public` | Visibile ovunque | `public` (default in TS) |
| `private` | Solo dentro questa classe | `private` |
| `protected` | Questa classe e le sottoclassi | `protected` |
| `internal` | Solo dentro lo stesso progetto/assembly | nessun diretto analogo |

In TypeScript il default è `public`. In C# il default è `private` per i campi,
quindi devi sempre scrivere `public` esplicitamente su ciò che vuoi esporre.

---

## `readonly` — campi immutabili

```csharp
private readonly ICatalogDataProxy _catalogDataProxy;
```

`readonly` significa che il campo può essere assegnato solo nel costruttore,
non successivamente. È come `const` in JavaScript ma per campi di classe.

```typescript
// TypeScript equivalente
private readonly catalogDataProxy: ICatalogDataProxy;
```

---

## Property vs campo

In C# si distingue tra **campo** (variabile diretta) e **property** (con getter/setter):

```csharp
// Campo — variabile diretta
private string _nome;

// Property — con getter e setter espliciti
public string Nome
{
    get { return _nome; }
    set { _nome = value; }
}

// Property automatica — shorthand, il compilatore genera il campo nascosto
public string Nome { get; set; }

// Property read-only — solo getter
public string Nome { get; }

// Property init-only — assegnabile solo alla costruzione
public string Nome { get; init; }
```

In TypeScript non esiste questa distinzione formale — le property di classe
sono direttamente campi. In C# la convenzione è usare property (con `{ get; set; }`)
invece di campi pubblici per poter controllare l'accesso.

---

## `static` — metodi senza istanza

Un metodo `static` appartiene alla classe, non a una istanza specifica.
Non può accedere a `this` (o in C#, ai campi dell'istanza).

```csharp
// Metodo statico — non accede a _catalogDataProxy, _logger, ecc.
private static CustomizationRequestReportRow BuildRow(
    CustomizationRequest cr,
    Dictionary<long, string> usersLookup,
    ...)
{
    // riceve tutto ciò di cui ha bisogno come parametri
    return new CustomizationRequestReportRow { ... };
}
```

Lo usiamo in `BuildRow` perché non ha bisogno di nulla dell'istanza —
riceve tutti i dati come parametri. Il compilatore ci dice se stiamo
erroneamente accedendo a qualcosa dell'istanza in un metodo `static`.

---

## Istanziare un oggetto

```csharp
// Creare una nuova istanza
var row = new CustomizationRequestReportRow {
    Id = 1,
    Title = "Test",
    Status = "Draft"
};

// Equivalente TypeScript
const row: CustomizationRequestReportRow = {
    id: 1,
    title: "Test",
    status: "Draft"
};
```

La sintassi `new NomeClasse { Prop1 = val1, Prop2 = val2 }` si chiama
**object initializer** — puoi assegnare le property direttamente alla creazione.

---

## `var` — inferenza di tipo

In C# puoi usare `var` e il compilatore deduce il tipo:

```csharp
var nome = "Mario";              // deduce string
var id = 42L;                    // deduce long (la L alla fine indica long)
var lista = new List<string>();  // deduce List<string>
```

È come `const` / `let` in TypeScript con inferenza automatica.
Il tipo rimane fisso dopo la prima assegnazione — non è come `any` in TypeScript.
