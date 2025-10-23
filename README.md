# Payment Orchestration System 

**Ita**: Sistema di Gestione Pagamenti Enterprise con Pattern Retributivi e Gestione Idempotenza per transazioni distribuite. 

**Eng**: Enterprise-grade Payment Processing System built with MuleSoft, implementing Compensation Patterns and Idempotency Handling for distributed transactions.


## Business Problem

**Ita**: Se hai bisogno di processare pagamenti in maniera affidabile attraverso sistemi distribuiti, come gestione degli ordini, gateway di pagamento, e/o inventario, i guasti alla rete e i tentativi di riconnessione potrebbero causare:

- Doppi addebiti ai clienti
- Stato degli ordini incoerente
- Perdita di transazioni
  
**Eng**:  If you need to process payments reliably across distributed systems, like order management, payment gateway and/or inventory, Network failures and retries can cause:

- Double charging customers
- Inconsistent order states
- Lost transactions

## Solution

**Ita**: 3 pattern testati
- **Compensation Logic**: Rollback automatico su transazioni parziali
- **Idempotency**: Gestione sicura di retry, prevenzione di duplicati
- **Correlation IDs**: Tracciamento end-to-end

**Eng**: 3 tested patterns
- **Compensation Logic**: Automatic rollback of partial transactions
- **Idempotency**: Safe retry handling, preventing duplicates
- **Correlation IDs**: End-to-end traceability

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
```{"customer_id":"CUST001","amount":99.99}```

**Response 200 Success:**

```json
{
  "status": "success",
  "order_id": 1,
  "correlation_id": "...",
  "amount": 99.99,
  "customer_id": "CUST001",
  "timestamp": "2025-01-15T..."
}```

**Database Verification:**
- `orders` table: 1 record, status=CONFIRMED ✅
- `payments` table: 1 record, status=SUCCESS ✅
- `idempotency_log` table: 1 record with 24h TTL ✅

---

### POST /orders - Idempotency Test
Repeat above request with **same Idempotency-Key**.

**Result:**
- ✅ Same response returned (order_id unchanged)
- ✅ Console log: "Idempotency hit: returning cached response"
- ✅ Database: Still only 1 order (no duplicate created)

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
}```

**Database Verification:**
- `orders` table: status=FAILED ✅ (compensation applied)
- `payments` table: NO record for this order_id ✅ (rollback)
- `idempotency_log` table: NO record ✅ (errors not cached)

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
}```

✅ Full order details including payment information

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
}```

✅ Correctly shows failed order with no payment (compensation visible)

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

## Testing

### Test 1: Successful Order Creation
```bash
curl -X POST http://localhost:8081/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-001" \
  -d '{"customer_id":"CUST001","amount":99.99}'
```
**Expected:** Order created, payment successful, status=CONFIRMED

**Verify in Database:**
```sql
SELECT * FROM orders WHERE id = 1;  -- status = CONFIRMED
SELECT * FROM payments WHERE order_id = 1;  -- status = SUCCESS
```

### Test 2: Idempotency
Repeat Test 1 with same Idempotency-Key
**Expected:** Same response, no duplicate order created

**Verify:**
```sql
SELECT COUNT(*) FROM orders WHERE customer_id = 'CUST001';  -- Should be 1
```

### Test 3: Payment Failure (Compensation)
Mock gateway has 20% random failure rate. Keep testing until failure occurs.

**Expected:** Order created with status=FAILED, no payment record

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
✓ Success flow: order CONFIRMED + payment SUCCESS

✓ Failure flow: order FAILED + compensation applied

✓ Idempotency: cached response returned, no duplicate processing

✓ GET endpoint: returns full order details including payment status

✓ GET endpoint: handles failed orders (null payment) correctly

✓ GET endpoint: returns 404-like for non-existent orders

## Implementation Patterns

### 1. Compensation Pattern (Saga)
**Problem:** Distributed transactions without 2PC  
**Solution:** Manual compensation on failure
```
Try:
  1. Create Order (status=PENDING)
  2. Call Payment Gateway
  3. Create Payment Record
  4. Update Order (status=CONFIRMED)

On Error:
  - COMPENSATE: Update Order (status=FAILED)
  - Prevents orphaned PENDING orders
  - No payment record created (logical rollback)
```

### 2. Idempotency Handling
**Problem:** Network retries causing duplicate charges  
**Solution:** Idempotency key caching (24h TTL)
```
Check Idempotency:
  1. Extract key from header (or generate UUID)
  2. Query: SELECT response FROM idempotency_log WHERE key = ? AND expires_at > NOW()
  3. If found → return cached response (no reprocessing)
  4. If new → process normally, cache response after success
```

### 3. Correlation IDs
**Problem:** Distributed tracing across components  
**Solution:** UUID propagated in all calls/logs
```
Flow:
  1. Generate correlationId = uuid()
  2. Pass in HTTP headers (X-Correlation-ID)
  3. Store in database (orders.correlation_id)
  4. Include in all log messages
  5. Return in API responses

Benefit: Track single request across all systems
```


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



### License

MIT License

Author: Ylenia Rossi

Purpose: Portfolio project showcasing MuleSoft integration patterns
Status: 🚧 In Development

