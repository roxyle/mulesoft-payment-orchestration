# Payment Orchestration System 

Enterprise-grade Payment Processing System built with MuleSoft, implementing Compensation Patterns and Idempotency Handling for distributed transactions.

## Business Problem

Need to process payments reliably across distributed systems, like order management, payment gateway, inventory? Network failures and retries can cause:

- Double charging customers
- Inconsistent order states
- Lost transactions

This system solves these problems through:

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

## Database Schema
**orders**: id, customer_id, amount, status, correlation_id, timestamps  
**payments**: id, order_id, transaction_id, status, amount  
**idempotency_log**: id, idempotency_key, order_id, response_payload, expires_at

### Database Setup
> SQL Run in phpMyAdmin
> CREATE DATABASE payment_system;
> Import schema from /docs/schema.sql

### Application Configuration
> Update src/main/resources/config.yaml with your DB credentials
> Import project in Anypoint Studio
> Run as Mule Application

API available at http://localhost:8081

### Project Status

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

### API Endpoints

Method   | Endpoint        | Purpose                     | Status    
---------|-----------------|-----------------------------|-----------
**POST** | `/orders`       | Create order with payment   | ✅ TESTED 
**GET**  | `/orders/{id}`  | Retrieve order details      | ✅ TESTED 
**GET**  | `/test-db`      | Database connectivity check | ✅ TESTED 
**POST** | `/mock-payment` | Mock payment gateway        | ✅ TESTED 

### POST /orders - Success Scenario
**Request:**
POST http://localhost:8081/orders
Headers:
"Content-Type: application/json"
"Idempotency-Key: unique-key-123"
Body:
{"customer_id":"CUST001","amount":99.99}

**Response:**
json
{
  "status": "success",
  "order_id": 1,
  "correlation_id": "...",
  "amount": 99.99,
  "customer_id": "CUST001",
  "timestamp": "2025-01-15T..."
}

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
**Response:**
json
{
  "status": "failed",
  "order_id": 2,
  "errorInfo": {
    "errorType": "APP:PAYMENT_GATEWAY_ERROR",
    "description": "Payment gateway timeout simulation"
  }
}
**Database Verification:**
- `orders` table: status=FAILED ✅ (compensation applied)
- `payments` table: NO record for this order_id ✅ (rollback)
- `idempotency_log` table: NO record ✅ (errors not cached)

---

### GET /orders/{id} - Retrieve Success Order
**Request:**
GET http://localhost:8081/orders/1

**Response:**
json
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

✅ Full order details including payment information

---

### GET /orders/{id} - Retrieve Failed Order
**Request:**
GET http://localhost:8081/orders/2

**Response:**
json
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

✅ Correctly shows failed order with no payment (compensation visible)

---

### GET /orders/{id} - Order Not Found
**Request:**
GET http://localhost:8081/orders/999

**Response:**
json
{
  "error": "Order not found",
  "order_id": null
}


Test Results:
✓ Success flow: order CONFIRMED + payment SUCCESS
✓ Failure flow: order FAILED + compensation applied
✓ Idempotency: cached response returned, no duplicate processing
✓ GET endpoint: returns full order details including payment status
✓ GET endpoint: handles failed orders (null payment) correctly
✓ GET endpoint: returns 404-like for non-existent orders

### License

MIT License

Author: Ylenia Rossi

Purpose: Portfolio project showcasing MuleSoft integration patterns
Status: 🚧 In Development

