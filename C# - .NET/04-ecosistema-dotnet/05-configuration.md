# 4.5 — Configuration

## Analogia diretta con Angular

| Angular | .NET |
|---------|------|
| `environment.ts` | `appsettings.json` |
| `environment.prod.ts` | `appsettings.Production.json` |
| `environment.development.ts` | `appsettings.Development.json` |

Stessa idea: configurazioni diverse per ambiente, senza toccare il codice.

---

## `appsettings.json` — il file di configurazione

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=FourShopCatalog;..."
  },
  "App": {
    "SupportEmail": "support@4shop.com",
    "MaxExportRows": 5000
  },
  "Hangfire": {
    "DashboardEnabled": true
  }
}
```

`appsettings.Development.json` sovrascrive solo i valori che vuoi
cambiare in sviluppo — non devi duplicare tutto.

---

## Leggere configurazione — modo semplice: `IConfiguration`

```csharp
public class MyService
{
    private readonly IConfiguration _configuration;

    public MyService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void DoSomething()
    {
        var email = _configuration["App:SupportEmail"];
        // "App:SupportEmail" = percorso con : come separatore

        var maxRows = _configuration.GetValue<int>("App:MaxExportRows");
        // GetValue<T> con tipo esplicito
    }
}
```

---

## `IOptions<T>` — il modo professionale

Invece di leggere singole chiavi con stringhe magiche, definisci
una classe C# che rappresenta la sezione di configurazione:

```csharp
// 1. Definisci la classe di configurazione
public class AppSettings
{
    public string SupportEmail { get; set; }
    public int MaxExportRows { get; set; }
}

// 2. Registra in Program.cs o nel modulo ABP
services.Configure<AppSettings>(configuration.GetSection("App"));

// 3. Inietta e usa
public class ReportService
{
    private readonly AppSettings _settings;

    public ReportService(IOptions<AppSettings> options)
    {
        _settings = options.Value;
    }

    public void Export()
    {
        var email = _settings.SupportEmail;    // type-safe, no stringhe magiche
        var max = _settings.MaxExportRows;     // int, non stringa
    }
}
```

**Perché è meglio di `IConfiguration`:**
- **Type-safe**: `MaxExportRows` è `int`, non una stringa da convertire
- **IntelliSense**: il compilatore conosce le proprietà disponibili
- **Testabile**: puoi creare un `AppSettings` finto nei test

---

## Variabili d'ambiente

In produzione, le configurazioni sensibili (connection string, API key)
non vanno in `appsettings.json` — vanno nelle variabili d'ambiente.

```bash
# Docker / Azure App Service
App__SupportEmail=production@4shop.com
ConnectionStrings__Default=Server=prod-db;...
```

.NET legge automaticamente le variabili d'ambiente e le sovrappone
a `appsettings.json`. Il `__` (doppio underscore) sostituisce il `:`.

---

## Nel progetto

Nel progetto il file `appsettings.json` è in `Catalog.Host/`.
Le classi di settings sono in `Catalog.Application/` o `Catalog.Domain/`.

```csharp
// Esempio dal progetto
public class KustomSettings
{
    public string DefaultDesignerEmail { get; set; }
    public int ReportMaxRows { get; set; }
}
```

---

## Riepilogo

| Concetto | Uso |
|---|---|
| `appsettings.json` | Configurazione base |
| `appsettings.{Env}.json` | Override per ambiente |
| `IConfiguration["chiave:sotto"]` | Lettura diretta (non ideale) |
| `IOptions<T>` | Lettura type-safe (consigliato) |
| Variabili d'ambiente | Segreti in produzione, sovrascrivono `appsettings` |
