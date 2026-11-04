# Design notes – Catalogue & Inventory (Eventi, Sessioni, Fasce di prezzo, Posti)

Note di design per M2. Decisioni principali:

- Un **evento** ha **N sessioni** (date), ognuna con il proprio inventario di posti.
- Il **layout dei posti è fisso**: ogni venue ha settori predefiniti (righe × posti per riga), caricati come dati iniziali.
- Le **fasce di prezzo** sono legate ai **settori** e valgono per tutte le sessioni dell'evento.
- La **Catalogue** possiede venue, settori, eventi, sessioni e fasce. **Inventory & Orders** genera e possiede i posti (`seat_state`) a partire dal messaggio `SessionPublished`.
- Gli importi sono **interi in centesimi** (`priceCents`), mai `double`.

> Outbox volutamente escluso da queste note (vedi sezione finale).

---

## 1. Schema dati – Catalogue

```
venue(
  id PK,
  name,                 -- "PALAOLIMPICO"
  city                  -- "TORINO"
)

sector(                 -- seat plan: layout fisso, dati iniziali
  id PK,
  venue_id FK → venue,
  code,                 -- "PLATEA"
  rows,                 -- 10
  seats_per_row,        -- 20   → Inventory genera A1..J20
  UNIQUE(venue_id, code)
)

event(
  id PK,
  title,
  venue_id FK → venue,
  organiser_id,         -- "sub" del JWT Keycloak, per autorizzare /sales
  created_at
)

session(
  id PK,
  event_id FK → event,
  starts_at,
  sales_open_at,
  status                -- SCHEDULED | CANCELLED
)

price_tier(
  id PK,
  event_id FK → event,  -- valida per tutte le sessioni dell'evento
  code,                 -- "GOLD"
  price_cents,          -- 8000
  currency,             -- "EUR"
  UNIQUE(event_id, code)
)

tier_sector(            -- quale fascia vale in quale settore
  event_id FK → event,
  sector_id FK → sector,
  price_tier_id FK → price_tier,
  PRIMARY KEY(event_id, sector_id)   -- un settore ha UNA sola fascia per evento
)

session_sales(          -- read model della dashboard, aggiornato solo da OrderPaid
  session_id FK → session,
  tier_code,
  seats_sold,
  revenue_cents,
  updated_at,
  PRIMARY KEY(session_id, tier_code)
)

processed_message(      -- idempotenza: OrderPaid duplicato = ignorato
  message_id PK,
  processed_at
)
```

---

## 2. `POST /events` – creazione evento con sessioni

### Request body

```json
{
  "title": "Rock Concert 2026",
  "venueId": 12,
  "priceTiers": [
    {
      "code": "GOLD",
      "priceCents": 8000,
      "currency": "EUR",
      "sectors": ["PLATEA"]
    },
    {
      "code": "SILVER",
      "priceCents": 5000,
      "currency": "EUR",
      "sectors": ["TRIBUNA_EST", "TRIBUNA_OVEST"]
    },
    {
      "code": "BRONZE",
      "priceCents": 3000,
      "currency": "EUR",
      "sectors": ["GALLERIA"]
    }
  ],
  "sessions": [
    {
      "startsAt": "2026-12-04T21:00:00Z",
      "salesOpenAt": "2026-11-10T10:00:00Z"
    },
    {
      "startsAt": "2026-12-05T21:00:00Z",
      "salesOpenAt": "2026-11-10T10:00:00Z"
    },
    {
      "startsAt": "2026-12-06T18:00:00Z",
      "salesOpenAt": "2026-11-10T10:00:00Z"
    }
  ]
}
```

### Response `201 Created`

Header: `Location: /api/v1/events/4711`

```json
{
  "id": 4711,
  "title": "Rock Concert 2026",
  "venue": { "id": 12, "name": "PALAOLIMPICO", "city": "TORINO" },
  "priceTiers": [
    { "code": "GOLD",   "priceCents": 8000, "currency": "EUR", "sectors": ["PLATEA"] },
    { "code": "SILVER", "priceCents": 5000, "currency": "EUR", "sectors": ["TRIBUNA_EST", "TRIBUNA_OVEST"] },
    { "code": "BRONZE", "priceCents": 3000, "currency": "EUR", "sectors": ["GALLERIA"] }
  ],
  "sessions": [
    { "id": 9001, "startsAt": "2026-12-04T21:00:00Z", "salesOpenAt": "2026-11-10T10:00:00Z", "status": "SCHEDULED" },
    { "id": 9002, "startsAt": "2026-12-05T21:00:00Z", "salesOpenAt": "2026-11-10T10:00:00Z", "status": "SCHEDULED" },
    { "id": 9003, "startsAt": "2026-12-06T18:00:00Z", "salesOpenAt": "2026-11-10T10:00:00Z", "status": "SCHEDULED" }
  ]
}
```

