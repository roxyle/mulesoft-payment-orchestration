# Payment Orchestration System



Enterprise-grade payment processing system built with MuleSoft, implementing compensation patterns and idempotency handling for distributed transactions.



## Business Problem



E-commerce platforms need to process payments reliably across distributed systems (order management, payment gateway, inventory). Network failures and retries can cause:

- Double charging customers

- Inconsistent order states

- Lost transactions



This system solves these problems through:

- **Compensation Logic**: Automatic rollback of partial transactions

- **Idempotency**: Safe retry handling preventing duplicates

- **Correlation IDs**: End-to-end traceability



## Architecture



Client Request

&nbsp;	↓

[API Layer - MuleSoft]

&nbsp;	↓

[Idempotency Check]

&nbsp;	↓

[Order Creation - MySQL]

&nbsp;	↓

[Payment Gateway Call]

&nbsp;	↓ (success)

[Order Confirmation]

&nbsp;	↓ (failure)

[Compensation: Mark Failed]



## Tech Stack



- **Integration**: MuleSoft Anypoint Platform 4.9.9 EE

- **Database**: MySQL 8.0 (via MAMP)

- **Patterns**: Saga/Compensation, Idempotency, Correlation IDs

- **API Design**: REST, JSON



## Database Schema



**orders**: id, customer_id, amount, status, correlation_id, timestamps  

**payments**: id, order_id, transaction_id, status, amount  

**idempotency_log**: id, idempotency_key, order_id, response_payload, expires_at



## Quick Start



### Prerequisites

- Anypoint Studio 7.x

- MySQL 8.0+ (MAMP)

- Java JDK 8 or 11



### Database Setup

``sql

-- Run in phpMyAdmin

CREATE DATABASE payment_system;

-- Import schema from /docs/schema.sql



### Application Configuration



Update src/main/resources/config.yaml with your DB credentials

Import project in Anypoint Studio

Run as Mule Application

API available at http://localhost:8081



### Implementation Status



&nbsp;- [x] Database schema

 - [x]Project structure

 - [x] Configuration files

 - [x] Database connector configuration -ok

 - [x] HTTP listener and request configs
  
 - [x] Basic flow test -ok

 - [ ] Main orchestration flow

 - [ ] Error handling and compensation

 - [ ] Idempotency implementation

 - [ ] Testing suite

### API Endpoints

POST /orders - Create new order with payment processing

GET /orders/{id} - Retrieve order details

### Learning Objectives

This project demonstrates:



Distributed transaction management without 2PC

Practical error handling in integration scenarios

Idempotency implementation for financial operations

Logging and observability best practices



### License

MIT License



Author: Ylenia Rossi

Purpose: Portfolio project showcasing MuleSoft integration patterns
Status: 🚧 In Development

