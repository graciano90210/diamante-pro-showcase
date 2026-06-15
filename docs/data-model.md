# Data Model

## Entity Hierarchy

Diamante Pro's data model follows a strict ownership chain. Every entity belongs to a company (`Empresa`), and `empresa_id` is present on every table to enforce multi-tenant isolation at the query layer.

```
Empresa
└── Sociedad (legal entity / holding)
    └── Oficina (branch office)
        └── Ruta (collection route)
            ├── Cobrador (field collector assigned to route)
            └── Cliente (borrower)
                └── Prestamo (loan)
                    └── Pago (payment installment)
```

---

## Entity Relationship Diagram

```mermaid
erDiagram
    Empresa {
        int id PK
        string nombre
        string estado_cuenta
        datetime trial_start
        string plan_tipo
    }

    Sociedad {
        int id PK
        int empresa_id FK
        string nombre
        string nit
    }

    Oficina {
        int id PK
        int empresa_id FK
        int sociedad_id FK
        string nombre
        string ciudad
    }

    Ruta {
        int id PK
        int empresa_id FK
        int oficina_id FK
        string nombre
        string moneda
    }

    Usuario {
        int id PK
        int empresa_id FK
        string email
        string rol
        string nombre
    }

    Cliente {
        int id PK
        int empresa_id FK
        int ruta_id FK
        string nombre
        string cedula
        string telefono
        float lat
        float lng
    }

    Prestamo {
        int id PK
        int empresa_id FK
        int cliente_id FK
        int ruta_id FK
        float monto
        float tasa_interes
        int num_cuotas
        string frecuencia
        string estado
        datetime fecha_inicio
        float score_credito
    }

    Pago {
        int id PK
        int empresa_id FK
        int prestamo_id FK
        float monto
        datetime fecha
        string metodo
        float lat
        float lng
    }

    MpPago {
        int id PK
        int empresa_id FK
        string mp_payment_id
        string estado
        float monto
        datetime fecha
    }

    Empresa ||--o{ Sociedad : "has"
    Empresa ||--o{ Oficina : "has"
    Empresa ||--o{ Ruta : "has"
    Empresa ||--o{ Usuario : "has"
    Empresa ||--o{ Cliente : "has"
    Empresa ||--o{ Prestamo : "has"
    Empresa ||--o{ Pago : "has"
    Empresa ||--o{ MpPago : "bills"
    Sociedad ||--o{ Oficina : "contains"
    Oficina ||--o{ Ruta : "contains"
    Ruta ||--o{ Cliente : "serves"
    Cliente ||--o{ Prestamo : "receives"
    Prestamo ||--o{ Pago : "generates"
```

---

## Key Design Decisions

### empresa_id Denormalization

`empresa_id` is stored on every entity (not just `Empresa`). This is intentional:

- Avoids multi-table JOINs just to filter by tenant
- Allows adding indexes on `(empresa_id, <field>)` for fast tenant-scoped queries
- Makes accidental cross-tenant data leaks visible at the row level

### Route Dual-Strategy Query

The `get_rutas_empresa_query(empresa_id)` helper uses a dual-strategy to handle legacy routes created before the `Ruta.empresa_id` column existed:

```python
# Strategy 1: direct empresa_id match (modern rows)
# Strategy 2: via cobrador subquery (legacy rows without empresa_id)
# Result: UNION of both — no data loss during migration
```

### Loan State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDIENTE: Loan created
    PENDIENTE --> ACTIVO: Approved & disbursed
    ACTIVO --> AL_DIA: Payments on schedule
    AL_DIA --> EN_MORA: Overdue > 1 day
    EN_MORA --> AL_DIA: Payment received
    AL_DIA --> CANCELADO: Fully paid
    EN_MORA --> CANCELADO: Fully paid
```

### Overdue Calculation

The canonical overdue calculation uses `_cuotas_esperadas_a_fecha()`, which compares **cumulative totals** from loan start date — not individual installment dates. This correctly handles:

- Irregular payment schedules
- Partial payments
- Rescheduled loans

### Geo-referenced Payments

`Pago` stores `lat`/`lng` at the moment of recording. This enables:

- Fraud detection (payment recorded 500km from client address)
- Route optimization analysis
- Field collector activity verification

---

## Subscription Model (`Empresa.estado_cuenta`)

| State | Access | Duration |
|---|---|---|
| `TRIAL` | Full access | 15 days from registration |
| `ACTIVO` | Full access | While subscription is paid |
| `MOROSO` | Read-only + warning | Grace period (configurable) |
| `CANCELADO` | Blocked, billing page only | Until reactivation |
| `PENDIENTE_SETUP` | Legacy state | — |
