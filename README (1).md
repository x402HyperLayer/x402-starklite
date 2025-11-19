# 🔐 x402-stark-lite

> Lightweight STARK proof ingestion + validation + verifier client for the x402 HyperLayer

A production-grade TypeScript SDK and daemon that accepts, validates, verifies signatures, stores, and forwards STARK proof artifacts to external verifier APIs.

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│         Express REST API                │
│  POST /api/v1/proofs                   │
│  GET  /api/v1/proofs/:id               │
│  GET  /api/v1/metrics                  │
└──────┬──────────────────┬───────────────┘
       │                  │
       ▼                  ▼
┌─────────────┐    ┌──────────────┐
│  Validator  │    │   Signer     │
│   (Ajv)     │    │ (TweetNaCl)  │
└─────────────┘    └──────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│         SQLite Database                 │
│  - Proof storage                        │
│  - Verifier status tracking             │
│  - Metrics aggregation                  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│      Verifier Client                    │
│  - Retry logic                          │
│  - Exponential backoff                  │
│  - Status persistence                   │
└─────────────────────────────────────────┘
```

## ✨ Features

- **JSON Schema Validation**: Strict validation using Ajv with custom formats
- **Ed25519 Signature Verification**: Cryptographic proof authenticity using TweetNaCl
- **SQLite Persistence**: Local proof storage with indexing
- **External Verifier Integration**: HTTP client with retry and exponential backoff
- **Size Metrics**: Raw and gzip compression analysis using pako
- **REST API**: Full-featured API with proof submission and retrieval
- **CLI Tool**: Command-line interface for proof ingestion and export
- **Web Dashboard**: Interactive UI for proof management
- **Production-Ready**: TypeScript, comprehensive tests, CI/CD pipeline

## 🚀 Quick Start

### Installation

```bash
npm install
```

### Configuration

Create a `.env` file (copy from `.env.example`):

```bash
PORT=8080
DB_PATH=./x402.db
VERIFIER_API_URL=https://verifier.example.com/verify
VERIFIER_API_KEY=your_api_key_here
SIGNER_PUBLIC_KEY=base64_encoded_public_key
SIG_ALGO=ed25519
```

### Running the Server

```bash
# Development mode with hot reload
npm run dev

# Production mode
npm run build
npm start
```

The server will be available at:
- API: `http://localhost:8080/api/v1`
- Dashboard: `http://localhost:8080/docs`
- Health: `http://localhost:8080/health`

## 📊 Example Proof JSON

```json
{
  "id": "proof_abc123",
  "chain": "starknet",
  "timestamp": "2025-11-19T10:30:00Z",
  "payload": {
    "program_hash": "0x1234567890abcdef",
    "output": ["0xabc", "0xdef"],
    "trace_length": 1024
  },
  "signature": "base64_encoded_ed25519_signature"
}
```

### Field Requirements

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique proof identifier |
| `chain` | string | ✅ | Blockchain identifier (e.g., "starknet") |
| `timestamp` | string | ✅ | ISO 8601 datetime |
| `payload` | object | ✅ | Proof-specific data |
| `signature` | string | ✅ | Base64-encoded Ed25519 signature |

## 🔄 Verifier Workflow

1. **Proof Submission**: Client POSTs proof to `/api/v1/proofs`
2. **Schema Validation**: Proof is validated against JSON schema
3. **Signature Verification**: Ed25519 signature is verified
4. **Database Storage**: Proof is stored with status `pending`
5. **Size Metrics**: Raw and gzip sizes are calculated
6. **Verifier Submission**: Proof is sent to external verifier API
7. **Retry Logic**: Failed submissions are retried with exponential backoff
8. **Status Update**: Verifier response updates proof status to `verified` or `failed`

### Status Flow

```
pending → submitted → verified
                   ↘ failed
```

## 📡 API Documentation

### POST /api/v1/proofs

Submit a new STARK proof for validation and verification.

**Request:**
```bash
curl -X POST http://localhost:8080/api/v1/proofs \
  -H "Content-Type: application/json" \
  -d @proof.json
```

**Response (201 Created):**
```json
{
  "success": true,
  "proofId": "proof_abc123",
  "validation": {
    "schema": true,
    "signature": true
  },
  "metrics": {
    "rawBytes": 256,
    "gzipBytes": 128,
    "compressionRatio": 0.5
  },
  "receivedAt": "2025-11-19T10:30:15.123Z"
}
```

**Error Response (400 Bad Request):**
```json
{
  "error": "Validation failed",
  "details": [
    " must have required property 'id'"
  ]
}
```

### GET /api/v1/proofs/:id

Retrieve a proof by ID.

**Request:**
```bash
curl http://localhost:8080/api/v1/proofs/proof_abc123
```

**Response (200 OK):**
```json
{
  "proof": {
    "id": "proof_abc123",
    "chain": "starknet",
    "timestamp": "2025-11-19T10:30:00Z",
    "payload": {...},
    "signature": "...",
    "receivedAt": "2025-11-19T10:30:15.123Z",
    "verifierStatus": "verified",
    "verifierMessage": "Proof verified successfully",
    "verifiedAt": "2025-11-19T10:30:18.456Z",
    "sizeRaw": 256,
    "sizeGzip": 128
  },
  "metrics": {
    "rawBytes": 256,
    "gzipBytes": 128,
    "compressionRatio": 0.5
  }
}
```

