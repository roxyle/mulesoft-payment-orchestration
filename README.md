# Payment Orchestration System 

**Ita**: Sistema di Gestione Pagamenti Enterprise con Pattern Retributivi e Gestione Idempotenza per transazioni distribuite. 

**Eng**: Enterprise-grade Payment Processing System built with MuleSoft, implementing Compensation Patterns and Idempotency Handling for distributed transactions.


## Business Problem

**Ita**: Se hai bisogno di processare pagamenti in maniera affidabile attraverso sistemi distribuiti, come gestione degli ordini, gateway di pagamento, e/o inventario, i guasti alla rete e i tentativi di riconnessione potrebbero causare doppi addebiti ai clienti, stato degli ordini incoerente e/o perdita di transazioni
  
**Eng**:  If you need to process payments reliably across distributed systems, like order management, payment gateway and/or inventory, Network failures and retries can cause double charging on customers, inconsistent order states and/or lost transactions

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

--- TABLES ---

- ORDERS / Pratiche (all)

   - ID, Client, Amount, Status (PENDING/CONFIRMED/FAILED), Correlation ID
   - Always write first
   - Update at the end (success or fail)

- PAYMENTS (only success)

   - Order ID, Transaction ID, Amount, Method
   - Generated ONLY if gateway response is OK
   - If we see Order #5 without PAYMENT = failed

- IDEMPOTENCY_LOG

   - Key, Order ID, Response, Expires (24h)
   - Checked BEFORE initiating order
   - Write AFTER success (not on error)

## Tech Stack
- **Integration**: MuleSoft Anypoint Platform 4.9.9 EE
- **Database**: MySQL 8.0 (via MAMP)
- **Patterns**: Saga/Compensation, Idempotency, Correlation IDs
- **API Design**: REST, JSON

### Prerequisites
- Anypoint Studio 7.x
- MySQL 8.0+ (MAMP)
- Java JDK 8 or 11

API available at http://localhost:8081

## Setup Instructions

1. **Clone Repository**
```bash
   git clone https://github.com/roxyle/mulesoft-payment-orchestration.git
   cd mulesoft-payment-orchestration
```

2. **Database Setup**
   - Start MAMP MySQL
   - Open phpMyAdmin: http://localhost:8888/phpMyAdmin
   - Create database `payment_system`
   - Execute SQL scripts from `/database/schema.sql`

3. **Configuration**
   - Copy `config.yaml.example` to `config.yaml`
   - Update database credentials if needed (default: root/root)

4. **Run Application**
   - Import project in Anypoint Studio
   - Run As → Mule Application
   - Wait for DEPLOYED message

5. **Test**
```bash
   curl -X POST http://localhost:8081/orders \
     -H "Content-Type: application/json" \
     -H "Idempotency-Key: test-001" \
     -d '{"customer_id":"CUST001","amount":99.99}'
```

## API Endpoints

Method   | Endpoint        | Purpose                     | Status    
---------|-----------------|-----------------------------|-----------
**POST** | `/orders`       | Create order with payment   | ✅ TESTED 
**GET**  | `/orders/{id}`  | Retrieve order details      | ✅ TESTED 
**GET**  | `/test-db`      | Database connectivity check | ✅ TESTED 
**POST** | `/mock-payment` | Mock payment gateway        | ✅ TESTED 

## Troubleshooting

### Issue: "Cannot connect to database"
**Cause:** MAMP MySQL not running or wrong port

**Solution:**
1. Start MAMP MySQL
2. Check port in MAMP Preferences → Ports
3. Update `config.yaml` with correct port (8889 or 3306)
4. Restart application

### Issue: "Column 'order_id' cannot be null"
**Cause:** Idempotency log trying to save before order creation

**Solution:** Ensure `save-idempotency-log` is called INSIDE Try block AFTER order confirmation

### Issue: "HTTP 404 on /mock-payment"
**Cause:** Mock payment gateway flow not deployed

**Solution:**
1. Verify flow exists in XML (search for "mock-payment-gateway")
2. Check console logs for deployment errors
3. Restart application with Clean build

**"Mock gateway 404"**  
→ Search in log: `mock-payment-gateway successfully started`, if missing: Clean + Restart

### Issue: "Idempotency not working"
**Cause:** Header name mismatch or TTL expired

**Solution:**
1. Use exact header: `Idempotency-Key` (case-sensitive)
2. Check `idempotency_log` table: `SELECT * FROM idempotency_log;`
3. Verify expires_at > NOW()


## HOW TO - API!

### POST /orders - Success Scenario

Create new order with payment processing

**Request:**

POST http://localhost:8081/orders

Headers:

"Content-Type: application/json"
"Idempotency-Key: unique-key-123" (optional, auto-generated if missing)

Body:

{"customer_id":"CUST001","amount":99.99}

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
- Console log: "Idempotency hit: returning cached response"
- Database: Still only 1 order (no duplicate created)

---

### POST /orders - Failure Scenario (Compensation)
Mock gateway randomly fails (20% rate). When failure occurs:

**Response 500 (Failure - Compensation Applied):**

```json
{
  "status": "failed",
  "order_id": 2,
  "errorInfo": {
    "errorType": "APP:PAYMENT_GATEWAY_ERROR",
    "description": "Payment gateway timeout simulation"
  }
}
```

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
json
{
  "error": "Order not found",
  "order_id": null
}

<img width="623" height="777" alt="Screenshot 2025-10-15 201809" src="https://github.com/user-attachments/assets/9d6795d2-54a4-4b64-a67f-4d06d5fa68d1" />

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
**Expected:** full order details with payment information

### Test Results:
Success flow: order CONFIRMED + payment SUCCESS

Failure flow: order FAILED + compensation applied

Idempotency: cached response returned, no duplicate processing

GET endpoint: returns full order details including payment status

GET endpoint: handles failed orders (null payment) correctly

GET endpoint: returns 404-like for non-existent orders

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
  2. Query: SELECT response FROM idempotency_log WHERE key = ? AND expires_at > NOW()
  3. If found → return cached response (no reprocessing)
  4. If new → process normally, cache response after success


### 3. Correlation IDs
**Problem:** Distributed tracing across components  
**Solution:** UUID propagated in all calls/logs

Flow:
  1. Generate correlationId = uuid()
  2. Pass in HTTP headers (X-Correlation-ID)
  3. Store in database (orders.correlation_id)
  4. Include in all log messages
  5. Return in API responses

Benefit: track single request across all systems

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

 - [ ] 

### About me

Author: Ylenia Rossi 
Purpose: portfolio project showcasing MuleSoft integration patterns
Portfolio: https://portfolio-ylenia-ten.vercel.app/
Status: 🚧 In Development

