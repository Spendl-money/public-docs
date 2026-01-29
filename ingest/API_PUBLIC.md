# Spendl Ingest API — Integration Guide

Welcome! This guide walks you through integrating with Spendl's data ingestion API.

## Getting Started

### Environments
- **Production**: https://ingest.spendl.money
- **Staging**: https://ingest.staging.spendl.money
- **Local**: http://localhost:3000

Contact your Spendl account manager to receive your API credentials.

---

## Step 1: Authenticate

Exchange your API credentials for a JWT token. Include this token in all subsequent requests.

### Request
```
POST /auth/token
Content-Type: application/json

{
  "apiKey": "ak_xxx",
  "apiSecret": "sk_xxx"
}
```

### Response
```json
{
  "accessToken": "<JWT>",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

**Notes:**
- Store the `accessToken` securely.
- Tokens expire in 7 days. Refresh by calling `/auth/token` again before expiry.
- Use `Authorization: Bearer <accessToken>` in protected endpoint headers.

---

## Step 2: Ingest Data

Send user data to Spendl. The API accepts single users, batches, or TaskFlow-formatted payloads.

### Request
```
POST /api/ingest
Authorization: Bearer <accessToken>
Content-Type: application/json
```

#### Single User
```json
{
  "data": {
    "userDetails": {
      "email": "user@example.com"
    }
  }
}
```

#### Batch Upload
```json
{
  "data": [
    { "userDetails": { "email": "user1@example.com" } },
    { "userDetails": { "email": "user2@example.com" } }
  ]
}
```

#### TaskFlow Format
```json
{
  "data": {
    "email": "user@example.com",
    "name": "John",
    "surname": "Doe",
    "birth_date": "1990-01-15",
    "phone": "+1234567890",
    "country_id": "US",
    "city_id": "New York",
    "zip": "10001",
    "street": "123 Main Street",
    "company": "Acme Inc",
    "employment_status": "employed",
    "accept_terms_and_conditions": "2024-01-01T12:00:00Z"
  }
}
```

### Responses

#### Success (Single)
```json
{
  "success": true,
  "user": {
    "_id": "507f1f77bcf86cd799439011",
    "userDetails": {
      "email": "user@example.com",
      "status": "active"
    },
    "createdByApiKeyId": "id_abc123def456",
    "createdByTenant": "Your Company Name",
    "createdAt": "2026-01-28T10:30:00Z",
    "updatedAt": "2026-01-28T10:30:00Z"
  }
}
```

#### Success (Batch)
```json
{
  "inserted": [
    {
      "_id": "507f1f77bcf86cd799439011",
      "userDetails": { "email": "user1@example.com" },
      "createdByApiKeyId": "id_abc123def456",
      "createdByTenant": "Your Company Name"
    },
    {
      "_id": "507f1f77bcf86cd799439012",
      "userDetails": { "email": "user2@example.com" },
      "createdByApiKeyId": "id_abc123def456",
      "createdByTenant": "Your Company Name"
    }
  ],
  "insertedCount": 2,
  "failed": [
    {
      "email": "duplicate@example.com",
      "reason": "DUPLICATE_EMAIL"
    }
  ],
  "failedCount": 1
}
```

#### Validation Error
```json
{
  "error": {
    "code": 422,
    "type": "VALIDATION_ERROR",
    "message": "userDetails.email is required"
  }
}
```

### Common Status Codes
| Code | Meaning |
|------|---------|
| 200 | Success |
| 401 | Invalid or missing authentication token |
| 409 | Duplicate email (single insert only) |
| 422 | Validation error in request body |

---

## Step 3: Upload KYC Documents

After ingesting users, upload their identity documents (front/back of ID, passport, proof of residence, etc.).

### Endpoint
```
POST /api/kyc/:userId/:documentType/:category/:filename
Authorization: Bearer <accessToken>
Content-Type: multipart/form-data
```

### Path Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| userId | string | ✓ | The MongoDB `_id` of the user |
| documentType | string | ✓ | Type of document (see below) |
| category | string | ✓ | `ID` for identity documents, `POR` for proof of residence |
| filename | string | ✓ | Original filename (e.g., `drivers-license.jpg`) |

### Document Types
| documentType | category | Description |
|--------------|----------|-------------|
| `idProofFront` | `ID` | Front of ID card |
| `idProofBack` | `ID` | Back of ID card |
| `proofPassport` | `ID` | Passport photo page |
| `proofPermit` | `ID` | Work/residence permit |
| `proofRes` | `POR` | Utility bill, bank statement, etc. |

### Request Body
Send the file as `multipart/form-data` with a `file` field.

### Supported File Types
- JPEG/JPG (`image/jpeg`)
- PNG (`image/png`)
- PDF (`application/pdf`)

**Max file size:** 10 MB

### Response
```json
{
  "success": true
}
```

### Error Responses
| Code | Type | Description |
|------|------|-------------|
| 401 | `INVALID_TOKEN` | Invalid or missing authentication token |
| 404 | `USER_NOT_FOUND` | User with given ID not found |
| 413 | `FILE_TOO_LARGE` | File exceeds 10MB limit |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | File type not supported |
| 422 | `INVALID_DOCUMENT_TYPE` | Invalid documentType parameter |
| 422 | `CATEGORY_MISMATCH` | Category doesn't match documentType |

### Example (cURL)
```bash
curl -X POST "https://ingest.spendl.money/api/kyc/507f1f77bcf86cd799439011/idProofFront/ID/drivers-license.jpg" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/drivers-license.jpg"
```

---

## Attribution

All records you create are automatically attributed to your API key:

### User Records
Each user includes:
- `createdByApiKeyId`: Your API key ID
- `createdByTenant`: Your organization name (from your API key's label)

### KYC Documents
Each uploaded document is tracked with full history:
- `uploadedByApiKeyId`: API key that uploaded the document
- `uploadedByTenant`: Organization that uploaded the document
- `uploadedAt`: When the document was uploaded

To retrieve the latest document for a type:
```javascript
const latestIdFront = user.kycDetails.s3Keys.idProofFront?.at(-1);
```

The tenant value comes from the `label` set during your API key generation. Contact your Spendl account manager to update your label.

---

## Step 4 (Optional): Validate Without Inserting

Test your payload for validity without creating users.

### Request
```
POST /api/validate
Authorization: Bearer <accessToken>
Content-Type: application/json

