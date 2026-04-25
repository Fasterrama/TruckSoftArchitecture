# TruckSoft TMS — Observability Strategy

**Version:** 1.0
**Date:** 2026-04-18
**Author:** Engineering Team — Minvy
**Stack:** Angular 15 · Spring Boot 3.3.1 · MySQL 8.0
**Analytics Platform:** Mixpanel

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Mixpanel Setup](#2-mixpanel-setup)
3. [Event Taxonomy](#3-event-taxonomy)
4. [User Properties](#4-user-properties)
5. [Key Metrics & Dashboards](#5-key-metrics--dashboards)
6. [Implementation Plan](#6-implementation-plan)
7. [Implementation Code Examples](#7-implementation-code-examples)
8. [Privacy & Compliance](#8-privacy--compliance)
9. [Alerting](#9-alerting)
10. [Cost Estimate](#10-cost-estimate)

---

## 1. Executive Summary

TruckSoft TMS is a mission-critical system that manages the full lifecycle of freight
operations in Mexico — from quoting (cotizacion) through invoicing (timbrado CFDI) and
payment collection (cobro). The system currently spans **139 controllers**, **595 API
endpoints**, **228 Angular components**, and **3 user portals** (main, customer,
operator). Despite this scale, we have near-zero visibility into what happens at runtime.

### Why Observability Matters Now

**1. CFDI Compliance Visibility**
Every invoice must be stamped (timbrado) through a PAC provider (Facturapi). A failed
timbrado means revenue cannot be legally recognized. Today we discover failures only when
users report them. With observability, we detect failures in minutes, understand root
causes (malformed RFC, missing addenda, PAC downtime), and track compliance rates.

**2. Revenue Tracking**
The pipeline cotizacion → remision → manifiesto → prefactura → timbrado → cobro is
TruckSoft's revenue engine. We need to measure conversion at each stage, identify where
shipments stall, and quantify the average time from quote to cash collection.

**3. Operational Efficiency**
With 595 endpoints, we do not know which features are used, which are dead code, or where
users encounter friction. Observability gives us data to prioritize engineering effort.

**4. User Adoption**
Three portals serve different user groups. We need to understand adoption rates, session
patterns, and feature usage to guide product decisions.

### Expected Outcomes

| Metric | Before | Target (90 days) |
|---|---|---|
| Timbrado failure detection time | Hours/days | < 5 minutes |
| Revenue pipeline visibility | None | Full funnel metrics |
| Feature usage data | Anecdotal | Quantified per module |
| User adoption tracking | None | DAU/WAU/MAU per portal |

---

## 2. Mixpanel Setup

### 2.1 Project Structure

Minvy already has two Mixpanel projects:

| Project | ID | Purpose |
|---|---|---|
| Minvy_QA | 2984481 | Testing / staging |
| Minvy_Production | 2987343 | Production data |

We will use these existing projects. All TruckSoft events will be prefixed or tagged to
distinguish them from other Minvy products if the projects are shared. If isolation is
preferred, create a dedicated project named **TruckSoft** under the same Mixpanel
organization.

### 2.2 Frontend SDK Installation (Angular)

```bash
npm install mixpanel-browser@^2.49.0
npm install --save-dev @types/mixpanel-browser
```

**Environment configuration:**

```typescript
// src/environments/environment.ts (Development / QA)
export const environment = {
  production: false,
  mixpanel: {
    token: '<MINVY_QA_TOKEN>',        // Project ID: 2984481
    projectId: 2984481,
    debug: true,
    apiHost: 'https://api.mixpanel.com',
  },
};

// src/environments/environment.prod.ts (Production)
export const environment = {
  production: true,
  mixpanel: {
    token: '<MINVY_PRODUCTION_TOKEN>', // Project ID: 2987343
    projectId: 2987343,
    debug: false,
    apiHost: 'https://api.mixpanel.com',
  },
};
```

### 2.3 Backend SDK Installation (Spring Boot)

Add to `pom.xml`:

```xml
<dependency>
    <groupId>com.mixpanel</groupId>
    <artifactId>mixpanel-java</artifactId>
    <version>1.5.2</version>
</dependency>
```

**Application properties:**

```yaml
# application.yml
mixpanel:
  token: ${MIXPANEL_TOKEN}
  api-host: https://api.mixpanel.com
  enabled: true

# application-dev.yml
mixpanel:
  token: ${MIXPANEL_QA_TOKEN}       # Project 2984481
  enabled: true

# application-prod.yml
mixpanel:
  token: ${MIXPANEL_PROD_TOKEN}     # Project 2987343
  enabled: true
```

### 2.4 Environment Separation

| Environment | Mixpanel Project | Token Source |
|---|---|---|
| Local dev | Minvy_QA (2984481) | `.env.local` |
| QA / Staging | Minvy_QA (2984481) | CI/CD secrets |
| Production | Minvy_Production (2987343) | CI/CD secrets |

Never hardcode tokens. Use environment variables injected at build/deploy time.

### 2.5 User Identification Strategy

```
Mixpanel distinct_id = tbl_usuarios.id (integer, primary key)
```

On login, call `mixpanel.identify(userId)` on the frontend and set the same `distinct_id`
on backend events. This links frontend and backend activity to the same user.

For anonymous visitors (customer portal before login), Mixpanel auto-generates a
`distinct_id`. On login, call `mixpanel.alias(anonymousId, userId)` once to merge.

---

## 3. Event Taxonomy

This is the core of the observability strategy. Every event follows these conventions:

- **Naming:** `snake_case`, English, verb-first or noun-first depending on clarity.
- **Properties:** typed, documented, with example values.
- **Source:** either `backend` (tracked in Java controller/service) or `frontend`
  (tracked in Angular component/service).

### Tier 1: Revenue & Compliance (Phase 1 — Weeks 1-2)

These events directly impact revenue recognition and tax compliance. Implement first.

---

#### `cfdi_timbrado_attempted`

Fires when the system initiates a timbrado request to the PAC (Facturapi).

| Property | Type | Description | Example |
|---|---|---|---|
| `prefactura_id` | integer | ID of the prefactura being stamped | `45231` |
| `remision_ids` | integer[] | Remisiones included in this invoice | `[1023, 1024]` |
| `tipo_comprobante` | string | CFDI type (I=Ingreso, E=Egreso, P=Pago) | `"I"` |
| `receptor_rfc` | string | Recipient RFC (masked) | `"XAX***010101***"` |
| `receptor_nombre` | string | Recipient business name | `"Transportes del Norte"` |
| `subtotal` | number | Subtotal amount MXN | `15000.00` |
| `total` | number | Total amount MXN with tax | `17400.00` |
| `moneda` | string | Currency code | `"MXN"` |
| `uso_cfdi` | string | CFDI usage code | `"G03"` |
| `metodo_pago` | string | Payment method | `"PPD"` |
| `regimen_fiscal` | string | Tax regime | `"601"` |
| `user_id` | integer | User who initiated | `42` |
| `locacion_id` | integer | Branch/location | `3` |
| `timestamp` | datetime | ISO 8601 | `"2026-04-18T14:30:00Z"` |

**Source:** Backend — `PrefacturasController.timbrar()` or equivalent service method.

**Example payload:**
```json
{
  "event": "cfdi_timbrado_attempted",
  "properties": {
    "prefactura_id": 45231,
    "remision_ids": [1023, 1024],
    "tipo_comprobante": "I",
    "receptor_rfc": "XAX***010101***",
    "receptor_nombre": "Transportes del Norte",
    "subtotal": 15000.00,
    "total": 17400.00,
    "moneda": "MXN",
    "uso_cfdi": "G03",
    "metodo_pago": "PPD",
    "regimen_fiscal": "601",
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `cfdi_timbrado_success`

Fires when the PAC returns a successful timbrado response.

| Property | Type | Description | Example |
|---|---|---|---|
| `prefactura_id` | integer | Prefactura ID | `45231` |
| `uuid_fiscal` | string | UUID assigned by SAT | `"A1B2C3D4-..."` |
| `folio_fiscal` | string | Fiscal folio | `"F-2026-0451"` |
| `serie` | string | Invoice series | `"A"` |
| `total` | number | Invoice total MXN | `17400.00` |
| `pac_provider` | string | PAC used | `"facturapi"` |
| `response_time_ms` | integer | PAC response time | `2340` |
| `tipo_comprobante` | string | CFDI type | `"I"` |
| `user_id` | integer | User who initiated | `42` |
| `locacion_id` | integer | Branch/location | `3` |

**Source:** Backend — `PrefacturasController.timbrar()` success path.

**Example payload:**
```json
{
  "event": "cfdi_timbrado_success",
  "properties": {
    "prefactura_id": 45231,
    "uuid_fiscal": "A1B2C3D4-E5F6-7890-ABCD-EF1234567890",
    "folio_fiscal": "F-2026-0451",
    "serie": "A",
    "total": 17400.00,
    "pac_provider": "facturapi",
    "response_time_ms": 2340,
    "tipo_comprobante": "I",
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `cfdi_timbrado_failed`

Fires when timbrado fails for any reason.

| Property | Type | Description | Example |
|---|---|---|---|
| `prefactura_id` | integer | Prefactura ID | `45231` |
| `error_code` | string | PAC/Facturapi error code | `"tax_id_invalid"` |
| `error_message` | string | Human-readable error | `"RFC del receptor no valido"` |
| `error_source` | string | Where the error originated | `"facturapi"` / `"validation"` |
| `total` | number | Attempted invoice total | `17400.00` |
| `receptor_rfc` | string | Masked RFC | `"XAX***010101***"` |
| `tipo_comprobante` | string | CFDI type | `"I"` |
| `retry_count` | integer | Number of retries so far | `0` |
| `user_id` | integer | User who initiated | `42` |
| `locacion_id` | integer | Branch/location | `3` |

**Source:** Backend — `PrefacturasController.timbrar()` catch block.

**Example payload:**
```json
{
  "event": "cfdi_timbrado_failed",
  "properties": {
    "prefactura_id": 45231,
    "error_code": "tax_id_invalid",
    "error_message": "RFC del receptor no valido ante el SAT",
    "error_source": "facturapi",
    "total": 17400.00,
    "receptor_rfc": "XAX***010101***",
    "tipo_comprobante": "I",
    "retry_count": 0,
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `cfdi_cancelled`

Fires when a previously stamped CFDI is cancelled.

| Property | Type | Description | Example |
|---|---|---|---|
| `prefactura_id` | integer | Prefactura ID | `45231` |
| `uuid_fiscal` | string | UUID of the cancelled CFDI | `"A1B2C3D4-..."` |
| `motivo_cancelacion` | string | SAT cancellation reason code | `"02"` |
| `total` | number | Invoice total that was cancelled | `17400.00` |
| `days_since_timbrado` | integer | Days between stamp and cancel | `5` |
| `user_id` | integer | User who cancelled | `42` |
| `locacion_id` | integer | Branch/location | `3` |

**Source:** Backend — Cancellation controller/service.

**Example payload:**
```json
{
  "event": "cfdi_cancelled",
  "properties": {
    "prefactura_id": 45231,
    "uuid_fiscal": "A1B2C3D4-E5F6-7890-ABCD-EF1234567890",
    "motivo_cancelacion": "02",
    "total": 17400.00,
    "days_since_timbrado": 5,
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `payment_received`

Fires when a payment (cobro) is recorded in the system.

| Property | Type | Description | Example |
|---|---|---|---|
| `cobro_id` | integer | Payment record ID | `8923` |
| `prefactura_id` | integer | Invoice being paid | `45231` |
| `monto` | number | Amount received | `17400.00` |
| `moneda` | string | Currency | `"MXN"` |
| `metodo_pago` | string | Transfer, check, cash, etc. | `"transferencia"` |
| `banco` | string | Bank name | `"BBVA"` |
| `days_since_timbrado` | integer | Days from invoice to payment | `18` |
| `is_partial` | boolean | Partial payment flag | `false` |
| `remaining_balance` | number | Outstanding balance after payment | `0.00` |
| `client_id` | integer | Client ID | `312` |
| `user_id` | integer | User recording payment | `42` |
| `locacion_id` | integer | Branch/location | `3` |

**Source:** Backend — Cobros controller.

**Example payload:**
```json
{
  "event": "payment_received",
  "properties": {
    "cobro_id": 8923,
    "prefactura_id": 45231,
    "monto": 17400.00,
    "moneda": "MXN",
    "metodo_pago": "transferencia",
    "banco": "BBVA",
    "days_since_timbrado": 18,
    "is_partial": false,
    "remaining_balance": 0.00,
    "client_id": 312,
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `payment_complement_created`

Fires when a Complemento de Pago (REP) CFDI is created.

| Property | Type | Description | Example |
|---|---|---|---|
| `complemento_id` | integer | Complement record ID | `1120` |
| `prefactura_ids` | integer[] | Related invoices | `[45231, 45232]` |
| `monto_total` | number | Total payment amount | `34800.00` |
| `num_parcialidades` | integer | Payment installment number | `1` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Payment complement controller.

---

#### `credit_note_created`

Fires when a nota de credito (Egreso CFDI) is issued.

| Property | Type | Description | Example |
|---|---|---|---|
| `nota_credito_id` | integer | Credit note ID | `503` |
| `prefactura_original_id` | integer | Original invoice being credited | `45231` |
| `monto` | number | Credit amount | `5000.00` |
| `motivo` | string | Reason for credit | `"devolucion"` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Credit note controller.

---

#### `remision_created`

Fires when a new remision (shipment order) is created.

| Property | Type | Description | Example |
|---|---|---|---|
| `remision_id` | integer | Remision ID | `10234` |
| `client_id` | integer | Client ID | `312` |
| `origen` | string | Origin city/state | `"Monterrey, NL"` |
| `destino` | string | Destination city/state | `"CDMX, CDMX"` |
| `tipo_servicio` | string | Service type | `"FTL"` |
| `tipo_unidad` | string | Vehicle type | `"caja_seca_53"` |
| `tarifa` | number | Agreed rate | `25000.00` |
| `moneda` | string | Currency | `"MXN"` |
| `peso_kg` | number | Cargo weight | `18000` |
| `num_bultos` | integer | Number of pieces | `24` |
| `fecha_recoleccion` | string | Pickup date | `"2026-04-20"` |
| `source` | string | How it was created | `"manual"` / `"cotizacion"` / `"bulk_upload"` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Remisiones controller.

**Example payload:**
```json
{
  "event": "remision_created",
  "properties": {
    "remision_id": 10234,
    "client_id": 312,
    "origen": "Monterrey, NL",
    "destino": "CDMX, CDMX",
    "tipo_servicio": "FTL",
    "tipo_unidad": "caja_seca_53",
    "tarifa": 25000.00,
    "moneda": "MXN",
    "peso_kg": 18000,
    "num_bultos": 24,
    "fecha_recoleccion": "2026-04-20",
    "source": "manual",
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `remision_status_changed`

Fires on every status transition of a remision.

| Property | Type | Description | Example |
|---|---|---|---|
| `remision_id` | integer | Remision ID | `10234` |
| `from_status` | string | Previous status | `"documentada"` |
| `to_status` | string | New status | `"en_transito"` |
| `time_in_previous_status_hours` | number | Hours spent in previous status | `4.5` |
| `client_id` | integer | Client ID | `312` |
| `user_id` | integer | User who changed status | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Remisiones status update endpoint.

**Example payload:**
```json
{
  "event": "remision_status_changed",
  "properties": {
    "remision_id": 10234,
    "from_status": "documentada",
    "to_status": "en_transito",
    "time_in_previous_status_hours": 4.5,
    "client_id": 312,
    "user_id": 42,
    "locacion_id": 3
  }
}
```

---

#### `cotizacion_created`

Fires when a new quote is created.

| Property | Type | Description | Example |
|---|---|---|---|
| `cotizacion_id` | integer | Quote ID | `7891` |
| `client_id` | integer | Client ID | `312` |
| `origen` | string | Origin | `"Guadalajara, JAL"` |
| `destino` | string | Destination | `"Queretaro, QRO"` |
| `tarifa_propuesta` | number | Quoted rate | `18500.00` |
| `moneda` | string | Currency | `"MXN"` |
| `tipo_servicio` | string | Service type | `"FTL"` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Cotizaciones controller.

---

#### `cotizacion_accepted`

Fires when a quote is converted into a remision (accepted).

| Property | Type | Description | Example |
|---|---|---|---|
| `cotizacion_id` | integer | Quote ID | `7891` |
| `remision_id` | integer | Resulting remision ID | `10234` |
| `tarifa_original` | number | Originally quoted rate | `18500.00` |
| `tarifa_final` | number | Final agreed rate | `18000.00` |
| `descuento_pct` | number | Discount percentage | `2.7` |
| `days_to_accept` | integer | Days from quote to acceptance | `3` |
| `client_id` | integer | Client ID | `312` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Cotizaciones acceptance endpoint.

---

### Tier 2: Operations (Phase 2 — Weeks 3-4)

---

#### `manifiesto_created`

Fires when a manifest (grouping of remisiones for a trip) is created.

| Property | Type | Description | Example |
|---|---|---|---|
| `manifiesto_id` | integer | Manifest ID | `5501` |
| `num_remisiones` | integer | Count of remisiones in manifest | `4` |
| `remision_ids` | integer[] | Remision IDs | `[10234, 10235, 10236, 10237]` |
| `operador_id` | integer | Assigned driver ID | `78` |
| `unidad_id` | integer | Assigned vehicle ID | `15` |
| `ruta` | string | Route description | `"MTY-GDL-CDMX"` |
| `km_estimados` | number | Estimated kilometers | `1050` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Manifiestos controller.

---

#### `carta_porte_generated`

Fires when a Carta Porte complement is generated for a CFDI.

| Property | Type | Description | Example |
|---|---|---|---|
| `carta_porte_id` | integer | Record ID | `3310` |
| `manifiesto_id` | integer | Related manifest | `5501` |
| `version_carta_porte` | string | CCP version | `"3.1"` |
| `tipo_transporte` | string | Transport type | `"01"` (autotransporte) |
| `num_mercancias` | integer | Number of goods entries | `3` |
| `peso_total_kg` | number | Total weight | `22000` |
| `distancia_recorrida_km` | number | Distance | `1050` |
| `num_ubicaciones` | integer | Number of stops | `4` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Carta Porte generation service.

---

#### `bulk_upload_completed`

Fires when a bulk import (Excel or file upload) completes.

| Property | Type | Description | Example |
|---|---|---|---|
| `upload_id` | string | Upload batch identifier | `"batch_20260418_001"` |
| `entity_type` | string | What was uploaded | `"remisiones"` / `"clientes"` / `"tarifas"` |
| `file_format` | string | File type | `"xlsx"` / `"csv"` |
| `total_rows` | integer | Total rows in file | `150` |
| `success_rows` | integer | Rows imported successfully | `142` |
| `error_rows` | integer | Rows that failed | `8` |
| `error_summary` | string | Top error reasons | `"RFC invalido (5), campo vacio (3)"` |
| `processing_time_ms` | integer | Total processing time | `4500` |
| `user_id` | integer | User | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Bulk upload controller.

---

#### `pdf_exported`

Fires when a user exports/downloads a PDF document.

| Property | Type | Description | Example |
|---|---|---|---|
| `document_type` | string | Type of document | `"factura"` / `"carta_porte"` / `"remision"` |
| `entity_id` | integer | Related record ID | `45231` |
| `format` | string | Output format | `"pdf"` / `"xml"` |
| `file_size_kb` | integer | File size | `245` |
| `user_id` | integer | User | `42` |

**Source:** Backend — PDF generation endpoints.

---

#### `exchange_rate_fetched`

Fires when the Banxico API is called for exchange rates.

| Property | Type | Description | Example |
|---|---|---|---|
| `currency_pair` | string | Currency pair | `"USD/MXN"` |
| `rate` | number | Exchange rate returned | `17.45` |
| `source` | string | Data source | `"banxico"` |
| `response_time_ms` | integer | API response time | `890` |
| `success` | boolean | Whether the call succeeded | `true` |
| `fallback_used` | boolean | Whether a cached/fallback rate was used | `false` |
| `date_requested` | string | Rate date requested | `"2026-04-18"` |

**Source:** Backend — Banxico integration service.

---

### Tier 3: User Behavior (Phase 3 — Weeks 5-6)

---

#### `page_viewed`

Fires on every route navigation in Angular.

| Property | Type | Description | Example |
|---|---|---|---|
| `page_path` | string | Angular route path | `"/remisiones/lista"` |
| `page_title` | string | Page title | `"Lista de Remisiones"` |
| `portal` | string | Which portal | `"main"` / `"customer"` / `"operator"` |
| `referrer_path` | string | Previous page | `"/dashboard"` |
| `load_time_ms` | integer | Page load time | `1200` |
| `user_id` | integer | User | `42` |

**Source:** Frontend — Angular Router events listener.

---

#### `search_performed`

Fires when a user executes a search or applies filters.

| Property | Type | Description | Example |
|---|---|---|---|
| `module` | string | Which module | `"remisiones"` / `"clientes"` / `"facturas"` |
| `search_term` | string | Search text (hashed if PII-risk) | `"transportes"` |
| `filters_applied` | object | Active filters | `{"status": "en_transito", "fecha_desde": "2026-04-01"}` |
| `results_count` | integer | Number of results | `23` |
| `response_time_ms` | integer | Search response time | `340` |
| `user_id` | integer | User | `42` |

**Source:** Frontend — Search/filter components.

---

#### `login_success`

Fires on successful authentication.

| Property | Type | Description | Example |
|---|---|---|---|
| `user_id` | integer | User ID | `42` |
| `portal` | string | Portal used | `"main"` |
| `auth_method` | string | Authentication method | `"password"` / `"sso"` |
| `ip_country` | string | GeoIP country | `"MX"` |
| `device_type` | string | Device category | `"desktop"` / `"mobile"` |
| `browser` | string | Browser name | `"Chrome 120"` |

**Source:** Frontend — Login component, after successful API response.

---

#### `login_failed`

Fires on failed authentication attempt.

| Property | Type | Description | Example |
|---|---|---|---|
| `attempted_username` | string | Hashed username | `"a1b2c3..."` |
| `failure_reason` | string | Why it failed | `"invalid_password"` / `"user_not_found"` / `"account_locked"` |
| `portal` | string | Portal | `"main"` |
| `ip_country` | string | GeoIP country | `"MX"` |

**Source:** Backend — Authentication controller.

---

#### `client_created`

Fires when a new client record is created.

| Property | Type | Description | Example |
|---|---|---|---|
| `client_id` | integer | New client ID | `313` |
| `tipo_persona` | string | Moral or fisica | `"moral"` |
| `regimen_fiscal` | string | Tax regime code | `"601"` |
| `has_csf` | boolean | Whether CSF was uploaded | `true` |
| `has_valid_rfc` | boolean | Whether RFC passed validation | `true` |
| `user_id` | integer | User who created | `42` |
| `locacion_id` | integer | Branch | `3` |

**Source:** Backend — Clientes controller.

---

#### `client_updated`

Fires when a client record is modified.

| Property | Type | Description | Example |
|---|---|---|---|
| `client_id` | integer | Client ID | `312` |
| `fields_changed` | string[] | Which fields were updated | `["rfc", "regimen_fiscal"]` |
| `user_id` | integer | User | `42` |

**Source:** Backend — Clientes update endpoint.

---

#### `portal_accessed`

Fires when a customer or operator accesses their portal.

| Property | Type | Description | Example |
|---|---|---|---|
| `portal` | string | Which portal | `"customer"` / `"operator"` |
| `entity_id` | integer | Client or operator ID | `312` |
| `action` | string | What they did | `"view_shipments"` / `"download_invoice"` / `"update_status"` |
| `session_number` | integer | Nth session this month | `5` |

**Source:** Frontend — Portal components.

---

## 4. User Properties

Set these on the Mixpanel user profile via `mixpanel.people.set()` (frontend) or the
Engage API (backend). Update on login and on relevant actions.

### Identity Properties (set once or on change)

| Property | Type | Source | Description |
|---|---|---|---|
| `$name` | string | `tbl_usuarios.nombre` | Full name (Mixpanel reserved) |
| `$email` | string | `tbl_usuarios.correo` | Email (Mixpanel reserved) |
| `user_id` | integer | `tbl_usuarios.id` | Internal user ID |
| `role` | string | `tbl_usuarios.rol` / permissions | User role |
| `locacion_id` | integer | `tbl_usuarios.id_locacion` | Branch/location ID |
| `locacion_name` | string | `tbl_locaciones.nombre` | Branch name |
| `empresa` | string | derived | Company name |
| `permissions` | string[] | derived | Permission list |
| `portal` | string | derived | Primary portal (`main`/`customer`/`operator`) |
| `created_at` | datetime | `tbl_usuarios.fecha_alta` | Account creation date |

### Behavioral Properties (increment/update on activity)

| Property | Type | Update Trigger | Description |
|---|---|---|---|
| `last_login` | datetime | On login | Last login timestamp |
| `total_logins` | integer | On login (`$add: 1`) | Lifetime login count |
| `active_days` | integer | Daily (deduplicated) | Total distinct active days |
| `last_active` | datetime | On any event | Last activity timestamp |
| `total_remisiones` | integer | On `remision_created` (`$add: 1`) | Lifetime remisiones created |
| `total_facturas` | integer | On `cfdi_timbrado_success` (`$add: 1`) | Lifetime invoices stamped |
| `total_cobros` | integer | On `payment_received` (`$add: 1`) | Lifetime payments recorded |
| `total_revenue_tracked` | number | On `payment_received` (`$add: monto`) | Cumulative revenue |
| `preferred_module` | string | Computed from `page_viewed` | Most-used module |

### Example: Setting User Properties on Login

```typescript
// Frontend — after successful login
this.mixpanelService.identify(user.id);
this.mixpanelService.people.set({
  $name: user.nombre,
  $email: user.correo,
  role: user.rol,
  locacion_id: user.idLocacion,
  locacion_name: user.locacionNombre,
  portal: 'main',
  last_login: new Date().toISOString(),
});
this.mixpanelService.people.increment('total_logins', 1);
```

---

## 5. Key Metrics & Dashboards

### Dashboard 1: Revenue & CFDI

**Purpose:** Monitor invoice stamping health and revenue flow.

| Report | Type | Definition |
|---|---|---|
| Timbrado Success Rate | Line (daily) | `cfdi_timbrado_success` / (`cfdi_timbrado_success` + `cfdi_timbrado_failed`) * 100 |
| Daily Timbrado Volume | Bar (daily) | Count of `cfdi_timbrado_attempted` |
| Top CFDI Failure Reasons | Pie | `cfdi_timbrado_failed` grouped by `error_code`, top 10 |
| Revenue by Client (monthly) | Bar (monthly) | Sum of `payment_received.monto` grouped by `client_id` |
| Revenue by Month | Line (monthly) | Sum of `cfdi_timbrado_success.total` |
| Payment Collection Rate | Line (weekly) | Count of `payment_received` / Count of `cfdi_timbrado_success` |
| Avg Days: Timbrado to Cobro | Line (weekly) | Average of `payment_received.days_since_timbrado` |
| Cancellation Rate | Line (monthly) | Count of `cfdi_cancelled` / Count of `cfdi_timbrado_success` * 100 |
| Pipeline Funnel | Funnel | `cotizacion_created` -> `cotizacion_accepted` -> `remision_created` -> `cfdi_timbrado_success` -> `payment_received` |
| PAC Response Time | Line (daily) | P50, P95 of `cfdi_timbrado_success.response_time_ms` |

### Dashboard 2: Operations

**Purpose:** Track shipment throughput and operational efficiency.

| Report | Type | Definition |
|---|---|---|
| Remisiones per Day | Bar (daily) | Count of `remision_created` |
| Remision Status Funnel | Funnel | `remision_status_changed` from documentada -> en_transito -> entregada |
| Avg Delivery Time (hours) | Line (weekly) | Avg time from status `documentada` to `entregada` |
| Manifests per Day | Bar (daily) | Count of `manifiesto_created` |
| Avg Remisiones per Manifest | Line (weekly) | Avg of `manifiesto_created.num_remisiones` |
| Top Routes | Table | `remision_created` grouped by `origen` + `destino`, sorted by count |
| Top Origins | Bar | `remision_created` grouped by `origen`, top 15 |
| Top Destinations | Bar | `remision_created` grouped by `destino`, top 15 |
| Bulk Upload Success Rate | Line (weekly) | Avg of (`bulk_upload_completed.success_rows` / `total_rows`) * 100 |
| Exchange Rate API Health | Line (daily) | `exchange_rate_fetched` success rate, avg response time |

### Dashboard 3: User Adoption

**Purpose:** Understand how users interact with TruckSoft across all portals.

| Report | Type | Definition |
|---|---|---|
| DAU / WAU / MAU | Line (daily) | Unique users with any event, 1d / 7d / 30d windows |
| Stickiness (DAU/MAU) | Line (daily) | DAU / MAU ratio |
| Feature Usage Heatmap | Table | Count of `page_viewed` grouped by `page_path`, top 30 |
| Module Usage Distribution | Pie | `page_viewed` grouped by module (extracted from path) |
| Portal Adoption | Bar (monthly) | Unique users per `portal` value |
| Session Duration (median) | Line (weekly) | Median session length from Mixpanel session tracking |
| New vs Returning Users | Stacked bar | Users by first-seen date vs returning |
| Login Frequency Distribution | Histogram | Distribution of `total_logins` per user |
| Least Used Modules | Table | Bottom 10 `page_path` by unique users |
| Mobile vs Desktop | Pie | Events grouped by `device_type` |

### Dashboard 4: Data Quality

**Purpose:** Monitor data hygiene issues discovered in the fiscal data audit.

| Report | Type | Definition |
|---|---|---|
| Clients with Fiscal Errors | Counter | Count of `client_created` where `has_valid_rfc` = false |
| Missing CSF Rate | Line (weekly) | `client_created` where `has_csf` = false / total `client_created` |
| Timbrado Failures by Error Type | Stacked bar (daily) | `cfdi_timbrado_failed` grouped by `error_source` |
| Validation Failures Trend | Line (daily) | `cfdi_timbrado_failed` where `error_source` = "validation" |
| RFC Error Rate by Client | Table | `cfdi_timbrado_failed` with `error_code` containing "rfc", grouped by `receptor_rfc` |
| Bulk Upload Error Rate | Line (weekly) | Avg of `bulk_upload_completed.error_rows` / `total_rows` |

---

## 6. Implementation Plan

### Phase 1: Setup + Revenue Events (Weeks 1-2)

| Day | Task | Owner | Deliverable |
|---|---|---|---|
| D1 | Install `mixpanel-browser` in Angular project | Frontend | `package.json` updated |
| D1 | Install `mixpanel-java` in Spring Boot project | Backend | `pom.xml` updated |
| D2 | Create `MixpanelService` (Angular) | Frontend | Service + tests |
| D2 | Create `MixpanelService` (Spring Boot) | Backend | Bean + config |
| D3 | Implement `cfdi_timbrado_attempted/success/failed` | Backend | 3 events in timbrado flow |
| D4 | Implement `cfdi_cancelled` | Backend | 1 event |
| D5 | Implement `payment_received`, `payment_complement_created` | Backend | 2 events |
| D6 | Implement `credit_note_created` | Backend | 1 event |
| D7 | Implement `remision_created`, `remision_status_changed` | Backend | 2 events |
| D8 | Implement `cotizacion_created`, `cotizacion_accepted` | Backend | 2 events |
| D9 | QA: verify all 11 events in Mixpanel QA project | QA | Verification report |
| D10 | Build Revenue & CFDI dashboard in Mixpanel | Product | Dashboard live |

**Phase 1 Definition of Done:**
- All 11 Tier 1 events firing in QA with correct properties
- Revenue & CFDI dashboard built with all 10 reports
- Deployed to production
- Timbrado failure alert configured

### Phase 2: Operations Events (Weeks 3-4)

| Day | Task | Owner | Deliverable |
|---|---|---|---|
| D11 | Implement `manifiesto_created` | Backend | 1 event |
| D12 | Implement `carta_porte_generated` | Backend | 1 event |
| D13 | Implement `bulk_upload_completed` | Backend | 1 event |
| D14 | Implement `pdf_exported` | Backend | 1 event |
| D15 | Implement `exchange_rate_fetched` | Backend | 1 event |
| D16-17 | QA: verify 5 events | QA | Verification |
| D18-20 | Build Operations dashboard + Data Quality dashboard | Product | Dashboards live |

### Phase 3: User Behavior (Weeks 5-6)

| Day | Task | Owner | Deliverable |
|---|---|---|---|
| D21 | Implement `page_viewed` via Router interceptor | Frontend | Auto-tracking |
| D22 | Implement `search_performed` | Frontend | 1 event |
| D23 | Implement `login_success` / `login_failed` | Frontend + Backend | 2 events |
| D24 | Implement `client_created` / `client_updated` | Backend | 2 events |
| D25 | Implement `portal_accessed` | Frontend | 1 event |
| D26 | Set up user profile properties (people.set) | Frontend + Backend | User profiles |
| D27-28 | QA: verify 7 events + profiles | QA | Verification |
| D29-30 | Build User Adoption dashboard | Product | Dashboard live |

---

## 7. Implementation Code Examples

### 7.1 Frontend: Angular MixpanelService

```typescript
// src/app/core/services/mixpanel.service.ts

import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';
import mixpanel from 'mixpanel-browser';

export interface MixpanelEventProperties {
  [key: string]: string | number | boolean | string[] | number[] | object | null;
}

@Injectable({
  providedIn: 'root',
})
export class MixpanelService {
  private initialized = false;

  constructor() {
    this.init();
  }

  private init(): void {
    if (this.initialized) return;

    mixpanel.init(environment.mixpanel.token, {
      debug: environment.mixpanel.debug,
      track_pageview: false,           // We handle page views manually
      persistence: 'localStorage',
      api_host: environment.mixpanel.apiHost,
      ignore_dnt: false,               // Respect Do Not Track
      batch_size: 10,                  // Batch events for performance
      batch_flush_interval_ms: 5000,   // Flush every 5 seconds
    });

    // Register super properties (sent with every event)
    mixpanel.register({
      app_version: environment.appVersion || '1.0.0',
      platform: 'web',
      environment: environment.production ? 'production' : 'qa',
    });

    this.initialized = true;
  }

  /**
   * Identify a user after login. Call once per session.
   * Uses tbl_usuarios.id as the distinct_id.
   */
  identify(userId: number): void {
    mixpanel.identify(userId.toString());
  }

  /**
   * Set user profile properties.
   */
  setUserProperties(properties: MixpanelEventProperties): void {
    mixpanel.people.set(properties);
  }

  /**
   * Increment a numeric user property.
   */
  incrementUserProperty(property: string, value: number = 1): void {
    mixpanel.people.increment(property, value);
  }

  /**
   * Track a named event with properties.
   */
  track(eventName: string, properties: MixpanelEventProperties = {}): void {
    if (!this.initialized) {
      console.warn(`[Mixpanel] Not initialized. Dropping event: ${eventName}`);
      return;
    }

    mixpanel.track(eventName, {
      ...properties,
      timestamp: new Date().toISOString(),
    });
  }

  /**
   * Track a page view. Called from the Router interceptor.
   */
  trackPageView(pagePath: string, pageTitle: string, portal: string): void {
    this.track('page_viewed', {
      page_path: pagePath,
      page_title: pageTitle,
      portal,
    });
  }

  /**
   * Reset identity on logout.
   */
  reset(): void {
    mixpanel.reset();
  }

  /**
   * Start timing an event (for measuring duration).
   * Call track() with the same event name later to record the duration.
   */
  timeEvent(eventName: string): void {
    mixpanel.time_event(eventName);
  }

  /**
   * Alias an anonymous ID to a known user ID (call once, on first login).
   */
  alias(userId: number): void {
    mixpanel.alias(userId.toString());
  }
}
```

### 7.2 Frontend: Router Interceptor for Page Views

```typescript
// src/app/core/interceptors/page-view.tracker.ts

import { Injectable } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';
import { MixpanelService } from '../services/mixpanel.service';
import { Title } from '@angular/platform-browser';

@Injectable({
  providedIn: 'root',
})
export class PageViewTracker {
  private previousPath: string = '';
  private portal: string = 'main';

  constructor(
    private router: Router,
    private mixpanelService: MixpanelService,
    private titleService: Title
  ) {}

  /**
   * Call this once in AppComponent.ngOnInit() to start tracking.
   */
  startTracking(portal: string = 'main'): void {
    this.portal = portal;

    this.router.events
      .pipe(filter((event) => event instanceof NavigationEnd))
      .subscribe((event: NavigationEnd) => {
        const pagePath = event.urlAfterRedirects;
        const pageTitle = this.titleService.getTitle() || '';

        this.mixpanelService.track('page_viewed', {
          page_path: pagePath,
          page_title: pageTitle,
          portal: this.portal,
          referrer_path: this.previousPath,
        });

        this.previousPath = pagePath;
      });
  }
}
```

### 7.3 Frontend: Usage in AppComponent

```typescript
// src/app/app.component.ts

import { Component, OnInit } from '@angular/core';
import { PageViewTracker } from './core/interceptors/page-view.tracker';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
})
export class AppComponent implements OnInit {
  constructor(private pageViewTracker: PageViewTracker) {}

  ngOnInit(): void {
    // Determine portal from URL or config
    const portal = this.detectPortal();
    this.pageViewTracker.startTracking(portal);
  }

  private detectPortal(): string {
    const host = window.location.hostname;
    if (host.includes('portal-cliente') || host.includes('customer')) return 'customer';
    if (host.includes('portal-operador') || host.includes('operator')) return 'operator';
    return 'main';
  }
}
```

### 7.4 Frontend: Event Tracking in a Component

```typescript
// Example: Tracking search in RemisionesListComponent

import { Component, OnInit } from '@angular/core';
import { MixpanelService } from '../../core/services/mixpanel.service';

@Component({
  selector: 'app-remisiones-list',
  templateUrl: './remisiones-list.component.html',
})
export class RemisionesListComponent implements OnInit {
  filters: any = {};
  searchTerm: string = '';
  results: any[] = [];

  constructor(private mixpanelService: MixpanelService) {}

  ngOnInit(): void {}

  onSearch(): void {
    const startTime = performance.now();

    this.remisionesService.search(this.searchTerm, this.filters).subscribe(
      (results) => {
        this.results = results;
        const responseTime = Math.round(performance.now() - startTime);

        this.mixpanelService.track('search_performed', {
          module: 'remisiones',
          search_term: this.searchTerm,
          filters_applied: this.filters,
          results_count: results.length,
          response_time_ms: responseTime,
        });
      }
    );
  }
}
```

### 7.5 Frontend: Login Tracking

```typescript
// src/app/auth/login/login.component.ts

import { Component } from '@angular/core';
import { MixpanelService } from '../../core/services/mixpanel.service';
import { AuthService } from '../../core/services/auth.service';

@Component({
  selector: 'app-login',
  templateUrl: './login.component.html',
})
export class LoginComponent {
  username: string = '';
  password: string = '';

  constructor(
    private authService: AuthService,
    private mixpanelService: MixpanelService
  ) {}

  onLogin(): void {
    this.authService.login(this.username, this.password).subscribe(
      (response) => {
        const user = response.user;

        // Identify user in Mixpanel
        this.mixpanelService.identify(user.id);

        // Set user profile properties
        this.mixpanelService.setUserProperties({
          $name: user.nombre,
          $email: user.correo,
          role: user.rol,
          locacion_id: user.idLocacion,
          locacion_name: user.locacionNombre,
          portal: 'main',
          last_login: new Date().toISOString(),
        });

        // Increment login count
        this.mixpanelService.incrementUserProperty('total_logins');

        // Track login event
        this.mixpanelService.track('login_success', {
          user_id: user.id,
          portal: 'main',
          auth_method: 'password',
          device_type: this.getDeviceType(),
          browser: this.getBrowserName(),
        });
      },
      (error) => {
        this.mixpanelService.track('login_failed', {
          attempted_username: this.hashString(this.username),
          failure_reason: error.status === 401 ? 'invalid_password' : 'server_error',
          portal: 'main',
        });
      }
    );
  }

  private getDeviceType(): string {
    return window.innerWidth < 768 ? 'mobile' : 'desktop';
  }

  private getBrowserName(): string {
    const ua = navigator.userAgent;
    if (ua.includes('Chrome')) return 'Chrome';
    if (ua.includes('Firefox')) return 'Firefox';
    if (ua.includes('Safari')) return 'Safari';
    if (ua.includes('Edge')) return 'Edge';
    return 'Other';
  }

  private hashString(input: string): string {
    // Simple hash for PII protection — use a proper hash in production
    let hash = 0;
    for (let i = 0; i < input.length; i++) {
      const char = input.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash |= 0;
    }
    return Math.abs(hash).toString(16);
  }
}
```

### 7.6 Backend: Spring Boot MixpanelService

```java
// src/main/java/com/trucksoft/tms/service/MixpanelService.java

package com.trucksoft.tms.service;

import com.mixpanel.mixpanelapi.MessageBuilder;
import com.mixpanel.mixpanelapi.MixpanelAPI;
import com.mixpanel.mixpanelapi.ClientDelivery;
import org.json.JSONObject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;
import java.io.IOException;
import java.time.Instant;
import java.util.Map;

@Service
public class MixpanelService {

    private static final Logger log = LoggerFactory.getLogger(MixpanelService.class);

    @Value("${mixpanel.token}")
    private String token;

    @Value("${mixpanel.enabled:true}")
    private boolean enabled;

    private MessageBuilder messageBuilder;
    private MixpanelAPI mixpanelApi;

    @PostConstruct
    public void init() {
        this.messageBuilder = new MessageBuilder(token);
        this.mixpanelApi = new MixpanelAPI();
        log.info("MixpanelService initialized. Enabled: {}", enabled);
    }

    /**
     * Track an event asynchronously. This method never throws — tracking
     * failures are logged but do not affect business logic.
     *
     * @param userId    tbl_usuarios.id of the user performing the action
     * @param eventName snake_case event name
     * @param properties event properties map
     */
    @Async("mixpanelExecutor")
    public void track(Long userId, String eventName, Map<String, Object> properties) {
        if (!enabled) return;

        try {
            JSONObject props = new JSONObject(properties);
            props.put("user_id", userId);
            props.put("timestamp", Instant.now().toString());

            JSONObject event = messageBuilder.event(
                userId.toString(),
                eventName,
                props
            );

            ClientDelivery delivery = new ClientDelivery();
            delivery.addMessage(event);
            mixpanelApi.deliver(delivery);

            log.debug("Mixpanel event sent: {} for user {}", eventName, userId);
        } catch (IOException e) {
            log.error("Failed to send Mixpanel event: {} for user {}. Error: {}",
                eventName, userId, e.getMessage());
        } catch (Exception e) {
            log.error("Unexpected error sending Mixpanel event: {}. Error: {}",
                eventName, e.getMessage());
        }
    }

    /**
     * Set user profile properties.
     */
    @Async("mixpanelExecutor")
    public void setUserProfile(Long userId, Map<String, Object> properties) {
        if (!enabled) return;

        try {
            JSONObject props = new JSONObject(properties);
            JSONObject update = messageBuilder.set(userId.toString(), props);

            ClientDelivery delivery = new ClientDelivery();
            delivery.addMessage(update);
            mixpanelApi.deliver(delivery);

            log.debug("Mixpanel profile updated for user {}", userId);
        } catch (IOException e) {
            log.error("Failed to update Mixpanel profile for user {}. Error: {}",
                userId, e.getMessage());
        }
    }

    /**
     * Increment a numeric property on the user profile.
     */
    @Async("mixpanelExecutor")
    public void incrementUserProperty(Long userId, String property, long value) {
        if (!enabled) return;

        try {
            Map<String, Long> incrementProps = Map.of(property, value);
            JSONObject update = messageBuilder.increment(
                userId.toString(), new JSONObject(incrementProps));

            ClientDelivery delivery = new ClientDelivery();
            delivery.addMessage(update);
            mixpanelApi.deliver(delivery);
        } catch (IOException e) {
            log.error("Failed to increment Mixpanel property {} for user {}. Error: {}",
                property, userId, e.getMessage());
        }
    }
}
```

### 7.7 Backend: Async Executor Configuration

```java
// src/main/java/com/trucksoft/tms/config/MixpanelConfig.java

package com.trucksoft.tms.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class MixpanelConfig {

    /**
     * Dedicated thread pool for Mixpanel event delivery.
     * Small pool — analytics should not consume application threads.
     */
    @Bean(name = "mixpanelExecutor")
    public Executor mixpanelExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("mixpanel-");
        executor.setRejectedExecutionHandler((r, e) -> {
            // Drop analytics events if queue is full — never block business logic
            org.slf4j.LoggerFactory.getLogger("MixpanelConfig")
                .warn("Mixpanel event queue full. Dropping event.");
        });
        executor.initialize();
        return executor;
    }
}
```

### 7.8 Backend: AOP Aspect for Controller Tracking

```java
// src/main/java/com/trucksoft/tms/aspect/MixpanelTrackingAspect.java

package com.trucksoft.tms.aspect;

import com.trucksoft.tms.annotation.TrackEvent;
import com.trucksoft.tms.service.MixpanelService;
import com.trucksoft.tms.security.SecurityUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Aspect
@Component
public class MixpanelTrackingAspect {

    private static final Logger log = LoggerFactory.getLogger(MixpanelTrackingAspect.class);

    private final MixpanelService mixpanelService;

    public MixpanelTrackingAspect(MixpanelService mixpanelService) {
        this.mixpanelService = mixpanelService;
    }

    /**
     * Intercepts methods annotated with @TrackEvent and sends analytics
     * after successful execution.
     */
    @Around("@annotation(trackEvent)")
    public Object trackEvent(ProceedingJoinPoint joinPoint, TrackEvent trackEvent)
            throws Throwable {

        long startTime = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long duration = System.currentTimeMillis() - startTime;

        try {
            Long userId = SecurityUtils.getCurrentUserId();
            Map<String, Object> properties = new HashMap<>();
            properties.put("controller", joinPoint.getTarget().getClass().getSimpleName());
            properties.put("method", joinPoint.getSignature().getName());
            properties.put("execution_time_ms", duration);
            properties.put("locacion_id", SecurityUtils.getCurrentLocacionId());

            mixpanelService.track(userId, trackEvent.value(), properties);
        } catch (Exception e) {
            log.warn("Failed to track event via AOP: {}", e.getMessage());
        }

        return result;
    }
}
```

### 7.9 Backend: TrackEvent Annotation

```java
// src/main/java/com/trucksoft/tms/annotation/TrackEvent.java

package com.trucksoft.tms.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotate a controller or service method to automatically track
 * the event in Mixpanel after successful execution.
 *
 * Usage: @TrackEvent("remision_created")
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackEvent {
    /**
     * The event name (snake_case).
     */
    String value();
}
```

### 7.10 Backend: Timbrado Event Tracking (PrefacturasController)

This is the most critical tracking implementation — it covers the CFDI stamping flow.

```java
// In PrefacturasController.java (or PrefacturasService.java)

package com.trucksoft.tms.controller;

import com.trucksoft.tms.service.MixpanelService;
import com.trucksoft.tms.security.SecurityUtils;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;
import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/prefacturas")
public class PrefacturasController {

    private final PrefacturasService prefacturasService;
    private final MixpanelService mixpanelService;
    private final FacturapiService facturapiService;

    public PrefacturasController(
            PrefacturasService prefacturasService,
            MixpanelService mixpanelService,
            FacturapiService facturapiService) {
        this.prefacturasService = prefacturasService;
        this.mixpanelService = mixpanelService;
        this.facturapiService = facturapiService;
    }

    @PostMapping("/{id}/timbrar")
    public ResponseEntity<?> timbrar(@PathVariable Long id) {
        Long userId = SecurityUtils.getCurrentUserId();
        Long locacionId = SecurityUtils.getCurrentLocacionId();
        Prefactura prefactura = prefacturasService.findById(id);

        // --- Track: timbrado attempted ---
        Map<String, Object> attemptProps = new HashMap<>();
        attemptProps.put("prefactura_id", prefactura.getId());
        attemptProps.put("remision_ids", prefactura.getRemisiones().stream()
            .map(r -> r.getId()).collect(Collectors.toList()));
        attemptProps.put("tipo_comprobante", prefactura.getTipoComprobante());
        attemptProps.put("receptor_rfc", maskRfc(prefactura.getReceptorRfc()));
        attemptProps.put("receptor_nombre", prefactura.getReceptorNombre());
        attemptProps.put("subtotal", prefactura.getSubtotal());
        attemptProps.put("total", prefactura.getTotal());
        attemptProps.put("moneda", prefactura.getMoneda());
        attemptProps.put("uso_cfdi", prefactura.getUsoCfdi());
        attemptProps.put("metodo_pago", prefactura.getMetodoPago());
        attemptProps.put("regimen_fiscal", prefactura.getRegimenFiscal());
        attemptProps.put("locacion_id", locacionId);
        mixpanelService.track(userId, "cfdi_timbrado_attempted", attemptProps);

        try {
            long startTime = System.currentTimeMillis();

            // Call PAC (Facturapi) to stamp the invoice
            FacturapiResponse response = facturapiService.timbrar(prefactura);
            long responseTime = System.currentTimeMillis() - startTime;

            // Update prefactura with fiscal data
            prefactura.setUuidFiscal(response.getUuid());
            prefactura.setFolioFiscal(response.getFolio());
            prefactura.setEstatus("timbrada");
            prefacturasService.save(prefactura);

            // --- Track: timbrado success ---
            Map<String, Object> successProps = new HashMap<>();
            successProps.put("prefactura_id", prefactura.getId());
            successProps.put("uuid_fiscal", response.getUuid());
            successProps.put("folio_fiscal", response.getFolio());
            successProps.put("serie", prefactura.getSerie());
            successProps.put("total", prefactura.getTotal());
            successProps.put("pac_provider", "facturapi");
            successProps.put("response_time_ms", responseTime);
            successProps.put("tipo_comprobante", prefactura.getTipoComprobante());
            successProps.put("locacion_id", locacionId);
            mixpanelService.track(userId, "cfdi_timbrado_success", successProps);

            // Increment user profile counter
            mixpanelService.incrementUserProperty(userId, "total_facturas", 1);

            return ResponseEntity.ok(prefactura);

        } catch (FacturapiException e) {
            // --- Track: timbrado failed ---
            Map<String, Object> failProps = new HashMap<>();
            failProps.put("prefactura_id", prefactura.getId());
            failProps.put("error_code", e.getErrorCode());
            failProps.put("error_message", e.getMessage());
            failProps.put("error_source", "facturapi");
            failProps.put("total", prefactura.getTotal());
            failProps.put("receptor_rfc", maskRfc(prefactura.getReceptorRfc()));
            failProps.put("tipo_comprobante", prefactura.getTipoComprobante());
            failProps.put("retry_count", 0);
            failProps.put("locacion_id", locacionId);
            mixpanelService.track(userId, "cfdi_timbrado_failed", failProps);

            return ResponseEntity.badRequest().body(Map.of(
                "error", e.getErrorCode(),
                "message", e.getMessage()
            ));

        } catch (Exception e) {
            // --- Track: timbrado failed (unexpected) ---
            Map<String, Object> failProps = new HashMap<>();
            failProps.put("prefactura_id", prefactura.getId());
            failProps.put("error_code", "unexpected_error");
            failProps.put("error_message", e.getMessage());
            failProps.put("error_source", "internal");
            failProps.put("total", prefactura.getTotal());
            failProps.put("receptor_rfc", maskRfc(prefactura.getReceptorRfc()));
            failProps.put("tipo_comprobante", prefactura.getTipoComprobante());
            failProps.put("retry_count", 0);
            failProps.put("locacion_id", locacionId);
            mixpanelService.track(userId, "cfdi_timbrado_failed", failProps);

            throw e;
        }
    }

    /**
     * Mask RFC for privacy: show first 3 and last 3 chars only.
     * "XAXX010101000" -> "XAX***010101***"
     */
    private String maskRfc(String rfc) {
        if (rfc == null || rfc.length() < 6) return "***";
        return rfc.substring(0, 3) + "***" + rfc.substring(rfc.length() - 3) + "***";
    }
}
```

### 7.11 Backend: Remision Event Tracking

```java
// In RemisionesController.java — create and status change tracking

@RestController
@RequestMapping("/api/remisiones")
public class RemisionesController {

    private final RemisionesService remisionesService;
    private final MixpanelService mixpanelService;

    // ... constructor injection ...

    @PostMapping
    public ResponseEntity<Remision> create(@RequestBody RemisionDTO dto) {
        Long userId = SecurityUtils.getCurrentUserId();
        Long locacionId = SecurityUtils.getCurrentLocacionId();

        Remision remision = remisionesService.create(dto);

        // --- Track: remision created ---
        Map<String, Object> props = new HashMap<>();
        props.put("remision_id", remision.getId());
        props.put("client_id", remision.getClienteId());
        props.put("origen", remision.getOrigen());
        props.put("destino", remision.getDestino());
        props.put("tipo_servicio", remision.getTipoServicio());
        props.put("tipo_unidad", remision.getTipoUnidad());
        props.put("tarifa", remision.getTarifa());
        props.put("moneda", remision.getMoneda());
        props.put("peso_kg", remision.getPesoKg());
        props.put("num_bultos", remision.getNumBultos());
        props.put("fecha_recoleccion", remision.getFechaRecoleccion().toString());
        props.put("source", dto.getSource() != null ? dto.getSource() : "manual");
        props.put("locacion_id", locacionId);
        mixpanelService.track(userId, "remision_created", props);

        // Increment user counter
        mixpanelService.incrementUserProperty(userId, "total_remisiones", 1);

        return ResponseEntity.ok(remision);
    }

    @PatchMapping("/{id}/status")
    public ResponseEntity<Remision> updateStatus(
            @PathVariable Long id,
            @RequestBody StatusUpdateDTO dto) {
        Long userId = SecurityUtils.getCurrentUserId();
        Long locacionId = SecurityUtils.getCurrentLocacionId();

        Remision remision = remisionesService.findById(id);
        String fromStatus = remision.getEstatus();
        long hoursInPreviousStatus = remisionesService.hoursInCurrentStatus(remision);

        remision = remisionesService.updateStatus(id, dto.getNewStatus());

        // --- Track: status changed ---
        Map<String, Object> props = new HashMap<>();
        props.put("remision_id", remision.getId());
        props.put("from_status", fromStatus);
        props.put("to_status", dto.getNewStatus());
        props.put("time_in_previous_status_hours", hoursInPreviousStatus);
        props.put("client_id", remision.getClienteId());
        props.put("locacion_id", locacionId);
        mixpanelService.track(userId, "remision_status_changed", props);

        return ResponseEntity.ok(remision);
    }
}
```

---

## 8. Privacy & Compliance

### 8.1 PII Protection Rules

| Data Type | Treatment | Example |
|---|---|---|
| RFC (tax ID) | Mask: show first 3 + last 3 chars | `"XAX***000***"` |
| Email | Hash with SHA-256 before sending | `"a1b2c3d4..."` |
| Phone number | Never include in events | Omit |
| Full address | City + State only | `"Monterrey, NL"` |
| Bank account numbers | Never include | Omit |
| Passwords | Never include | Omit |
| IP addresses | Country-level geo only | `"MX"` |
| User names | OK in user profile ($name) | Allowed |

### 8.2 Mexican Data Protection (LFPDPPP) Compliance

Mexico's Ley Federal de Proteccion de Datos Personales en Posesion de los Particulares
requires:

1. **Privacy Notice (Aviso de Privacidad):** Update the existing privacy notice to
   disclose that analytics data is collected and processed. Mention Mixpanel as a
   data processor.

2. **Consent:** Obtain informed consent for analytics tracking. Implement a cookie/
   analytics consent banner for first-time users. Track only after consent is given.

3. **Purpose Limitation:** Analytics data is collected solely for service improvement,
   operational monitoring, and compliance verification. Do not use for advertising.

4. **Right of Access/Deletion (ARCO rights):** Implement a process to:
   - Export a user's Mixpanel data on request
   - Delete a user's Mixpanel data on request (use Mixpanel's GDPR API)

### 8.3 Implementation: Consent Check

```typescript
// Frontend — only initialize Mixpanel if user has consented
export class MixpanelService {
  private init(): void {
    const consent = localStorage.getItem('analytics_consent');
    if (consent !== 'granted') {
      this.initialized = false;
      return; // Do not initialize Mixpanel
    }
    // ... proceed with initialization
  }
}
```

### 8.4 Data Retention

| Data | Retention Period | Justification |
|---|---|---|
| Mixpanel events | 12 months | Sufficient for YoY analysis |
| User profiles | Lifetime (until deletion request) | Ongoing relationship |
| Raw event logs (backend) | 90 days | Debugging window |

### 8.5 Mixpanel GDPR/Data Deletion API

```java
// Backend endpoint for user data deletion requests
@DeleteMapping("/api/users/{id}/analytics-data")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<?> deleteAnalyticsData(@PathVariable Long id) {
    // Call Mixpanel's deletion API
    // POST https://mixpanel.com/api/app/data-deletions/v3.0/
    // with the user's distinct_id

    mixpanelService.requestDataDeletion(id);
    return ResponseEntity.ok().build();
}
```

---

## 9. Alerting

### 9.1 Critical Alerts

Configure these alerts using Mixpanel Custom Alerts or an external monitoring layer.

| Alert | Condition | Channel | Severity |
|---|---|---|---|
| Timbrado failure spike | `cfdi_timbrado_failed` rate > 10% in last 1 hour | Slack #tms-alerts | CRITICAL |
| Zero remisiones | 0 `remision_created` events in 2 consecutive hours (business hours only, Mon-Sat 8am-8pm CST) | Slack #tms-alerts | WARNING |
| Facturapi down | 3+ consecutive `cfdi_timbrado_failed` with `error_source=facturapi` in 10 minutes | Slack #tms-alerts + PagerDuty | CRITICAL |
| Banxico API down | `exchange_rate_fetched` with `success=false` for 30+ minutes | Slack #tms-alerts | WARNING |
| Timbrado volume anomaly | `cfdi_timbrado_attempted` volume drops > 50% vs same day last week | Slack #tms-alerts | WARNING |
| High cancellation rate | `cfdi_cancelled` count > 5 in 1 hour | Slack #tms-alerts | WARNING |

### 9.2 Slack Webhook Integration

```java
// src/main/java/com/trucksoft/tms/service/AlertService.java

package com.trucksoft.tms.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.Map;

@Service
public class AlertService {

    @Value("${slack.webhook.tms-alerts}")
    private String slackWebhookUrl;

    private final RestTemplate restTemplate = new RestTemplate();

    @Async("mixpanelExecutor")
    public void sendSlackAlert(String severity, String title, String message) {
        String emoji = switch (severity) {
            case "CRITICAL" -> ":red_circle:";
            case "WARNING" -> ":warning:";
            default -> ":information_source:";
        };

        String payload = String.format("""
            {
              "text": "%s *[%s] %s*",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "%s *[%s] %s*\\n%s"
                  }
                }
              ]
            }
            """, emoji, severity, title, emoji, severity, title, message);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<String> request = new HttpEntity<>(payload, headers);

        try {
            restTemplate.postForEntity(slackWebhookUrl, request, String.class);
        } catch (Exception e) {
            // Log but don't throw — alerting failures should not cascade
            org.slf4j.LoggerFactory.getLogger(AlertService.class)
                .error("Failed to send Slack alert: {}", e.getMessage());
        }
    }
}
```

### 9.3 Timbrado Failure Rate Monitor

```java
// src/main/java/com/trucksoft/tms/scheduler/TimbradoAlertScheduler.java

package com.trucksoft.tms.scheduler;

import com.trucksoft.tms.service.AlertService;
import com.trucksoft.tms.repository.PrefacturaRepository;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Component
public class TimbradoAlertScheduler {

    private final PrefacturaRepository prefacturaRepository;
    private final AlertService alertService;

    public TimbradoAlertScheduler(
            PrefacturaRepository prefacturaRepository,
            AlertService alertService) {
        this.prefacturaRepository = prefacturaRepository;
        this.alertService = alertService;
    }

    /**
     * Check timbrado failure rate every 15 minutes.
     */
    @Scheduled(fixedRate = 900000) // 15 minutes
    public void checkTimbradoFailureRate() {
        LocalDateTime oneHourAgo = LocalDateTime.now().minusHours(1);

        long attempted = prefacturaRepository.countTimbradoAttemptsSince(oneHourAgo);
        long failed = prefacturaRepository.countTimbradoFailuresSince(oneHourAgo);

        if (attempted == 0) return;

        double failureRate = (double) failed / attempted * 100;

        if (failureRate > 10.0) {
            alertService.sendSlackAlert(
                "CRITICAL",
                "Timbrado Failure Rate High",
                String.format(
                    "Failure rate: %.1f%% (%d failed out of %d attempted) in the last hour.\\n"
                    + "Check Mixpanel dashboard: https://mixpanel.com/project/%s/view",
                    failureRate, failed, attempted, "2987343"
                )
            );
        }
    }

    /**
     * Check for zero remisiones during business hours (Mon-Sat, 8am-8pm CST).
     */
    @Scheduled(fixedRate = 3600000) // 1 hour
    public void checkZeroRemisiones() {
        LocalDateTime now = LocalDateTime.now();
        int hour = now.getHour();
        int dayOfWeek = now.getDayOfWeek().getValue(); // 1=Mon, 7=Sun

        // Only check during business hours (Mon-Sat, 8am-8pm)
        if (dayOfWeek == 7 || hour < 8 || hour >= 20) return;

        LocalDateTime twoHoursAgo = now.minusHours(2);
        long remisionCount = prefacturaRepository.countRemisionesSince(twoHoursAgo);

        if (remisionCount == 0) {
            alertService.sendSlackAlert(
                "WARNING",
                "Zero Remisiones Created",
                "No remisiones have been created in the last 2 hours during business hours."
            );
        }
    }
}
```

---

## 10. Cost Estimate

### 10.1 Monthly Event Volume Estimate

Based on the system's current scale and typical usage patterns:

| Event | Estimated Monthly Volume | Calculation |
|---|---|---|
| `cfdi_timbrado_attempted` | 3,000 | ~150/day * 20 business days |
| `cfdi_timbrado_success` | 2,850 | 95% success rate |
| `cfdi_timbrado_failed` | 150 | 5% failure rate |
| `cfdi_cancelled` | 100 | ~3% of stamped invoices |
| `payment_received` | 2,500 | Multiple payments per invoice cycle |
| `payment_complement_created` | 1,500 | PPD-heavy client base |
| `credit_note_created` | 200 | ~7% of invoices |
| `remision_created` | 5,000 | ~250/day * 20 days |
| `remision_status_changed` | 20,000 | ~4 transitions per remision |
| `cotizacion_created` | 2,000 | ~100/day |
| `cotizacion_accepted` | 1,200 | 60% conversion |
| `manifiesto_created` | 1,500 | ~75/day |
| `carta_porte_generated` | 1,500 | 1:1 with manifests |
| `bulk_upload_completed` | 200 | ~10/day |
| `pdf_exported` | 8,000 | ~400/day |
| `exchange_rate_fetched` | 600 | ~30/day |
| `page_viewed` | 150,000 | ~50 users * 150 pages/day * 20 days |
| `search_performed` | 30,000 | ~50 users * 30 searches/day * 20 days |
| `login_success` | 1,500 | ~50 users * 1.5 logins/day * 20 days |
| `login_failed` | 300 | ~20% of login attempts fail |
| `client_created` | 200 | ~10/day |
| `client_updated` | 1,000 | ~50/day |
| `portal_accessed` | 5,000 | Customer/operator portal sessions |
| **TOTAL** | **~238,300** | |

### 10.2 Mixpanel Pricing (as of 2026)

| Plan | Monthly Events | Price (USD/month) | Notes |
|---|---|---|---|
| Free | Up to 20M | $0 | Core analytics, 5 saved reports |
| Growth | Up to 100M | From $28/mo | Unlimited reports, Group Analytics |
| Enterprise | Custom | Custom | SSO, data pipeline, advanced permissions |

**Recommendation:** At ~238K events/month, TruckSoft fits comfortably within the
**Free tier** (20M events/month limit). Even with 10x growth, we remain within the free
tier.

If the Minvy_QA and Minvy_Production projects already have volume from other products,
verify the combined monthly event count. At our projected volume, there is no incremental
cost concern.

### 10.3 Infrastructure Cost

| Item | Cost | Notes |
|---|---|---|
| Mixpanel SDK | $0 | Open source |
| Backend thread pool (2-4 threads) | Negligible | Runs on existing servers |
| Network (event payloads ~1KB each) | ~240 MB/month | Negligible |
| **Total additional infrastructure cost** | **$0** | |

---

## Appendix A: Event Reference Quick Sheet

| # | Event Name | Tier | Source | Phase |
|---|---|---|---|---|
| 1 | `cfdi_timbrado_attempted` | 1 | Backend | 1 |
| 2 | `cfdi_timbrado_success` | 1 | Backend | 1 |
| 3 | `cfdi_timbrado_failed` | 1 | Backend | 1 |
| 4 | `cfdi_cancelled` | 1 | Backend | 1 |
| 5 | `payment_received` | 1 | Backend | 1 |
| 6 | `payment_complement_created` | 1 | Backend | 1 |
| 7 | `credit_note_created` | 1 | Backend | 1 |
| 8 | `remision_created` | 1 | Backend | 1 |
| 9 | `remision_status_changed` | 1 | Backend | 1 |
| 10 | `cotizacion_created` | 1 | Backend | 1 |
| 11 | `cotizacion_accepted` | 1 | Backend | 1 |
| 12 | `manifiesto_created` | 2 | Backend | 2 |
| 13 | `carta_porte_generated` | 2 | Backend | 2 |
| 14 | `bulk_upload_completed` | 2 | Backend | 2 |
| 15 | `pdf_exported` | 2 | Backend | 2 |
| 16 | `exchange_rate_fetched` | 2 | Backend | 2 |
| 17 | `page_viewed` | 3 | Frontend | 3 |
| 18 | `search_performed` | 3 | Frontend | 3 |
| 19 | `login_success` | 3 | Frontend | 3 |
| 20 | `login_failed` | 3 | Backend | 3 |
| 21 | `client_created` | 3 | Backend | 3 |
| 22 | `client_updated` | 3 | Backend | 3 |
| 23 | `portal_accessed` | 3 | Frontend | 3 |

---

## Appendix B: Naming Conventions

- **Events:** `snake_case`, English. Pattern: `entity_action` (e.g., `remision_created`).
- **Properties:** `snake_case`, English. Use common suffixes: `_id`, `_ms`, `_count`, `_pct`, `_kg`.
- **User properties:** `snake_case` for custom; `$name`, `$email` for Mixpanel reserved.
- **No abbreviations** except: `id`, `rfc`, `cfdi`, `pct`, `ms`, `kg`, `km`.

---

## Appendix C: Testing Checklist

Before deploying each phase, verify in Mixpanel QA project (2984481):

- [ ] Event appears in Live View within 5 seconds of action
- [ ] All documented properties are present (no missing keys)
- [ ] Property types match documentation (number vs string)
- [ ] `user_id` is correctly set on every event
- [ ] `locacion_id` is present on backend events
- [ ] RFC values are masked (never sent in plain text)
- [ ] Email addresses are hashed or omitted from event properties
- [ ] User profile is updated after login (check People section)
- [ ] Events do not fire when analytics consent is not granted
- [ ] No duplicate events for a single user action
- [ ] Async tracking does not slow down API response times (< 5ms overhead)
