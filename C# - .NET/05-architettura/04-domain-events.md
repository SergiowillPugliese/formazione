# 5.4 — Domain Events

## Il problema che risolvono

Quando una `CustomizationRequest` viene approvata commercialmente,
cosa deve succedere?
- Inviare email di notifica
- Aggiornare statistiche
- Creare un log entry
- Notificare il designer

Se metti tutto questo nella logica di approvazione, il Domain diventa
un groviglio di responsabilità:

```csharp
// SBAGLIATO — il Domain conosce troppo
public void ApproveCommercially(long userId, DateTime approvedAt)
{
    // logica business
    Status = CustomizationRequestStatus.CommerciallyApproved;

    // ma poi...
    _emailService.SendApprovalEmail(this);    // dipendenza da email service
    _analyticsService.Track(this);            // dipendenza da analytics
    _notificationHub.Notify(userId, this);   // dipendenza da SignalR
}
```

---

## La soluzione: eventi di dominio

Il Domain **pubblica un evento** — "è successa questa cosa".
Altri componenti **ascoltano** quell'evento e reagiscono.
Il Domain non sa chi ascolta, né cosa fanno.

```csharp
// Il Domain pubblica l'evento
public void ApproveCommercially(long userId, DateTime approvedAt)
{
    Status = CustomizationRequestStatus.CommerciallyApproved;
    CommercialApprovalByUserId = userId;
    CommercialApprovalAt = approvedAt;

    // Pubblica l'evento — non sa chi lo gestirà
    AddLocalEvent(new CustomizationRequestCommerciallyApprovedDomainEvent(this, userId));
}
```

```csharp
// Un handler nell'Application layer gestisce l'evento
public class SendEmailOnCommercialApprovalHandler
    : ILocalEventHandler<CustomizationRequestCommerciallyApprovedDomainEvent>
{
    public async Task HandleEventAsync(
        CustomizationRequestCommerciallyApprovedDomainEvent eventData)
    {
        await _emailService.SendApprovalEmail(eventData.CustomizationRequest);
    }
}
```

---

## Analogia Angular: EventEmitter e Observables

In Angular conosci già questo pattern:

```typescript
// Componente figlio — pubblica un evento
@Component({ ... })
export class ApprovalButtonComponent {
    @Output() approved = new EventEmitter<ApprovalEvent>();

    onApprove() {
        this.approved.emit({ requestId: this.requestId, userId: this.userId });
        // Non sa chi ascolterà — è il padre a decidere
    }
}

// Componente padre — ascolta l'evento
@Component({
    template: `<approval-button (approved)="onApproved($event)"></approval-button>`
})
export class ParentComponent {
    onApproved(event: ApprovalEvent) {
        this.emailService.send(event);
        this.analytics.track(event);
    }
}
```

I Domain Events seguono lo stesso principio: chi pubblica non sa chi
ascolta, chi ascolta non sa chi pubblica.

---

## Come sono strutturati nel progetto

```
Kustom.Domain/
    DomainEvents/
        CustomizationRequestCommerciallyApprovedDomainEvent.cs
        CustomizationRequestGraphicallyApprovedDomainEvent.cs
        CustomizationRequestRejectedDomainEvent.cs
        ...
```

```csharp
// La classe dell'evento — solo dati, nessuna logica
public class CustomizationRequestCommerciallyApprovedDomainEvent
{
    public CustomizationRequest CustomizationRequest { get; }
    public long ApprovedByUserId { get; }
    public DateTime ApprovedAt { get; }

    public CustomizationRequestCommerciallyApprovedDomainEvent(
        CustomizationRequest cr,
        long approvedByUserId)
    {
        CustomizationRequest = cr;
        ApprovedByUserId = approvedByUserId;
        ApprovedAt = DateTime.UtcNow;
    }
}
```

---

## `AddLocalEvent` vs eventi distribuiti

Nel progetto usi `AddLocalEvent` — eventi **locali**, nello stesso
processo, nello stesso Unit of Work:

```csharp
AddLocalEvent(new CustomizationRequestCommerciallyApprovedDomainEvent(this, userId));
// ↑ viene processato quando il DbContext fa SaveChanges
```

Esistono anche **eventi distribuiti** (messaggi tra microservizi,
via RabbitMQ, Azure Service Bus...) — ma sono un argomento più avanzato.

---

## Vantaggi

1. **Separazione delle responsabilità**: il Domain approva, l'Application manda email
2. **Estendibilità**: vuoi loggare le approvazioni? Aggiungi un nuovo handler — non tocchi il Domain
3. **Testabilità**: puoi testare l'approvazione senza che vengano mandate email
4. **Open/Closed Principle**: aperto all'estensione (nuovi handler), chiuso alla modifica (Domain non cambia)

---

## Riepilogo

| Concetto | Ruolo |
|---|---|
| Domain Event class | Contiene i dati dell'evento |
| `AddLocalEvent(evento)` | Pubblica l'evento dall'entità |
| `ILocalEventHandler<T>` | Implementa la reazione all'evento |
| Handler | Vive nell'Application layer — sa usare i servizi |
| Domain | Pubblica e basta — non sa chi gestirà |
