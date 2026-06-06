# 04 Event Lifecycle

This state diagram models the transition states an event undergoes from creation to completion, detailing which roles possess the authority to execute specific state transitions.

```mermaid
stateDiagram-v2
    [*] --> DRAFT : Club Admin / Core Member Creates
    DRAFT --> PENDING_APPROVAL : Club Admin Submits
    
    PENDING_APPROVAL --> PUBLISHED : Faculty Mentor / Admin Approves
    PENDING_APPROVAL --> DRAFT : Faculty Mentor / Admin Rejects
    
    PUBLISHED --> ARCHIVED : Event Completes / Admin Archives
    PUBLISHED --> DRAFT : Admin Cancels
    
    ARCHIVED --> [*]
```
