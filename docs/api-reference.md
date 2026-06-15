# API Reference

Diamante Pro exposes two API surfaces:

- **API v1** (`/api/`) — Legacy REST consumed by the Flutter mobile app. Frozen; no new endpoints.
- **API v2** (`/api/v2/`) — OpenAPI 3.0 with Pydantic schemas. All new integrations use this.

Both require JWT authentication. All responses are JSON.

---

## Authentication

### POST `/auth/login`

Authenticates a user and returns a JWT access token.

**Request**
```json
{
  "email": "admin@empresa.com",
  "password": "secret"
}
```

**Response `200 OK`**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "nombre": "Juan Graciano",
    "rol": "ADMIN",
    "empresa_id": 42,
    "empresa_nombre": "Inversiones del Norte"
  }
}
```

**JWT Payload**
```json
{
  "sub": "1",
  "empresa_id": 42,
  "rol": "ADMIN",
  "exp": 1718000000
}
```

The `empresa_id` embedded in the token is the tenant key — every subsequent query is automatically scoped to this company.

---

## API v2 Endpoints

### Clients

#### GET `/api/v2/clientes`

Returns paginated list of clients for the authenticated company.

**Headers**
```
Authorization: Bearer <token>
```

**Query Parameters**
| Param | Type | Description |
|---|---|---|
| `ruta_id` | int | Filter by route |
| `estado` | string | `activo`, `inactivo` |
| `page` | int | Page number (default: 1) |
| `per_page` | int | Results per page (default: 20, max: 100) |

**Response `200 OK`**
```json
{
  "items": [
    {
      "id": 101,
      "nombre": "Maria Lopez",
      "cedula": "1234567890",
      "telefono": "+573001234567",
      "ruta_id": 5,
      "ruta_nombre": "Ruta Norte",
      "prestamos_activos": 1,
      "saldo_pendiente": 1250000.00
    }
  ],
  "total": 847,
  "page": 1,
  "pages": 43
}
```

---

### Loans

#### GET `/api/v2/prestamos`

Returns loan portfolio for the authenticated company.

**Query Parameters**
| Param | Type | Description |
|---|---|---|
| `estado` | string | `ACTIVO`, `EN_MORA`, `CANCELADO` |
| `ruta_id` | int | Filter by route |
| `mora_min` | int | Minimum days overdue |
| `fecha_desde` | date | Disbursement date from |

**Response `200 OK`**
```json
{
  "items": [
    {
      "id": 2301,
      "cliente_nombre": "Maria Lopez",
      "monto": 5000000.00,
      "saldo_pendiente": 1250000.00,
      "cuotas_pagadas": 8,
      "cuotas_totales": 12,
      "dias_mora": 0,
      "estado": "ACTIVO",
      "score_credito": 0.82,
      "proxima_cuota": {
        "fecha": "2026-06-20",
        "monto": 416667.00
      }
    }
  ],
  "resumen": {
    "cartera_total": 125000000.00,
    "en_mora": 8500000.00,
    "tasa_mora": 0.068
  }
}
```

#### POST `/api/v2/prestamos`

Creates a new loan.

**Request**
```json
{
  "cliente_id": 101,
  "ruta_id": 5,
  "monto": 5000000.00,
  "tasa_interes": 5.0,
  "num_cuotas": 12,
  "frecuencia": "semanal",
  "fecha_inicio": "2026-06-15"
}
```

**Response `201 Created`**
```json
{
  "id": 2305,
  "estado": "PENDIENTE",
  "tabla_amortizacion": [
    { "cuota": 1, "fecha": "2026-06-22", "capital": 396825, "interes": 250000, "total": 646825 },
    { "cuota": 2, "fecha": "2026-06-29", "capital": 416666, "interes": 230159, "total": 646825 }
  ]
}
```

---

### Payments

#### POST `/api/v2/pagos`

Records a payment. Accepts geo-coordinates for field validation.

**Request**
```json
{
  "prestamo_id": 2301,
  "monto": 416667.00,
  "metodo": "efectivo",
  "lat": 4.7110,
  "lng": -74.0721,
  "foto_comprobante": "base64_encoded_image"
}
```

**Response `201 Created`**
```json
{
  "id": 8821,
  "prestamo_id": 2301,
  "monto": 416667.00,
  "fecha": "2026-06-15T09:32:00Z",
  "saldo_restante": 833333.00,
  "recibo_url": "/cobro/recibo/8821"
}
```

---

### Reports

#### GET `/api/v2/reportes/cuadre-diario`

Returns daily cash reconciliation data for a route.

**Query Parameters**
| Param | Type | Description |
|---|---|---|
| `ruta_id` | int | Required |
| `fecha` | date | Default: today |

**Response `200 OK`**
```json
{
  "ruta": "Ruta Norte",
  "fecha": "2026-06-15",
  "cobrador": "Carlos Perez",
  "resumen": {
    "clientes_visitados": 32,
    "pagos_recibidos": 28,
    "total_recaudado": 11680000.00,
    "pendiente_cobro": 1640000.00,
    "tasa_efectividad": 0.875
  },
  "detalle": [...]
}
```

---

## Error Responses

All errors follow a consistent envelope:

```json
{
  "error": "LOAN_NOT_FOUND",
  "message": "Loan 9999 does not exist or belongs to another company",
  "status": 404
}
```

| Code | Meaning |
|---|---|
| `401` | Missing or expired JWT |
| `403` | Insufficient role permissions |
| `404` | Resource not found (or cross-tenant access attempt) |
| `422` | Validation error (Pydantic schema) |
| `429` | Rate limit exceeded |

---

## Role-Based Access

| Role | Access Level |
|---|---|
| `SUPER_ADMIN` | All companies, all data |
| `ADMIN` | Full access within their company |
| `GERENTE` | Read access + report generation |
| `SUPERVISOR` | Routes under their supervision only |
| `COBRADOR` | Only their assigned route |