### GET /api/v1/metrics

Get system-wide metrics.

**Request:**
```bash
curl http://localhost:8080/api/v1/metrics
```

**Response (200 OK):**
```json
{
  "totalProofs": 150,
  "byStatus": {
    "pending": 5,
    "submitted": 10,
    "verified": 130,
    "failed": 5
  },
  "byChain": {
    "starknet": 100,
    "ethereum": 50
  },
  "totalSizeRaw": 38400,
  "totalSizeGzip": 19200
}
```

## 🖥️ CLI Documentation

### Ingest Proof

Read a proof from a JSON file, validate it, and submit to verifier.

```bash
npm run cli ingest proof.json
```

**Output:**
```
[2025-11-19T10:30:15.123Z] INFO: Reading proof from proof.json
[2025-11-19T10:30:15.234Z] INFO: Signature verification: PASSED
[2025-11-19T10:30:15.345Z] INFO: Proof ingested: proof_abc123
[2025-11-19T10:30:15.456Z] INFO: Raw size: 256 bytes, Gzip: 128 bytes
[2025-11-19T10:30:15.567Z] INFO: Submitting to verifier...
[2025-11-19T10:30:18.123Z] INFO: Verifier accepted proof
```

### Export Proof

Export a proof by ID to a JSON file.

```bash
npm run cli export proof_abc123
```

**Output:**
```
[2025-11-19T10:35:00.123Z] INFO: Proof exported to proof-proof_abc123.json
[2025-11-19T10:35:00.234Z] INFO: Status: verified
[2025-11-19T10:35:00.345Z] INFO: Verifier message: Proof verified successfully
```

## 🗄️ Database Schema

### `proofs` Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PRIMARY KEY | Unique proof identifier |
| `chain` | TEXT | Blockchain identifier |
| `timestamp` | TEXT | Proof timestamp |
| `payload` | TEXT | JSON-encoded proof payload |
| `signature` | TEXT | Ed25519 signature |
| `receivedAt` | TEXT | When proof was received |
| `sizeRaw` | INTEGER | Raw size in bytes |
| `sizeGzip` | INTEGER | Gzip-compressed size |
| `verifierStatus` | TEXT | pending/submitted/verified/failed |
| `verifierMessage` | TEXT | Verifier response message |
| `verifiedAt` | TEXT | When verification completed |

**Indexes:**
- `idx_chain` on `chain`
- `idx_timestamp` on `timestamp`
- `idx_verifierStatus` on `verifierStatus`

## 🧪 Testing

```bash
# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run with coverage
npm run test:coverage
```

### Test Coverage

- **Validator Tests**: Schema validation edge cases
- **Signer Tests**: Ed25519 signature verification
- **Database Tests**: CRUD operations and metrics

## 🔒 Security Warnings

⚠️ **IMPORTANT SECURITY CONSIDERATIONS**

1. **Signature Verification**: Always configure a valid `SIGNER_PUBLIC_KEY`. Without it, signature verification will be skipped.

2. **API Key Protection**: Keep your `VERIFIER_API_KEY` secret. Never commit it to version control.

3. **Input Validation**: All proofs are validated against a strict JSON schema, but be aware of payload size limits.

4. **Database Security**: The SQLite database is stored locally. Ensure proper file permissions in production.

5. **TLS/HTTPS**: Always use HTTPS for the verifier API in production to prevent man-in-the-middle attacks.

6. **Rate Limiting**: Consider implementing rate limiting on API endpoints to prevent abuse.

## 📦 Example Usage

See the `examples/submit-proof.sh` script for a complete example:

```bash
chmod +x examples/submit-proof.sh
./examples/submit-proof.sh
```

## 🛠️ Development

### Project Structure

```
x402-stark-lite/
├── src/
│   ├── index.ts          # Main server
│   ├── cli.ts            # CLI tool
│   ├── config.ts         # Configuration loader
│   ├── utils/            # Utilities
│   │   ├── logger.ts     # Timestamped logger
│   │   └── size.ts       # Size calculation
│   ├── db/
│   │   └── store.ts      # Database layer
│   ├── proof/
│   │   ├── schema.json   # JSON schema
│   │   ├── types.ts      # TypeScript types
│   │   ├── validator.ts  # Ajv validator
│   │   ├── signer.ts     # Signature verification
│   │   └── storage.ts    # Storage layer
│   ├── verifier/
│   │   └── client.ts     # Verifier HTTP client
│   └── api/
│       └── routes.ts     # API routes
├── docs/                 # Web dashboard
├── test/                 # Test suite
└── examples/             # Usage examples
```

### Building

```bash
npm run build
```

Output will be in the `dist/` directory.

## 🔧 Dependencies

### Production
- **express**: Web server framework
- **better-sqlite3**: SQLite database
- **ajv**: JSON schema validation
- **tweetnacl**: Ed25519 cryptography
- **axios**: HTTP client
- **pako**: Gzip compression
- **dotenv**: Environment configuration

### Development
- **typescript**: Type-safe development
- **vitest**: Fast unit testing
- **tsx**: TypeScript execution

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

## 🤝 Contributing

Contributions are welcome! Please ensure:

1. All tests pass: `npm test`
2. Code builds successfully: `npm run build`
3. Follow existing code style
4. Add tests for new features

## 📞 Support

For issues and questions, please open a GitHub issue.

---

**Built with ❤️ for the x402 HyperLayer ecosystem**
