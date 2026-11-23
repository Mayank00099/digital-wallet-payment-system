# LedgerFlow — Digital Wallet & Payment Platform

A wallet and payment-management application built to explore the engineering problems behind reliable money movement: concurrent transfers, duplicate requests, balance integrity, transaction history, and auditable changes.

The application combines a Java 17 / Spring Boot 3 backend, MySQL 8 persistence, and a lightweight browser interface. Its focus is not just processing a payment request, but preserving a consistent account state when requests overlap or are retried.

> **Project status:** Educational/demo application. Wallet top-ups are simulated; this project is not connected to a real payment processor and must not be used to hold or transfer real funds.

## Contents

- [What it does](#what-it-does)
- [Engineering highlights](#engineering-highlights)
- [Application design](#application-design)
- [Main workflows](#main-workflows)
- [API overview](#api-overview)
- [Technology](#technology)
- [Run locally](#run-locally)
- [Run the tests](#run-the-tests)
- [Try the APIs with Postman](#try-the-apis-with-postman)
- [Security and configuration notes](#security-and-configuration-notes)
- [Project structure](#project-structure)
- [Project origin and license](#project-origin-and-license)

## What it does

LedgerFlow provides user wallets, peer-to-peer transfers, simulated top-ups, utility-bill settlement, transaction statements, and administrative controls.

### User capabilities

- Register an account and sign in using JWT authentication.
- View wallet details and current balance.
- Add demo funds through a simulated top-up flow.
- Transfer funds to another wallet.
- Browse and settle available utility bills.
- Review a paginated transaction statement and export it as CSV.

### Administrative capabilities

- View wallets across the platform.
- Freeze or unfreeze a wallet.
- Inspect audit events produced by database triggers.
- Review daily transaction summaries and top-user reports.

## Engineering highlights

The project makes several database and backend concepts visible in the implementation rather than treating the database as a passive storage layer.

| Concern | Approach used |
| --- | --- |
| Transaction consistency | Multi-step money movement runs within explicit transaction boundaries with commit / rollback handling. |
| Concurrent wallet updates | MySQL row-level locking with `SELECT ... FOR UPDATE`. |
| Deadlock reduction | Wallet rows are locked in a deterministic ID order during transfers. |
| Duplicate payment requests | An idempotency key is checked in the application and protected by a database `UNIQUE` constraint. |
| Transaction history | Transfers create paired `DEBIT` and `CREDIT` ledger entries. |
| Auditability | Database triggers record balance and status changes in an audit table. |
| Reporting | MySQL views provide statement and transaction-summary data. |
| Database access | JDBC and HikariCP are used so query execution and transaction handling remain explicit. |

### Why idempotency matters

A client may retry a request after a timeout even when the server completed the original operation. The transfer endpoint accepts an idempotency key so that repeating the same request does not create a second transfer. The database uniqueness constraint provides an additional safeguard.

### How concurrent transfers are handled

The transfer workflow obtains wallet locks in a consistent order, validates the wallets and available balance after acquiring the locks, updates both balances, writes the transaction and ledger records, and commits the operation. Consistent lock ordering helps avoid circular waits when two wallets transfer to each other at the same time.

## Application design

The backend follows a layered request flow:

```text
Browser / Postman
       |
       v
JWT authentication and role checks
       |
       v
Controllers  ------> HTTP requests and responses
       |
       v
Services     ------> Business rules and transaction boundaries
       |
       v
Transaction manager
       |
       v
DAO layer    ------> JDBC prepared statements
       |
       v
HikariCP connection pool
       |
       v
MySQL 8      ------> Tables, constraints, procedures, triggers and views
```

The browser interface is a single-page frontend made with HTML, CSS, and JavaScript. It is served by Spring Boot from `src/main/resources/static/`, so a separate frontend build step is not required.

## Main workflows

### 1. Account registration and sign-in

A new user account is created with a wallet and the standard user role. A successful login returns a JWT that is used for protected API requests.

### 2. Peer-to-peer transfer

A transfer request includes an idempotency key. The backend checks for a previously completed request, locks the relevant wallets, validates the transfer, updates balances, records the transaction, and appends debit and credit ledger entries.

### 3. Bill settlement

A signed-in user can view their bills and settle an outstanding bill using their wallet balance. The settlement is processed as a database operation rather than as a client-side balance change.

### 4. Administration and reporting

Administrative endpoints support wallet status controls, audit-log inspection, daily transaction summaries, and top-user reporting. These operations are protected by role-based authorization.

## API overview

The application exposes REST endpoints under `/api`. The Swagger UI is available at `http://localhost:8080/swagger-ui.html` after the application starts.

| Area | Method | Endpoint | Purpose |
| --- | --- | --- | --- |
| Authentication | `POST` | `/api/auth/register` | Create an account and wallet |
| Authentication | `POST` | `/api/auth/login` | Authenticate and receive a JWT |
| Wallet | `GET` | `/api/wallet/me` | Get the signed-in user's wallet |
| Wallet | `GET` | `/api/wallet/me/statement` | Retrieve a paginated ledger statement |
| Wallet | `POST` | `/api/wallet/topup` | Add simulated demo funds |
| Transfers | `POST` | `/api/transfer` | Make a peer-to-peer transfer; send `X-Idempotency-Key` |
| Transfers | `GET` | `/api/transfer/{txnId}` | Retrieve a transaction |
| Bills | `GET` | `/api/bills/my` | List the signed-in user's bills |
| Bills | `POST` | `/api/bills/{billId}/pay` | Settle an outstanding bill |
| Admin | `GET` | `/api/admin/wallets` | List wallets |
| Admin | `POST` | `/api/admin/wallets/{id}/freeze` | Freeze a wallet |
| Admin | `POST` | `/api/admin/wallets/{id}/unfreeze` | Unfreeze a wallet |
| Admin | `GET` | `/api/admin/audit` | View audit events |
| Admin | `GET` | `/api/admin/reports/daily` | View daily transaction metrics |
| Admin | `GET` | `/api/admin/reports/top-users` | View users ranked by outgoing volume |

Protected endpoints require a valid bearer token. Administrative endpoints additionally require the admin role.

## Technology

| Layer | Tools |
| --- | --- |
| Backend | Java 17, Spring Boot 3 |
| API | REST, JWT authentication |
| Data access | JDBC, HikariCP |
| Database | MySQL 8, InnoDB |
| Frontend | HTML5, CSS3, JavaScript (ES6+) |
| Visualization | Chart.js |
| API testing | Postman |
| Build | Maven |
| Automated tests | JUnit-based tests with H2 in MySQL compatibility mode |

## Run locally

### Prerequisites

- Java 17 or later
- Maven 3.8 or later
- MySQL 8.0 or later
- Git

### 1. Get the source

```bash
git clone <your-repository-url>
cd <your-repository-folder>
```

### 2. Create the database schema and load the SQL objects

Run the SQL scripts from the project root in the following order. Use a MySQL account with permission to create and modify the project database.

```bash
# Create the database and tables
mysql -u root -p < sql/01_schema.sql

# Add stored procedures
mysql -u root -p wallet_db < sql/03_procedures.sql

# Add audit triggers
mysql -u root -p wallet_db < sql/04_triggers.sql

# Add reporting views
mysql -u root -p wallet_db < sql/05_views.sql

# Load base users and roles
mysql -u root -p wallet_db < sql/02_seed.sql

# Optional: load the larger demonstration dataset
mysql -u root -p wallet_db < sql/06_seed_demo_data.sql
```

If your local MySQL setup uses a different database name or account, update the commands and application configuration consistently.

### 3. Configure the application

The application reads its database and JWT settings from environment variables. Set these in your shell before starting the service:

```bash
export DB_URL="jdbc:mysql://localhost:3306/wallet_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC"
export DB_USERNAME="your_mysql_username"
export DB_PASSWORD="your_mysql_password"
export JWT_SECRET="replace-with-a-long-random-secret"
```

Use your own local credentials and a strong random JWT secret. Do not commit real credentials, private keys, or production secrets to Git.

### 4. Build and start

```bash
mvn clean package -DskipTests
mvn spring-boot:run
```

Open the following URLs:

- Web application: `http://localhost:8080/`
- Swagger UI: `http://localhost:8080/swagger-ui.html`

## Run the tests

Run the complete test suite:

```bash
mvn test
```

The test configuration uses an in-memory H2 database in MySQL compatibility mode, so a separate MySQL server is not required for the automated tests.

Run the concurrency test separately:

```bash
mvn test -Dtest=ConcurrencyTest
```

The concurrency test launches 100 workers that attempt simultaneous transfers. It checks that the final account balances preserve the original total and that concurrent updates do not silently overwrite one another.

## Try the APIs with Postman

A Postman collection is included at:

```text
postman/wallet_collection.json
```

Import the collection into Postman to explore account registration, login, transfers, bill payments, statements, and administrative reporting. The collection captures the JWT after login for use by subsequent requests.

## Security and configuration notes

- This is a learning project, not a production-ready financial service.
- Top-ups are simulated and no real payment gateway is integrated.
- Use a unique, strong `JWT_SECRET` outside local development.
- Change or remove seeded demo credentials before exposing the application outside your machine.
- Do not publish personal data, secrets, or real financial information in screenshots, sample requests, or database dumps.
- A production payment system would require additional controls, operational monitoring, compliance review, threat modelling, and integration with a regulated payment provider.

## Project structure

```text
.
├── docs/       # Design and database documentation
├── postman/    # API request collection
├── sql/        # Schema, seed data, procedures, triggers and views
├── src/        # Spring Boot application and static frontend
├── pom.xml     # Maven build configuration
└── LICENSE     # Project license and required notices
```

## Project origin and license

This repository is an adapted version of the open-source project [`modimihir07/digital-wallet-payment-system`](https://github.com/modimihir07/digital-wallet-payment-system). The upstream project is distributed under the MIT License. The original license and copyright notice must remain included when redistributing the code. Any modifications made in this repository should be described accurately in commit messages and project notes.

See [`LICENSE`](LICENSE) for the license text. Please review the upstream license and repository history before publishing your version.

---

*Built as a hands-on exploration of backend APIs, relational database design, transaction safety, and reliable payment workflows.*
