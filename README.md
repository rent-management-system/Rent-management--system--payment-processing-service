# Rent Management - Payment Processing Service

[![Python Version](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Framework](https://img.shields.io/badge/framework-FastAPI-green.svg)](https://fastapi.tiangolo.com/)
[![Database](https://img.shields.io/badge/database-PostgreSQL-blue.svg)](https://www.postgresql.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A robust and scalable microservice built with FastAPI to handle payment processing for property listings. This service integrates with the Chapa payment gateway to securely manage transactions, verify payments, and communicate with other core components of the Rent Management System.

---

## Table of Contents

- [About The Project](#about-the-project)
- [Architectural Design](#architectural-design)
  - [Microservice Architecture](#microservice-architecture)
  - [Internal Layered Architecture](#internal-layered-architecture)
- [Technology Stack](#technology-stack)
- [External Service Integrations](#external-service-integrations)
  - [Chapa Payment Gateway](#chapa-payment-gateway)
  - [Inter-Service Communication](#inter-service-communication)
- [Key Features](#key-features)
- [Database Schema](#database-schema)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Local Installation](#local-installation)
  - [Running with Docker](#running-with-docker)
- [API Endpoint Documentation](#api-endpoint-documentation)
- [Running Tests](#running-tests)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

---

## About The Project

This service is a critical component of the **Rent Management System**. Its primary responsibility is to provide a centralized, secure, and reliable API for processing a fixed fee required for publishing property listings. By decoupling payment logic into a dedicated microservice, we can independently manage payment provider integrations, enhance security, and ensure high availability without impacting other parts of the system.

It is designed to be resilient and auditable, providing a clear trail for every transaction from initiation to verification and final confirmation.

---

## Architectural Design

This service is designed using a combination of high-level and internal architectural patterns to ensure separation of concerns, scalability, and maintainability.

### Microservice Architecture

This application is a **Microservice**. It is a small, autonomous service that works with other services to form the larger Rent Management platform. This approach provides several advantages:
- **Decoupling**: Payment logic is completely isolated, so it can be updated (e.g., adding a new payment provider) or scaled independently of services like user management or property search.
- **Security**: Isolating payment processing reduces the attack surface area for other services.
- **Resilience**: A failure in this service will not bring down the entire platform. Its asynchronous communication patterns and retry mechanisms further enhance system stability.

### Internal Layered Architecture

Internally, the service follows a classic **Layered (or N-Tier) Architecture** to separate responsibilities.

```mermaid
graph TD
    A[Routers Layer] --> B[Services Layer]
    B --> C[Data Access Layer]
    C --> D[(Database)]

    subgraph "Presentation Layer"
        A
    end
    subgraph "Business Logic Layer"
        B
    end
    subgraph "Data Layer"
        C
    end
```

-   **Presentation Layer (`app/routers`)**: Handles HTTP requests and responses. It defines API endpoints, validates incoming data using Pydantic schemas, and manages authentication (JWT & API Key). It delegates business logic to the Services Layer.
-   **Business Logic Layer (`app/services`)**: The core of the application. It contains the business rules, such as orchestrating the payment flow with the Chapa service, sending notifications, and confirming payments with the Property Listing service.
-   **Data Access Layer (`app/models`, `app/dependencies/database.py`)**: Responsible for all communication with the PostgreSQL database. It uses SQLAlchemy to define the schema and handle all CRUD operations.

---

## Technology Stack

- **Backend Framework:** [FastAPI](https://fastapi.tiangolo.com/)
- **Database:** [PostgreSQL](https://www.postgresql.org/)
- **ORM:** [SQLAlchemy](https://www.sqlalchemy.org/)
- **Data Validation:** [Pydantic](https://pydantic-docs.helpmanual.io/)
- **Authentication:** JWT Bearer Tokens & API Keys
- **Asynchronous Task Scheduling:** [APScheduler](https://apscheduler.readthedocs.io/)
- **Caching & Rate Limiting:** [Redis](https://redis.io/)
- **Containerization:** [Docker](https://www.docker.com/)
- **Testing:** [Pytest](https://docs.pytest.com/)

---

## External Service Integrations

### Chapa Payment Gateway

This service relies on the **Chapa Payment Gateway** for all transaction processing. The integration is encapsulated within the `app/services/chapa.py` module.

**Integration Flow:**
1.  **Initiation**: The `/payments/initiate` endpoint calls the Chapa API to generate a unique checkout URL for a fixed fee.
2.  **User Redirect**: The user is redirected to the Chapa checkout page to complete the payment.
3.  **Webhook Notification**: Upon completion, Chapa sends a secure webhook to the `/webhook/chapa` endpoint.
4.  **Verification**: The webhook handler first verifies the authenticity of the request using a HMAC-SHA256 signature. It then makes a server-to-server call back to the Chapa API to confirm the transaction status, ensuring a secure, double-verification process.
5.  **Database Update**: The payment record in the database is updated to `SUCCESS` or `FAILED`.

```mermaid
sequenceDiagram
    participant Client
    participant Payment Service
    participant Chapa API
    participant User

    Client->>Payment Service: POST /api/v1/payments/initiate
    Payment Service->>Chapa API: Initialize Transaction
    Chapa API-->>Payment Service: Return checkout_url
    Payment Service-->>Client: 202 Accepted with checkout_url
    Client->>User: Redirect to checkout_url
    User->>Chapa API: Completes Payment
    Chapa API-->>Payment Service: POST /api/v1/webhook/chapa (Signature Verified)
    Payment Service->>Chapa API: GET /transaction/verify/{tx_ref}
    Chapa API-->>Payment Service: Confirm transaction status
    Payment Service->>Payment Service: Update DB (SUCCESS/FAILED)
```

### Inter-Service Communication

The Payment Service communicates with other microservices to create a cohesive system:
-   **User Management Service**: Verifies JWTs to authenticate users and fetches user details (email, phone) for payment initiation and notifications.
-   **Property Listing Service**: Notifies this service upon successful payment so a property listing can be approved and published.
-   **Notification Service**: Sends emails or SMS messages to users regarding the status of their payments (initiated, successful, failed).

---

## Key Features

- **Secure Payment Processing**: Integrates with Chapa using secure signature verification for webhooks and a double-verification model.
- **Idempotent Initiation**: The `request_id` ensures that the same payment request sent multiple times does not result in duplicate transactions.
- **Dual Authentication**: Supports JWT for end-users (property owners) and `X-API-Key` for secure service-to-service communication.
- **Automated Timeouts**: A daily background job automatically marks pending payments as `FAILED` after 7 days, preventing stale records.
- **Resilient API Calls**: All external API calls use an automatic retry mechanism with exponential backoff to handle transient network issues.
- **Rate Limiting**: Protects the API from abuse by limiting the number of requests a client can make.
- **Health & Metrics**: Provides `/health` and `/metrics` endpoints for monitoring service status and transaction volumes.
- **Multi-language Notifications**: Supports sending notifications in English, Amharic, and Oromo.

---

## Database Schema

The service uses a single primary table, `payments`, to track all transaction details.

```mermaid
erDiagram
    payments {
        UUID id PK "Primary Key"
        UUID request_id UK "Idempotency key from client"
        UUID property_id "FK to property in Listing Service"
        UUID user_id "FK to user in User Management Service"
        DECIMAL amount "Fixed fee amount (e.g., 500.00)"
        payment_status status "PENDING, SUCCESS, or FAILED"
        TEXT chapa_tx_ref "Unique transaction ref from Chapa"
        TEXT failure_reason "Reason for failure, if any"
        TIMESTAMP created_at "Timestamp of creation"
        TIMESTAMP updated_at "Timestamp of last update"
        TIMESTAMP approved_at "Timestamp of successful payment"
    }
```

---

## Getting Started

### Prerequisites

- Python 3.10+ & `pip`
- Docker & Docker Compose
- A running PostgreSQL instance
- A running Redis instance

### Local Installation

1.  **Clone the repository:**
    ```sh
    git clone https://github.com/rent-management-system/Rent-management--system--payment-processing-service.git
    cd Rent-management--system--payment-processing-service
    ```

2.  **Set up the environment file:**
    Create a `.env` file in the root directory and add the following variables. Fill them in with your local configuration.
    ```env
    DATABASE_URL="postgresql+asyncpg://user:password@host:port/dbname"
    CHAPA_API_KEY="YOUR_CHAPA_API_KEY"
    CHAPA_SECRET_KEY="YOUR_CHAPA_SECRET_KEY"
    CHAPA_WEBHOOK_SECRET="YOUR_CHAPA_WEBHOOK_SECRET"
    JWT_SECRET="YOUR_JWT_SECRET"
    USER_MANAGEMENT_URL="http://localhost:8001"
    NOTIFICATION_SERVICE_URL="http://localhost:8003"
    PROPERTY_LISTING_SERVICE_URL="http://localhost:8002"
    ENCRYPTION_KEY="GENERATED_32_BYTE_URL_SAFE_BASE64_KEY"
    REDIS_URL="redis://localhost:6379"
    PAYMENT_SERVICE_API_KEY="A_STRONG_SECRET_API_KEY"
    FRONTEND_REDIRECT_URL="http://localhost:3000/payment-status"
    ```

3.  **Create a virtual environment and install dependencies:**
    ```sh
    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    ```

4.  **Apply the database schema:**
    Connect to your PostgreSQL database and run the contents of `sql/schema.sql`.

5.  **Run the application:**
    ```sh
    uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
    ```
    The API documentation will be available at `http://localhost:8000/docs`.

### Running with Docker

1.  **Ensure your `.env` file is created and configured.**

2.  **Build and run with Docker Compose (Recommended):**
    *(Note: A `docker-compose.yml` file would need to be created for this)*
    ```sh
    docker-compose up --build
    ```

3.  **Or, run the container directly:**
    ```sh
    docker build -t payment-processing-service .
    docker run -p 8000:8000 --env-file .env payment-processing-service
    ```

---

## API Endpoint Documentation

The API is versioned under `/api/v1`.

### `POST /payments/initiate`
Initiates a payment for a property listing fee. This is an idempotent endpoint.

- **Authentication:** Required (Owner JWT or Service API Key)
- **Body:**
  ```json
  {
    "request_id": "e9a1f2b3-c4d5-e6f7-a8b9-c0d1e2f3a4b5",
    "property_id": "a1b2c3d4-e5f6-a7b8-c9d0-e1f2a3b4c5d6",
    "user_id": "f0e1d2c3-b4a5-f6e7-d8c9-b0a1f2e3d4c5",
    "amount": 500.00
  }
  ```
- **Success Response (`202 Accepted`):**
  ```json
  {
    "id": "...",
    "status": "PENDING",
    "checkout_url": "https://checkout.chapa.co/checkout/payment/...",
    // ... other fields
  }
  ```

### `POST /webhook/chapa`
Endpoint for receiving payment status updates from Chapa. This endpoint should not be called directly by users.

- **Authentication:** Valid HMAC-SHA256 signature in `X-Chapa-Signature` header.
- **Success Response (`200 OK`):**
  ```json
  {
    "message": "Webhook processed successfully"
  }
  ```

### `GET /health`
Performs a health check on the service, including database and Chapa API connectivity.

- **Authentication:** None
- **Success Response (`200 OK`):**
  ```json
  {
    "status": "healthy",
    "db": "ok",
    "chapa_api": "ok"
  }
  ```
- **Error Response (`503 Service Unavailable`):** If any dependency is down.

---

## Running Tests

To run the test suite, execute the following command from the root directory:
```sh
pytest
```

---

## Contributing

Contributions are greatly appreciated. Please fork the repository and open a pull request with your proposed changes.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## License

Distributed under the MIT License.

---

## Contact

**Dagmawi Teferi**

- **Email:** [dagiteferi2011@gmail.com](mailto:dagiteferi2011@gmail.com)
- **Project Link:** [https://github.com/rent-management-system/Rent-management--system--payment-processing-service](https://github.com/rent-management-system/Rent-management--system--payment-processing-service)
