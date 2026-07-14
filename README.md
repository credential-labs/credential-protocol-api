# Credential Protocol — API Server

A robust, production-grade Express.js and TypeScript-based API server for the Credential Protocol decentralized professional credential verification platform. This repository contains the backend REST API and WebSocket services that orchestrate interactions between the frontend applications and Stellar Soroban smart contracts, providing credential issuance, verification, attestation, and quorum slice management endpoints.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Development](#development)
- [API Endpoints](#api-endpoints)
- [WebSocket Events](#websocket-events)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [Error Handling](#error-handling)
- [Database](#database)
- [Testing](#testing)
- [Deployment](#deployment)
- [Performance](#performance)
- [Security](#security)
- [Monitoring](#monitoring)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

The Credential Protocol API Server is the central orchestration layer that:

1. **Manages HTTP REST endpoints** for credential lifecycle operations
2. **Facilitates smart contract interactions** with Stellar Soroban contracts
3. **Provides real-time updates** via WebSocket connections
4. **Handles authentication** and authorization for users and issuers
5. **Implements rate limiting** and abuse prevention
6. **Caches frequently accessed data** for performance
7. **Logs and monitors** all operations for security and debugging
8. **Validates credentials** and attestation requests

This split repository contains only the API backend code, separated from frontend and smart contract layers for independent scaling and deployment. The API is designed to be stateless for horizontal scaling across multiple server instances.

## 🏗️ Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend Applications                     │
│              (Vue.js + React Dashboard)                      │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTP/REST & WebSocket
┌────────────────────▼────────────────────────────────────────┐
│                   API Server (Node.js)                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Express.js Routes & Middleware                      │  │
│  │  • REST API Endpoints                                │  │
│  │  • WebSocket Server                                  │  │
│  │  • Authentication & Authorization                    │  │
│  │  • Rate Limiting & Throttling                        │  │
│  │  • Request Validation & Logging                      │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Business Logic Services                             │  │
│  │  • Credential Service                                │  │
│  │  • Attestation Service                               │  │
│  │  • Quorum Slice Service                              │  │
│  │  • Stellar/Contract Service                          │  │
│  │  • Analytics Service                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Infrastructure Layer                                │  │
│  │  • Database (PostgreSQL/MongoDB)                     │  │
│  │  • Redis Cache                                       │  │
│  │  • File Storage                                      │  │
│  │  • Logging & Monitoring                              │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────────┘
                     │ JSON RPC
┌────────────────────▼────────────────────────────────────────┐
│          Stellar Soroban Smart Contracts                     │
│  • CredentialProtocol Contract                              │
│  • SBT Registry Contract                                    │
│  • ZK Verifier Contract (Stub)                              │
└─────────────────────────────────────────────────────────────┘
```

### Request Flow

```
User Request (Frontend)
    ↓
Express Router (Route matching)
    ↓
Middleware Stack (Auth, Validation, Logging)
    ↓
Controller (Request handling)
    ↓
Service Layer (Business logic)
    ↓
Data/Stellar Layer (DB/Contract calls)
    ↓
Response (JSON)
    ↓
Frontend (State update & UI render)
```

## ✨ Features

### Core Features

- **RESTful API**: Clean, standards-compliant HTTP endpoints
- **Credential Management**: Issue, retrieve, revoke, and verify credentials
- **Attestation Workflow**: Multi-step attestation by designated authorities
- **Quorum Slice Management**: Create and manage trust networks
- **SBT Operations**: Mint, burn, and transfer Soulbound Tokens
- **Credential Verification**: Cross-check credentials against smart contracts
- **Batch Operations**: Bulk issue and verify credentials
- **Real-time Updates**: WebSocket events for credential state changes

### Advanced Features

- **Smart Contract Integration**: Direct interaction with Stellar Soroban
- **WebSocket Support**: Real-time credential and attestation updates
- **Multi-tenant Architecture**: Support for multiple issuers/authorities
- **Analytics & Reporting**: Track credential metrics and trends
- **Audit Logging**: Complete audit trail of all operations
- **Rate Limiting**: Protect against abuse and DoS attacks
- **Caching Strategy**: Redis integration for performance
- **Error Handling**: Comprehensive error codes and messages
- **Request Validation**: Input validation and sanitization
- **CORS Support**: Cross-origin resource sharing for frontend apps

## 🔧 Prerequisites

### System Requirements

- **Node.js**: 18.x LTS or higher
- **npm**: 9.x or higher (or yarn/pnpm)
- **Git**: 2.x or higher
- **Docker** (optional): For containerized deployment

### Runtime Dependencies

- **PostgreSQL** or **MongoDB**: Primary database
- **Redis**: For caching and session management
- **Stellar Test Account**: For development/testing

### Development Tools

- **TypeScript**: 5.x
- **ESLint**: Code quality
- **Prettier**: Code formatting
- **Vitest** or **Jest**: Testing framework
- **Docker Compose** (optional): Local development environment

## 📥 Installation

### 1. Clone Repository

```bash
git clone https://github.com/credential-labs/credential-protocol-api.git
cd credential-protocol-api
```

### 2. Install Dependencies

```bash
npm install
# or
yarn install
```

### 3. Environment Setup

Create `.env.local` from template:

```bash
cp .env.example .env.local
```

Configure environment variables:

```env
# Server Configuration
NODE_ENV=development
PORT=3000
HOST=localhost
API_VERSION=v1
LOG_LEVEL=debug

# Database Configuration
DATABASE_URL=postgresql://user:password@localhost:5432/credential_db
# or for MongoDB:
MONGODB_URL=mongodb://localhost:27017/credential_db
DB_POOL_SIZE=20
DB_MAX_CONNECTIONS=50

# Redis Configuration
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=
REDIS_DB=0

# Stellar Configuration
STELLAR_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_ACCOUNT_SECRET=SBXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Smart Contract Addresses
CONTRACT_CREDENTIAL_PROTOCOL=CBXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
CONTRACT_SBT_REGISTRY=CBXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
CONTRACT_ZK_VERIFIER=CBXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# API Keys & Security
JWT_SECRET=your-super-secret-jwt-key-min-32-chars
JWT_EXPIRY=24h
ISSUER_API_KEY=sk_live_XXXXXXXXXXXXXXXXXXXX
ADMIN_API_KEY=sk_admin_XXXXXXXXXXXXXXXXXXXX

# CORS Configuration
CORS_ORIGIN=http://localhost:5173,http://localhost:5174
CORS_CREDENTIALS=true

# Rate Limiting
RATE_LIMIT_WINDOW=900000
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_MAX_AUTH_REQUESTS=10

# Logging
LOG_FORMAT=json
LOG_OUTPUT=stdout,file
LOG_FILE_PATH=./logs

# Feature Flags
FEATURE_ZK_VERIFICATION=false
FEATURE_BATCH_CREDENTIALS=true
FEATURE_ANALYTICS=true
FEATURE_WEBHOOK_EVENTS=true

# External Services
SENTRY_DSN=
DATADOG_API_KEY=
SLACK_WEBHOOK_URL=
```

### 4. Database Setup

Initialize database:

```bash
# PostgreSQL
npm run db:migrate
npm run db:seed

# MongoDB
npm run db:init
```

### 5. Build Project

```bash
npm run build
```

## ⚙️ Configuration

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

### Express Middleware Stack

```typescript
// Order matters!
app.use(helmet())                    // Security headers
app.use(cors(corsOptions))           // CORS handling
app.use(express.json({ limit }))     // JSON parser
app.use(requestLogger)               // Request logging
app.use(rateLimiter)                 // Rate limiting
app.use(authentication)              // Auth middleware
app.use(requestValidator)            // Input validation
app.use(errorHandler)                // Error handling
```

## 🚀 Development

### Start Development Server

```bash
npm run dev
```

Server runs on `http://localhost:3000`

### Development with Hot Reload

```bash
npm run dev:watch
```

Uses `nodemon` for automatic restart on file changes.

### Development with Docker

```bash
docker-compose -f docker-compose.dev.yml up
```

Includes PostgreSQL, Redis, and API server containers.

## 📡 API Endpoints

### Authentication

```http
POST /api/v1/auth/login
POST /api/v1/auth/register
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
GET  /api/v1/auth/me
```

### Credentials

```http
POST   /api/v1/credentials              # Create credential
GET    /api/v1/credentials              # List credentials
GET    /api/v1/credentials/:id          # Get credential
PATCH  /api/v1/credentials/:id          # Update credential
DELETE /api/v1/credentials/:id          # Revoke credential
POST   /api/v1/credentials/batch        # Batch issue
GET    /api/v1/credentials/:id/verify   # Verify credential
```

### Attestations

```http
POST   /api/v1/attestations             # Create attestation
GET    /api/v1/attestations             # List attestations
GET    /api/v1/attestations/:id         # Get attestation
PATCH  /api/v1/attestations/:id/approve # Approve attestation
PATCH  /api/v1/attestations/:id/reject  # Reject attestation
GET    /api/v1/attestations/pending     # List pending
```

### Quorum Slices

```http
POST   /api/v1/slices                   # Create slice
GET    /api/v1/slices                   # List slices
GET    /api/v1/slices/:id               # Get slice
PATCH  /api/v1/slices/:id               # Update slice
DELETE /api/v1/slices/:id               # Delete slice
POST   /api/v1/slices/:id/attestors     # Add attestor
DELETE /api/v1/slices/:id/attestors/:aid # Remove attestor
```

### SBT Management

```http
POST /api/v1/sbt/mint                   # Mint SBT
POST /api/v1/sbt/burn                   # Burn SBT
GET  /api/v1/sbt/balance/:owner         # Check balance
```

### Analytics

```http
GET /api/v1/analytics/credentials      # Credential metrics
GET /api/v1/analytics/attestations     # Attestation metrics
GET /api/v1/analytics/users            # User metrics
GET /api/v1/analytics/contracts        # Contract metrics
```

## 🔗 WebSocket Events

Connect to `wss://api.credential-labs.org/ws` or `ws://localhost:3000/ws`

### Subscription Events

```javascript
// Subscribe to credential updates
socket.emit('subscribe:credentials', { userId: 'user-id' })

// Subscribe to attestation notifications
socket.emit('subscribe:attestations', { issuerId: 'issuer-id' })

// Listen for credential created
socket.on('credential:created', (credential) => {})

// Listen for attestation status change
socket.on('attestation:status', (attestation) => {})

// Listen for error events
socket.on('error', (error) => {})
```

## 🔐 Authentication

### JWT Token Flow

```
1. User logs in with email/password or Stellar account
2. API generates JWT token (18 hour expiry)
3. Client stores token in localStorage/sessionStorage
4. Client includes token in Authorization header
5. API validates token on each request
6. Token can be refreshed before expiry
```

### API Key Authentication

For service-to-service communication:

```bash
curl -H "X-API-Key: sk_live_XXXX" https://api.example.com/credentials
```

### Stellar Account Authentication

Optional direct authentication with Stellar account:

```bash
POST /api/v1/auth/stellar
Content-Type: application/json

{
  "publicKey": "GBXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "signature": "SIGNATURE_OF_CHALLENGE"
}
```

## 🛡️ Rate Limiting

### Rate Limit Tiers

```
Public Endpoints:    100 requests/15 minutes
Authenticated:       500 requests/15 minutes
Issuer API Key:      1000 requests/15 minutes
Admin API Key:       5000 requests/15 minutes
```

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1634567890
```

### Rate Limit Response

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{
  "error": "Too many requests",
  "retryAfter": 45
}
```

## ⚠️ Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "CREDENTIAL_NOT_FOUND",
    "message": "Credential with ID not found",
    "statusCode": 404,
    "timestamp": "2024-07-14T10:30:00Z",
    "path": "/api/v1/credentials/123",
    "requestId": "req_abc123def456"
  }
}
```

### Common Error Codes

```
400  Bad Request           - Invalid input
401  Unauthorized          - Missing/invalid auth
403  Forbidden             - Insufficient permissions
404  Not Found             - Resource not found
409  Conflict              - State conflict
422  Unprocessable Entity  - Validation failed
429  Too Many Requests     - Rate limited
500  Internal Server Error - Server error
503  Service Unavailable   - Maintenance/overload
```

## 💾 Database

### Schema Overview

```sql
-- Users/Accounts
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR UNIQUE,
  stellar_account VARCHAR UNIQUE,
  created_at TIMESTAMP
);

-- Credentials
CREATE TABLE credentials (
  id UUID PRIMARY KEY,
  issuer_id UUID REFERENCES users,
  subject_id UUID REFERENCES users,
  credential_type VARCHAR,
  metadata JSONB,
  issued_at TIMESTAMP,
  revoked_at TIMESTAMP
);

-- Attestations
CREATE TABLE attestations (
  id UUID PRIMARY KEY,
  credential_id UUID REFERENCES credentials,
  attestor_id UUID REFERENCES users,
  status ENUM('pending', 'approved', 'rejected'),
  created_at TIMESTAMP
);

-- Quorum Slices
CREATE TABLE quorum_slices (
  id UUID PRIMARY KEY,
  owner_id UUID REFERENCES users,
  attestors JSONB,
  threshold INTEGER,
  created_at TIMESTAMP
);
```

## 🧪 Testing

### Run All Tests

```bash
npm run test
```

### Run Tests in Watch Mode

```bash
npm run test:watch
```

### Coverage Report

```bash
npm run test:coverage
```

### Test Structure

```typescript
describe('Credentials API', () => {
  it('should create a new credential', async () => {
    const response = await request(app)
      .post('/api/v1/credentials')
      .set('Authorization', `Bearer ${token}`)
      .send({
        subjectId: 'user-123',
        type: 'engineering_license',
        metadata: { state: 'CA' }
      })
    
    expect(response.status).toBe(201)
    expect(response.body.credential.id).toBeDefined()
  })
})
```

## 🚀 Deployment

### Build for Production

```bash
npm run build
```

### Start Production Server

```bash
npm start
```

### Docker Deployment

```bash
docker build -t credential-api:latest .
docker run -p 3000:3000 \
  -e DATABASE_URL=postgresql://... \
  -e REDIS_URL=redis://... \
  credential-api:latest
```

### Environment-Specific Deployment

```bash
# Testnet
NODE_ENV=staging STELLAR_NETWORK=testnet npm start

# Mainnet
NODE_ENV=production STELLAR_NETWORK=mainnet npm start
```

## ⚡ Performance

### Caching Strategy

- Credentials: 5 minute TTL
- Quorum Slices: 10 minute TTL
- User data: 30 minute TTL
- Smart contract state: 1 minute TTL

### Database Optimization

- Connection pooling (20-50 connections)
- Indexed queries on frequently searched fields
- Batch operations for bulk inserts
- Query result pagination (limit 100)

### API Optimization

- Response compression (gzip)
- HTTP/2 support
- ETag support for caching
- Lazy loading of related data

## 🔒 Security

### Security Headers

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000
Content-Security-Policy: default-src 'self'
```

### Input Validation

- All inputs validated and sanitized
- SQL injection prevention (parameterized queries)
- XSS prevention (output encoding)
- CSRF tokens on state-changing operations

### Secrets Management

- Environment variables for sensitive data
- Never commit secrets to git
- Use HashiCorp Vault or AWS Secrets Manager in production
- Regular secret rotation

## 📊 Monitoring

### Health Check Endpoint

```bash
GET /health
GET /health/ready
GET /health/live
```

### Logging

```bash
# View logs
npm run logs

# Export logs
npm run logs:export
```

### Metrics

- Request count and latency
- Error rates and types
- Database query performance
- Cache hit/miss rates

## 🤝 Contributing

1. Fork repository
2. Create feature branch: `git checkout -b feature/new-feature`
3. Commit changes: `git commit -m 'Add new feature'`
4. Push branch: `git push origin feature/new-feature`
5. Open Pull Request

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

## 🆘 Support

- Open issues: [GitHub Issues](https://github.com/credential-labs/credential-protocol-api/issues)
- Documentation: [API Docs](./docs)
- Slack: #api-support

---

**Built with ❤️ by Credential Labs**
