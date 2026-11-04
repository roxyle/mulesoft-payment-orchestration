# Payment Orchestration System 

**Ita**: Sistema di Gestione Pagamenti Enterprise con Pattern di Compensazione e Gestione dell'Idempotenza per transazioni distribuite.

**Eng**: Enterprise-grade Payment Processing System built with MuleSoft, implementing Compensation Patterns and Idempotency Handling for distributed transactions.

---
**Not a developer? Start here!**

**ITA**: Questo progetto dimostra la gestione affidabile di pagamenti in sistemi distribuiti,
implementando tre pattern architetturali avanzati: compensazione automatica su errore,
prevenzione di addebiti doppi e tracciabilità end-to-end.
La demo simula il comportamento reale del sistema: puoi creare ordini, vedere le tabelle
del database aggiornarsi in tempo reale, e provocare scenari di errore con rollback automatico.
Il codice sorgente completo è qui sotto per chi vuole approfondire l'implementazione tecnica.

Per vedere il sistema in azione senza installare nulla, usa la demo interattiva cliccando sul seguente link:
**[▶ Apri la Demo Interattiva](https://mulesoft-payment-ui-demo.vercel.app/)**

**ENG**: This project demonstrates reliable payment processing across distributed systems,
implementing three advanced architectural patterns: automatic failure compensation,
duplicate charge prevention, and end-to-end traceability.
The  demo simulates the real system behaviour: create orders, watch database tables update
in real time, and trigger failure scenarios with automatic rollback.
The full source code is below for those who want to explore the technical implementation.

To see the system in action without installing anything, use the interactive demo:
**[▶ Open Interactive Demo](https://mulesoft-payment-ui-demo.vercel.app/)**

---

## Business Problem

**Ita**: Se hai bisogno di processare pagamenti in maniera affidabile attraverso sistemi distribuiti, come gestione degli ordini, gateway di pagamento, e/o inventario, i guasti alla rete e i tentativi di riconnessione potrebbero causare doppi addebiti ai clienti, stato degli ordini incoerente e/o perdita di transazioni
  
**Eng**: If you need to process payments reliably across distributed systems, such as order management, payment gateways and inventory, network failures and retries can cause double charges for customers, inconsistent order states and lost transactions.

## Solution

**Ita**: 3 pattern testati
- **Compensation Logic**: Rollback automatico su transazioni parziali
- **Idempotency**: Gestione sicura di retry, prevenzione di duplicati
- **Correlation IDs**: Tracciamento end-to-end

**Eng**: 3 tested patterns
- **Compensation Logic**: Automatic rollback of partial transactions
- **Idempotency**: Safe retry handling, preventing duplicates
- **Correlation IDs**: End-to-end traceability

  <img width="568" height="1288" alt="Workflow" src="https://github.com/user-attachments/assets/7310fa23-901a-442f-990e-bf4982544241" />

## Database Tables

- **orders** (every request)
   - ID, customer, amount, status (PENDING/CONFIRMED/FAILED), correlation ID
   - Written first, updated at the end (success or failure)

- **payments** (successful payments only)
   - Order ID, transaction ID, amount, method
   - Created ONLY if the gateway response is OK
   - An order without a payment record is a failed order

- **idempotency_log**
   - Key, order ID, response, expiry (24h)
   - Checked BEFORE the order is created
   - Written AFTER success (not on error)

## Tech Stack
- **Integration**: Mule Runtime 4.9.9 EE (Anypoint Studio 7.21)
- **Database**: MySQL 5.7 (bundled with MAMP, tested on 5.7.24)
- **Patterns**: Saga/Compensation, Idempotency, Correlation IDs
- **API Design**: REST, JSON

### Prerequisites
- Anypoint Studio 7.21 or later
- MySQL 5.7 (MAMP), tested on 5.7.24
- Java JDK 17 (bundled with Anypoint Studio 7.21+)

API available at http://localhost:8081
All examples in this README use the default port 8081. If you changed `api.port` in `config.yaml`, replace 8081 with your value.

## Setup Instructions

1. **Clone Repository**
```bash
   git clone https://github.com/roxyle/mulesoft-payment-orchestration.git
   cd mulesoft-payment-orchestration
```

2. **Database Setup**
   - Start MAMP (Apache and MySQL)
   - Open phpMyAdmin: http://localhost:8888/phpMyAdmin (served by MAMP on its Apache port 8888, not on the API port)
   - Create the database `payment_system`
   - Select it, open the **Import** tab and import `database/schema.sql`

3. **Configuration**
   - Settings are in `src/main/resources/config.yaml`: database (host `localhost`, port `8889`, user and password `root`), API port `8081`
   - Edit this file if your MAMP uses different ports or credentials

4. **Run Application**
   - Import project in Anypoint Studio
   - Run As → Mule Application
   - Wait for DEPLOYED message

5. **Test**

First check the database connection: open `GET http://localhost:8081/test-db` in Postman or a browser. You should receive:

```json
{
  "status": "db connection successful",
  "result": [
    { "test": 1, "timestamp": "yyyy-mm-ddThh:mm:ss" }
  ]
}
```

Then create a test order:

```bash
curl -X POST http://localhost:8081/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: test-001" \
  -d '{"customer_id":"CUST001","amount":99.99}'
```


or open in Postman or a browser: GET http://localhost:8081/test-db


   you should receive:

```
   {
    "status": "db connection successful",
    "result": [
        {
            "test": 1,
            "timestamp": "yyyy-mm-ddThh:mm:ss"
        }
    ]
}
```
## Authentication

This is a local demo: the API does not require authentication and runs over plain HTTP (no TLS).
The only request headers are `Content-Type: application/json` and the optional `Idempotency-Key`.

## API Endpoints

Method   | Endpoint        | Purpose                     | Status    
---------|-----------------|-----------------------------|-----------
**POST** | `/orders`       | Create order with payment   | ✅ TESTED 
**GET**  | `/orders/{id}`  | Retrieve order details      | ✅ TESTED 
**GET**  | `/test-db`      | Database connectivity check | ✅ TESTED 
**POST** | `/mock-payment` | Mock payment gateway        | ✅ TESTED 

## Error responses

**Payment failure (500)**, returned by `POST /orders`. The order is created, then marked `FAILED` (compensation).

| Field | Type | Description |
|---|---|---|
| `status` | string | Always `failed` |
| `order_id` | integer or null | ID of the order marked `FAILED`; `null` if the error occurs before the order is created |
| `correlation_id` | string or null | UUID used to trace the request across logs |
| `errorInfo.errorType` | string | Mule error identifier, for example `INTERNAL_SERVER_ERROR` when the payment gateway returns 500; `UNKNOWN` if not available |
| `errorInfo.description` | string | Error message from the failed step |
| `timestamp` | string | ISO 8601 date and time with offset |

**Order not found (404)**, returned by `GET /orders/{id}`:

| Field | Type | Description |
|---|---|---|
| `error` | string | Always `Order not found` |
| `order_id` | string | The ID requested in the URL, returned as text |

## Troubleshooting

### Issue: "Cannot connect to database"
**Cause:** MAMP MySQL not running or wrong port

**Solution:**
1. Start MAMP MySQL
2. Check port in MAMP Preferences → Ports
3. Update `config.yaml` with correct port (8889 or 3306)
4. Restart application

**Solution:** Ensure `save-idempotency-log` is called INSIDE Try block AFTER order confirmation

### Issue: "HTTP 404 on /mock-payment"
**Cause:** Mock payment gateway flow not deployed

**Solution:**
1. Verify flow exists in XML (search for "mock-payment-gateway")
2. Check console logs for deployment errors
3. Restart application with Clean build

4. In the console, look for `Message source 'listener' on flow 'mock-payment-gateway' successfully started`. If it is missing: Clean + Restart

### Issue: "Idempotency not working"
**Cause:** Header name mismatch or TTL expired

**Solution:**
1. Use the header name `Idempotency-Key`
2. Check `idempotency_log` table: `SELECT * FROM idempotency_log;`
3. Verify expires_at > NOW()


## HOW TO - API!

### POST /orders - Success Scenario

Create new order with payment processing

**Request:**

```
POST http://localhost:8081/orders
Content-Type: application/json
Idempotency-Key: unique-key-123   (optional, auto-generated if missing)

{"customer_id":"CUST001","amount":99.99}
```

**Response 200 Success:**

```json
{
  "status": "success",
  "order_id": 1,
  "correlation_id": "...",
  "amount": 99.99,
  "customer_id": "CUST001",
  "timestamp": "2025-01-15T..."
}
```
<img width="1449" height="953" alt="arcpost" src="https://github.com/user-attachments/assets/11c3b412-676f-4487-bf1a-5417c5a7056f" />

**Database Verification:**
- `orders` table: 1 record, status=CONFIRMED
- `payments` table: 1 record, status=SUCCESS
- `idempotency_log` table: 1 record with 24h TTL

---

### POST /orders - Idempotency Test
Repeat above request with **same Idempotency-Key**.

**Result:**
- Same response returned (order_id unchanged)
- Console log: "idempotency hit: returning cached response for key <your-key>"
- Database: Still only 1 order (no duplicate created)

---

### POST /orders - Failure Scenario (Compensation)
Mock gateway randomly fails (20% rate). When failure occurs:

**Response 500 (Failure - Compensation Applied):**

```json
{
  "status": "failed",
  "order_id": 8,
  "correlation_id": "00d17680-3c93-46e3-8f2d-45b1942f6d9e",
  "errorInfo": {
    "errorType": "INTERNAL_SERVER_ERROR",
    "description": "HTTP POST on resource 'http://localhost:8081/mock-payment' failed: internal server error (500)."
  },
  "timestamp": "2026-11-04T12:55:08.9270546+01:00"
}
```
The mock gateway fails at random in about 1 request out of 5 (`randomInt(100) < 80` in the `mock-payment-gateway` flow). Repeat the request until you get a 500. Send each attempt without `Idempotency-Key`, or with a new one: a reused key returns the saved response instead of a new attempt.

**Database Verification:**
- `orders` table: status=FAILED (compensation applied)
- `payments` table: NO record for this order_id (rollback)
- `idempotency_log` table: NO record (errors not cached)



---

### GET /orders/{id} - Retrieve Success Order
**Request:**
GET http://localhost:8081/orders/1


**Response:**

```json
{
  "order_id": 1,
  "customer_id": "CUST001",
  "amount": 99.99,
  "status": "CONFIRMED",
  "correlation_id": "abc-123...",
  "transaction_id": "xyz-789...",
  "payment_status": "SUCCESS",
  "created_at": "2025-01-15T...",
  "updated_at": "2025-01-15T..."
}
```

Full order details including payment information

<img width="716" height="951" alt="Screenshot 2025-10-15 201757" src="https://github.com/user-attachments/assets/1584d828-e99b-4e46-82f1-6582b8aca2f5" />

---

### GET /orders/{id} - Retrieve Failed Order
**Request:**
GET http://localhost:8081/orders/2

**Response:**

```json
{
  "order_id": 2,
  "customer_id": "CUST001",
  "amount": 99.99,
  "status": "FAILED",
  "correlation_id": "def-456...",
  "transaction_id": null,
  "payment_status": null,
  "created_at": "2025-01-15T...",
  "updated_at": "2025-01-15T..."
}
```

Correctly shows failed order with no payment (compensation visible)

<img width="750" height="953" alt="Screenshot 2025-10-15 201746" src="https://github.com/user-attachments/assets/291f20f4-bac3-4caa-b320-cad8b1a2bf30" />

---

### GET /orders/{id} - Order Not Found
**Request:**
GET http://localhost:8081/orders/999

**Response 404:**
```json
{ "error": "Order not found", "order_id": "999" }
```

<img width="782" height="850" alt="image" src="https://github.com/user-attachments/assets/5b27419a-90db-4eac-820e-96ae0be989ea" />


---

## Anypoint Studio Flows
<img width="1149" height="400" alt="image" src="https://github.com/user-attachments/assets/79dc0fd1-684b-4c0b-b5c2-8ebbbcdac202" />

<img width="439" height="580" alt="image" src="https://github.com/user-attachments/assets/af46cc87-c13d-4e4c-b6f6-fd72a270b97d" />

<img width="544" height="468" alt="image" src="https://github.com/user-attachments/assets/e6a0e8e2-ebe1-4825-b9e0-d0eac8a6dff8" />

<img width="1246" height="369" alt="image" src="https://github.com/user-attachments/assets/ffa9040a-1925-4d8b-a540-a76c3cf6302c" />

<img width="362" height="393" alt="image" src="https://github.com/user-attachments/assets/0745138d-7890-4001-81b5-a12f61c3c9d3" />


## Testing

### Test 1: Successful Order Creation
```bash
curl -X POST http://localhost:8081/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-001" \
  -d '{"customer_id":"CUST001","amount":99.99}'
```
**Expected:** order created, payment successful, status=CONFIRMED

**Verify in Database:**
```sql
SELECT * FROM orders WHERE id = 1;  -- status = CONFIRMED
SELECT * FROM payments WHERE order_id = 1;  -- status = SUCCESS
```

### Test 2: Idempotency
Repeat Test 1 with same Idempotency-Key
**Expected:** same response, no duplicate order created

**Verify:**
```sql
SELECT COUNT(*) FROM orders WHERE customer_id = 'CUST001';  -- Should be 1
```

### Test 3: Payment Failure (Compensation)
Mock gateway has 20% random failure rate. Keep testing until failure occurs.

**Expected:** order created with status=FAILED, no payment record

**Verify:**
```sql
SELECT * FROM orders WHERE status = 'FAILED';  -- Should exist
SELECT * FROM payments WHERE order_id IN (
    SELECT id FROM orders WHERE status = 'FAILED'
);  -- Should be empty (compensation worked)
```

### Test 4: Retrieve Order
```bash
curl http://localhost:8081/orders/1
```

**Expected:** Full order details with payment information

### Test Results:
Success flow: order CONFIRMED + payment SUCCESS

Failure flow: order FAILED + compensation applied

Idempotency: cached response returned, no duplicate processing

GET endpoint: returns full order details including payment status

GET endpoint: handles failed orders (null payment) correctly

GET endpoint: returns 404 for non-existent orders

## Implementation Patterns

### 1. Compensation Pattern (Saga)
**Problem:** Distributed transactions without 2PC  
**Solution:** Manual compensation on failure

Try:
  1. Create Order (status=PENDING)
  2. Call Payment Gateway
  3. Create Payment Record
  4. Update Order (status=CONFIRMED)

On Error:
  - COMPENSATE: Update Order (status=FAILED)
  - Prevents orphaned PENDING orders
  - No payment record created (logical rollback)


### 2. Idempotency Handling
**Problem:** Network retries causing duplicate charges  
**Solution:** Idempotency key caching (24h TTL)

Check Idempotency:
  1. Extract key from header (or generate UUID)
  2. Query: SELECT order_id, response_payload FROM idempotency_log WHERE idempotency_key = ? AND expires_at > NOW()
  3. If found → return cached response (no reprocessing)
  4. If new → process normally, cache response after success


### 3. Correlation IDs
**Problem:** Distributed tracing across components  
**Solution:** UUID propagated in all calls/logs

Flow:
  1. Generate correlationId = uuid()
  2. Send it to the payment gateway in the X-Correlation-ID header
  3. Store it in the database (orders.correlation_id)
  4. Return it in API responses

Benefit: link a single request across the API response, the database and the gateway call

## Project Status

 - [x] Database schema

 - [x] Project structure

 - [x] Configuration files

 - [x] Database connector configuration -Test Ok

 - [x] HTTP listener and request configs 

 - [x] Basic flow -Test Ok

 - [x] Main orchestration flow -Test Ok

 - [x] Error handling and compensation -Test Ok

 - [x] Idempotency implementation -Test Ok

 - [x] GET /orders/{id} endpoint -Test Ok

 - [x] Final Testing -Test Ok

 - [x] Documentation finalization

 - [x] Working on a simulated interface

 - [x] Deployed interface

### About me

Author: Ylenia Rossi 
Purpose: portfolio project showcasing MuleSoft integration patterns
Portfolio: https://portfolio-ylenia-ten.vercel.app/
Status: complete. The UI demo is deployed on Vercel; the Mule API runs locally.