### Validazioni (`422 Unprocessable Content`)

- `venueId` deve esistere.
- Ogni settore della venue deve comparire in **esattamente una** fascia (nessun settore mancante, inesistente o ripetuto).
- `code` delle fasce unici, `priceCents > 0`, `currency` ISO 4217 (3 lettere).
- Almeno una sessione, nessun `startsAt` duplicato, date nel futuro, `salesOpenAt < startsAt`.

### Non presenti nel body

- `organiserId` → dal JWT.
- `city` → dalla venue.
- `status` → deciso dal server.

---

## 3. Cosa fa la Catalogue

Alla `POST /events`:

1. **Valida** il body e verifica il ruolo organiser.
2. **Salva** `event`, N `session`, `price_tier`, `tier_sector` (`organiser_id` dal JWT).
3. **Pubblica** un `SessionPublished` per ogni sessione sul broker.
4. Risponde **`201`**.

Poi:

- **Lettura catalogo**: `GET /events`, `GET /events/{id}` (evento, sessioni, fasce; **non** lo stato dei posti).
- **Dashboard**: consuma `OrderPaid` → aggiorna `session_sales`; espone `GET /sessions/{id}/sales` (solo all'organiser proprietario, altrimenti `403`).

---

## 4. Messaggi sul broker

### Pubblicato da Catalogue: `SessionPublished`

Uno per sessione. Routing key: `catalogue.session.published`.
I settori sono già "risolti" con fascia e prezzo: Inventory non deve conoscere `venue` né `tier_sector`.

```json
{
  "messageId": "6f1c2a9e-3b7d-4e2a-9c1f-8d5e4b3a2f10",
  "type": "SessionPublished",
  "occurredAt": "2026-11-04T15:40:23Z",
  "data": {
    "sessionId": 9001,
    "eventId": 4711,
    "startsAt": "2026-12-04T21:00:00Z",
    "salesOpenAt": "2026-11-10T10:00:00Z",
    "sectors": [
      { "code": "PLATEA",        "rows": 10, "seatsPerRow": 20, "tierCode": "GOLD",   "priceCents": 8000, "currency": "EUR" },
      { "code": "TRIBUNA_EST",   "rows": 8,  "seatsPerRow": 25, "tierCode": "SILVER", "priceCents": 5000, "currency": "EUR" },
      { "code": "TRIBUNA_OVEST", "rows": 8,  "seatsPerRow": 25, "tierCode": "SILVER", "priceCents": 5000, "currency": "EUR" },
      { "code": "GALLERIA",      "rows": 5,  "seatsPerRow": 30, "tierCode": "BRONZE", "priceCents": 3000, "currency": "EUR" }
    ]
  }
}
```

- `messageId`: usato dai consumer per scartare i duplicati (consegna at-least-once).

### Pubblicato da Catalogue (opzionale, per cancellazioni/rimborsi): `SessionCancelled`

```json
{
  "messageId": "...",
  "type": "SessionCancelled",
  "occurredAt": "...",
  "data": { "sessionId": 9001, "reason": "ORGANISER_CANCELLED" }
}
```

→ Inventory blocca nuovi hold; Payments/Ticketing gestiscono i rimborsi.

### Consumato da Catalogue: `OrderPaid` (pubblicato da Inventory)

Consumato da Catalogue **e** Ticketing.

```json
{
  "messageId": "...",
  "type": "OrderPaid",
  "occurredAt": "...",
  "data": {
    "orderId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
    "sessionId": 9001,
    "seat": "PLATEA-E14",
    "tierCode": "GOLD",
    "amountCents": 8000,
    "currency": "EUR",
    "customerId": "kc-sub-7f3a..."
  }
}
```

→ Catalogue incrementa `session_sales(sessionId, tierCode)`, scartando i `messageId` già visti.
(Con i rimborsi: consuma anche `OrderRefunded` per decrementare.)

---

## 5. Inventory & Orders – generazione dei posti

### Schema dati (parte posti)

```
session(                    -- copia locale minima, dal messaggio
  id PK,                    -- = sessionId della Catalogue
  event_id,
  starts_at,
  sales_open_at,
  status                    -- ON_SALE | CANCELLED
)

seat_state(
  session_id FK → session,
  label,                    -- "PLATEA-E14"
  sector,                   -- "PLATEA"
  row_label,                -- "E"
  seat_number,              -- 14
  tier_code,                -- "GOLD"
  price_cents,              -- 8000 (copia del prezzo al momento della pubblicazione)
  currency,                 -- "EUR"
  status,                   -- FREE | HELD | SOLD
  hold_id NULL,
  PRIMARY KEY(session_id, label)
)

processed_message(message_id PK, processed_at)
```

### Consumo di `SessionPublished` (tutto in **una transazione**)

1. **Deduplica**

   ```sql
   INSERT INTO processed_message(message_id, processed_at)
   VALUES ($1, now()) ON CONFLICT DO NOTHING;
   ```

   0 righe inserite → duplicato → ack e stop.

2. **Sessione**: `INSERT INTO session(...)`.

3. **Generazione posti**: per ogni settore, righe `A, B, C…`, numeri `1..seatsPerRow`.

   ```rust
   for s in &msg.sectors {
       for r in 0..s.rows {
           let row = row_letter(r);              // 0→"A", 25→"Z", 26→"AA"
           for n in 1..=s.seats_per_row {
               labels.push(format!("{}-{}{}", s.code, row, n));
               // + sector, row, n, tier_code, price_cents, currency
           }
       }
   }
   ```

4. **Bulk insert** con `status = 'FREE'`. Con SQLx:
   `INSERT ... SELECT * FROM UNNEST($1::text[], $2::text[], ...)` → una sola query invece di migliaia.

5. **Commit → ack** al broker.

### Gestione errori

| Caso | Comportamento |
| :--- | :--- |
| Errore prima del commit | Rollback, nessun ack → il broker riconsegna, si riparte puliti |
| Crash dopo commit, prima dell'ack | Riconsegna → `processed_message` lo riconosce come duplicato → nessun effetto |
| Messaggio sempre invalido (poison) | Dopo N tentativi → dead-letter queue, non blocca la coda |
| Posti duplicati | Impossibili: `PRIMARY KEY(session_id, label)` |

### Perché i posti vivono qui: niente overselling

FREE → HELD è un update condizionato, nello stesso DB degli ordini:

```sql
UPDATE seat_state SET status = 'HELD', hold_id = $3
WHERE session_id = $1 AND label = $2 AND status = 'FREE';
-- 0 righe aggiornate → 409 Conflict
```

100 richieste concorrenti su `PLATEA-E14` → solo una aggiorna la riga, le altre 99 ricevono `409`.
È esattamente ciò che verifica il test di concorrenza richiesto alle review.

---

## 6. Endpoint aggiornati (con le sessioni)

Già applicati in `openapi/`.

**Catalogue**

- `GET /venues`, `GET /venues/{venueId}` – venue e settori (servono all'organiser per i codici settore)
- `POST /events` – crea evento + sessioni + fasce
- `GET /events` – lista eventi (filtro `city`)
- `GET /events/{eventId}` – evento + sessioni + fasce
- `GET /sessions/{sessionId}/sales` – dashboard organiser, anche per fascia
- *(opzionale, non ancora in spec)* `POST /events/{eventId}/sessions` – aggiunge date

**Inventory & Orders**

- `GET /sessions/{sessionId}/seats` – mappa posti con stato, fascia e prezzo (ETag / 304)
- `POST /sessions/{sessionId}/holds` – FREE → HELD (`Idempotency-Key`)
- `DELETE /holds/{holdId}` – HELD → FREE
- `POST /orders` (`Idempotency-Key`), `GET /orders` (i miei ordini), `GET /orders/{orderId}`

**Ticketing**

- `GET /tickets` (i miei biglietti), `GET /tickets/{ticketId}`, `POST /tickets/{ticketId}/scan`

Negli schemi `eventId` → `sessionId` dove si parla di posti (`Hold`, `Order`, `Ticket`, `SessionSales`), e tutti gli importi sono `...Cents` interi.

---

## 7. Da fare / decisioni aperte

- **Outbox**: senza, salvataggio (passo 2) e pubblicazione (passo 3) non sono atomici. Se il servizio crolla tra i due, l'evento esiste ma Inventory non crea mai i posti → sessione invendibile. Da risolvere con il transactional outbox.
- **Modifica prezzi**: proposta → vietata dopo `salesOpenAt` (`409`). Alternativa: evento `PriceTiersChanged` applicato solo ai posti FREE.
- **ADR**: documentare layout fisso dei posti e fasce legate ai settori come semplificazioni consapevoli.
- **Sequence diagram** (`purchase-flw-sequence-diagram.jpg`): da ridisegnare con `sessionId`, mappa posti letta da Inventory, `Idempotency-Key` su hold e ordine.