{
  "data": { "userDetails": { "email": "user@example.com" } }
}
```

### Response
```json
{
  "valid": true
}
```

Or if invalid:
```json
{
  "valid": false,
  "errors": [
    {
      "email": "user@example.com",
      "errors": ["userDetails.email format is invalid"]
    }
  ]
}
```

---

## API Request Headers

All requests must include:
```
Content-Type: application/json
```

Protected endpoints (`/api/*`) also require:
```
Authorization: Bearer <accessToken>
```

---

## Data Fields

### Attribution Fields (Read-Only)
| Field | Type | Notes |
|-------|------|-------|
| createdByApiKeyId | string | API key that created the user (auto-populated) |
| createdByTenant | string | null | Organization name from API key label (auto-populated) |

### Core User Fields
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| email | string | ✓ | Valid email address |
| password | string | | If provided, user can log in |
| status | string | | One of: `active`, `inactive`, `suspended`, `pending` |

### KYC Data Fields
| Field | Type | Notes |
|-------|------|-------|
| name | string | First name or full name |
| surname | string | Last name |
| dob | string | Date of birth (ISO 8601 format) |
| phone | string | Primary phone number |
| altPhone | string | Alternate phone number |
| nationality | string | ISO country code |
| gender | string | Gender identifier |
| company | string | Employer or company name |
| employmentStatus | string | e.g., "employed", "self-employed", "unemployed" |
| sourceOfFunds | string | Primary income source |
| streetNumber | string | Street number/building number |
| addressLine1 | string | Primary address line |
| addressLine2 | string | Secondary address line |
| suburb | string | Suburb or district |
| city | string | City name |
| province | string | State/province |
| postalCode | string | ZIP or postal code |
| country | string | Country name |
| tncAccepted | boolean | Terms and conditions acceptance |
| tncAcceptedAt | string | ISO timestamp when accepted |

---

## Examples

### cURL — Get Token
```bash
curl -X POST "https://ingest.spendl.money/auth/token" \
  -H "Content-Type: application/json" \
  -d '{"apiKey":"ak_xxx","apiSecret":"sk_xxx"}'
```

### cURL — Ingest Single User
```bash
curl -X POST "https://ingest.spendl.money/api/ingest" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "data": {
      "userDetails": { "email": "user@example.com" },
      "name": "John",
      "surname": "Doe"
    }
  }'
```

### cURL — Batch Upload
```bash
curl -X POST "https://ingest.spendl.money/api/ingest" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "data": [
      { "userDetails": { "email": "user1@example.com", "name": "John" } },
      { "userDetails": { "email": "user2@example.com", "name": "Jane" } }
    ]
  }'
```

---

## Postman Collection

An importable Postman collection is available at `docs/auth.postman_collection.json`.

Use the environment template `docs/postman_environment.template.json` to configure:
- `baseUrl` — Your API environment
- `apiKey` — Your API key
- `apiSecret` — Your API secret
- `accessToken` — Will be auto-populated after token request

---

## Support

For questions or technical issues, contact your Spendl integration partner.

---

**Last Updated:** January 2026 — Added tenant and API key attribution for users and KYC uploads
