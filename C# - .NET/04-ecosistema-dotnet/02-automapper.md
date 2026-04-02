# 11 — AutoMapper: mappare oggetti

## Il problema che risolve

Nel progetto hai oggetti che rappresentano la stessa informazione
in layer diversi:
- `AgencyDto` (lato Application Contracts — il "contratto" esposto al mondo)
- `AgencyInfoModel` (lato Domain — il modello interno del domulo Kustom)

Hanno le stesse proprietà, ma sono tipi diversi. Devi convertire uno nell'altro.

Senza AutoMapper dovresti farlo a mano:

```csharp
// SENZA AutoMapper — noioso e fragile
var model = new AgencyInfoModel
{
    Id = dto.Id,
    Name = dto.Name,
    Code = dto.Code,
    // ... 10 altre proprietà
};
// Se aggiungi una nuova property a AgencyDto e ti dimentichi qui: bug silenzioso
```

AutoMapper automatizza questa conversione.

---

## Come funziona (concetto)

AutoMapper è come una "ricetta di traduzione": gli dici una volta
"per convertire un `AgencyDto` in `AgencyInfoModel`, copia le proprietà
con lo stesso nome". Poi usa questa ricetta ogni volta che serve.

```typescript
// Analogia Angular — immagina un adapter service
@Injectable()
class AgencyAdapter {
    toModel(dto: AgencyDto): AgencyInfoModel {
        return { ...dto }; // spread operator — copia tutto
    }
}
```

In C# questa "ricetta" si chiama **profilo di mapping** (`Profile`).

---

## Il profilo di mapping

Nel progetto, in `CatalogDataProxyAutoMapperProfile.cs`:

```csharp
public class CatalogDataProxyAutoMapperProfile : Profile
{
    public CatalogDataProxyAutoMapperProfile()
    {
        // "Per convertire AgencyDto in AgencyInfoModel, usa questa regola"
        CreateMap<AgencyDto, AgencyInfoModel>();
    }
}
```

`CreateMap<TSource, TDestination>()` dice ad AutoMapper:
"quando qualcuno vuole convertire un `AgencyDto` in un `AgencyInfoModel`,
copia tutte le proprietà con lo stesso nome automaticamente."

`Profile` è la classe base ABP/AutoMapper — il costruttore è il posto
giusto dove definire le regole di mapping.

---

## Usare il mapping — `ObjectMapper`

Una volta definita la regola, la usi tramite `ObjectMapper`:

```csharp
// In CatalogDataProxy
public async Task<AgencyInfoModel> GetAgencyOfUserOrThrowAsync(long userId, CancellationToken ct)
{
    // Chiama il servizio — ritorna AgencyDto (tipo del layer Application)
    var dto = await _agenciesAppService.GetAgencyOfUserOrThrowAsync(userId);

    // Converte AgencyDto → AgencyInfoModel (tipo del layer Domain)
    return ObjectMapper.Map<AgencyDto, AgencyInfoModel>(dto);
}
```

`ObjectMapper.Map<TSource, TDestination>(source)` è la chiamata che
esegue la conversione usando la ricetta definita nel profilo.

---

## Mapping con personalizzazione — `ForMember`

A volte le property non si chiamano allo stesso modo, oppure
vuoi una logica di trasformazione custom:

```csharp
CreateMap<AgencyDto, AgencyInfoModel>()
    .ForMember(
        dest => dest.Code,          // property di destinazione
        opt => opt.MapFrom(          // come ricavarla
            src => src.AgencyCode   // da questa property della sorgente
        )
    );
```

`ForMember` è un override per una singola property.
Senza `ForMember`, AutoMapper copia solo le property con lo stesso nome.

### Nel progetto

```csharp
// Esempio di mapping con ForMember
CreateMap<AgencyStoreDto, AgencyStoreInfoModel>()
    .ForMember(
        dest => dest.Code,
        opt => opt.MapFrom(src => src.AgencyCode) // AgencyCode → Code
    );
```

---

## Perché non mappare a mano?

1. **Meno codice boilerplate**: con 10 proprietà, `CreateMap<A, B>()` è 1 riga.
   A mano sarebbero 10 righe.

2. **Manutenzione**: se aggiungi una property a entrambi i tipi con lo stesso nome,
   AutoMapper la copia automaticamente. A mano dovresti ricordarti di aggiungerla.

3. **Separazione dei layer garantita**: `AgencyInfoModel` (Domain) non dipende
   da `AgencyDto` (Application Contracts). Il mapping avviene nel layer Application,
   che può dipendere da entrambi.

---

## Dove vivono i profili nel progetto

```
modules/Kustom/src/Kustom.Application/
    DomainServices/
        CatalogData/
            CatalogDataProxyAutoMapperProfile.cs  ← qui
```

ABP trova automaticamente tutte le classi che estendono `Profile`
e le registra nel container AutoMapper. Non devi fare niente altro.

---

## Riepilogo

| Concetto | Cosa fa |
|---|---|
| `Profile` | Classe base — contenitore delle regole di mapping |
| `CreateMap<A, B>()` | "Copia le property con lo stesso nome da A a B" |
| `ForMember(dest, opt)` | Override per una singola property |
| `ObjectMapper.Map<A, B>(source)` | Esegue la conversione usando le regole |
