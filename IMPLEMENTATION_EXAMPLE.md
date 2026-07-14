# API Implementation Example

## Credential Verification & Attestation Endpoint

This example demonstrates how to implement a production-grade API endpoint for verifying credentials and managing attestations with proper error handling, validation, and logging.

### Express Route Handler

```typescript
// routes/credentials.ts
import { Router, Request, Response, NextFunction } from 'express'
import { CredentialService } from '@/services/CredentialService'
import { validateCredentialInput } from '@/middleware/validation'
import { authenticate, authorize } from '@/middleware/auth'
import { logger } from '@/utils/logger'

const router = Router()
const credentialService = new CredentialService()

// Verify a credential by ID
router.get(
  '/credentials/:credentialId/verify',
  authenticate,
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { credentialId } = req.params
      const { includeAttesters = false } = req.query

      // Validate input
      if (!credentialId || typeof credentialId !== 'string') {
        return res.status(400).json({
          error: {
            code: 'INVALID_CREDENTIAL_ID',
            message: 'Credential ID must be a valid string'
          }
        })
      }

      logger.info('Verifying credential', {
        credentialId,
        userId: req.user?.id,
        timestamp: new Date().toISOString()
      })

      // Fetch credential from smart contract
      const credential = await credentialService.verifyCredential(credentialId)

      if (!credential) {
        return res.status(404).json({
          error: {
            code: 'CREDENTIAL_NOT_FOUND',
            message: `Credential ${credentialId} not found`,
            statusCode: 404
          }
        })
      }

      // Build response
      const response: any = {
        credential: {
          id: credential.id,
          type: credential.credential_type,
          issuer: credential.issuer,
          subject: credential.subject,
          issuedAt: credential.issued_at,
          revokedAt: credential.revoked_at,
          isRevoked: !!credential.revoked_at,
          metadataHash: credential.metadata_hash
        },
        verification: {
          isValid: !credential.revoked_at,
          verifiedAt: new Date().toISOString(),
          verificationMethod: 'soroban_contract'
        }
      }

      // Include attesters if requested
      if (includeAttesters) {
        const attesters = await credentialService.getAttestations(credentialId)
        response.attestations = attesters.map(att => ({
          attestor: att.attestor,
          status: att.status,
          attestedAt: att.attested_at
        }))
      }

      logger.info('Credential verified successfully', {
        credentialId,
        isValid: !credential.revoked_at
      })

      return res.status(200).json(response)
    } catch (error) {
      next(error)
    }
  }
)

// Create attestation request
router.post(
  '/credentials/:credentialId/attest',
  authenticate,
  authorize(['issuer', 'attestor']),
  validateCredentialInput,
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { credentialId } = req.params
      const { attestorAddress, signatureProof } = req.body
      const userId = req.user?.id

      logger.info('Creating attestation', {
        credentialId,
        attestor: attestorAddress,
        userId
      })

      // Verify attestor is authorized
      const isAuthorized = await credentialService.isAuthorizedAttestor(
        credentialId,
        attestorAddress
      )

      if (!isAuthorized) {
        return res.status(403).json({
          error: {
            code: 'UNAUTHORIZED_ATTESTOR',
            message: 'Address is not authorized to attest this credential',
            statusCode: 403
          }
        })
      }

      // Verify signature
      const isSignatureValid = await credentialService.verifySignature(
        credentialId,
        attestorAddress,
        signatureProof
      )

      if (!isSignatureValid) {
        return res.status(401).json({
          error: {
            code: 'INVALID_SIGNATURE',
            message: 'Signature verification failed',
            statusCode: 401
          }
        })
      }

      // Submit attestation to smart contract
      const txHash = await credentialService.submitAttestation(
        credentialId,
        attestorAddress,
        signatureProof
      )

      logger.info('Attestation created', {
        credentialId,
        txHash,
        attestor: attestorAddress
      })

      return res.status(201).json({
        success: true,
        attestation: {
          id: `att_${credentialId}_${attestorAddress}`,
          credentialId,
          attestor: attestorAddress,
          status: 'pending',
          transactionHash: txHash,
          createdAt: new Date().toISOString()
        }
      })
    } catch (error) {
      next(error)
    }
  }
)

// Batch verify credentials
router.post(
  '/credentials/batch/verify',
  authenticate,
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { credentialIds } = req.body

      if (!Array.isArray(credentialIds) || credentialIds.length === 0) {
        return res.status(400).json({
          error: {
            code: 'INVALID_INPUT',
            message: 'credentialIds must be a non-empty array'
          }
        })
      }

      if (credentialIds.length > 100) {
        return res.status(400).json({
          error: {
            code: 'BATCH_SIZE_EXCEEDED',
            message: 'Maximum 100 credentials per batch'
          }
        })
      }

      logger.info('Batch verifying credentials', {
        count: credentialIds.length,
        userId: req.user?.id
      })

      // Verify all credentials in parallel
      const results = await Promise.all(
        credentialIds.map(id =>
          credentialService
            .verifyCredential(id)
            .then(credential => ({
              id,
              success: true,
              isValid: !credential?.revoked_at,
              credential
            }))
            .catch(error => ({
              id,
              success: false,
              error: error.message
            }))
        )
      )

      const summary = {
        total: results.length,
        successful: results.filter(r => r.success).length,
        failed: results.filter(r => !r.success).length,
        validCredentials: results.filter(r => r.success && r.isValid).length
      }

      logger.info('Batch verification completed', summary)

      return res.status(200).json({
        summary,
        results
      })
    } catch (error) {
      next(error)
    }
  }
)

export default router
```

### Service Implementation

```typescript
// services/CredentialService.ts
import { SorobanClient } from '@stellar/js-sdk'
import { logger } from '@/utils/logger'
import { cache } from '@/utils/cache'

export class CredentialService {
  private sorobanClient: SorobanClient

  constructor() {
    this.sorobanClient = new SorobanClient({
      rpcUrl: process.env.STELLAR_RPC_URL,
      allowHttp: false
    })
  }

  /**
   * Verify a credential by calling the smart contract
   */
  async verifyCredential(credentialId: string) {
    try {
      // Check cache first (5 minute TTL)
      const cached = await cache.get(`credential:${credentialId}`)
      if (cached) {
        logger.debug('Credential cache hit', { credentialId })
        return JSON.parse(cached)
      }

      // Call smart contract
      const credential = await this.sorobanClient.call(
        process.env.CONTRACT_CREDENTIAL_PROTOCOL!,
        'get_credential',
        [credentialId]
      )

      // Cache result
      await cache.set(
        `credential:${credentialId}`,
        JSON.stringify(credential),
        300 // 5 minutes
      )

      return credential
    } catch (error) {
      logger.error('Failed to verify credential', {
        credentialId,
        error: error instanceof Error ? error.message : String(error)
      })
      throw error
    }
  }

  /**
   * Get all attestations for a credential
   */
  async getAttestations(credentialId: string) {
    try {
      const attestations = await this.sorobanClient.call(
        process.env.CONTRACT_CREDENTIAL_PROTOCOL!,
        'get_attestors',
        [credentialId]
      )

      return attestations || []
    } catch (error) {
      logger.error('Failed to get attestations', { credentialId, error })
      return []
    }
  }

  /**
   * Check if address is authorized to attest
   */
  async isAuthorizedAttestor(credentialId: string, attestorAddress: string) {
    try {
      const credential = await this.verifyCredential(credentialId)
      const authorizedAttestors = credential.authorized_attestors || []

      return authorizedAttestors.includes(attestorAddress)
    } catch (error) {
      logger.error('Failed to check attestor authorization', {
        credentialId,
        attestorAddress,
        error
      })
      return false
    }
  }

  /**
   * Verify cryptographic signature
   */
  async verifySignature(
    credentialId: string,
    signer: string,
    signature: string
  ): Promise<boolean> {
    try {
      // Verify signature using Stellar SDK
      const message = `Attest-${credentialId}`
      const isValid = this.sorobanClient.verifySignature(
        signer,
        message,
        signature
      )

      return isValid
    } catch (error) {
      logger.error('Signature verification failed', { credentialId, signer })
      return false
    }
  }

  /**
   * Submit attestation to blockchain
   */
  async submitAttestation(
    credentialId: string,
    attestorAddress: string,
    signature: string
  ): Promise<string> {
    try {
      const txHash = await this.sorobanClient.call(
        process.env.CONTRACT_CREDENTIAL_PROTOCOL!,
        'attest',
        [credentialId, attestorAddress, signature]
      )

      logger.info('Attestation submitted', {
        credentialId,
        attestor: attestorAddress,
        txHash
      })

      return txHash
    } catch (error) {
      logger.error('Failed to submit attestation', {
        credentialId,
        attestor: attestorAddress,
        error
      })
      throw error
    }
  }
}
```

### Middleware & Error Handling

```typescript
// middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express'
import { logger } from '@/utils/logger'

export const errorHandler = (
  error: any,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const statusCode = error.statusCode || 500
  const errorCode = error.code || 'INTERNAL_SERVER_ERROR'
  const message = error.message || 'An unexpected error occurred'

  logger.error('Request error', {
    method: req.method,
    path: req.path,
    statusCode,
    errorCode,
    message,
    stack: error.stack
  })

  res.status(statusCode).json({
    error: {
      code: errorCode,
      message,
      statusCode,
      timestamp: new Date().toISOString(),
      requestId: req.id
    }
  })
}

// middleware/validation.ts
export const validateCredentialInput = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const { attestorAddress, signatureProof } = req.body

  if (!attestorAddress || typeof attestorAddress !== 'string') {
    return res.status(400).json({
      error: {
        code: 'INVALID_ATTESTOR_ADDRESS',
        message: 'attestorAddress must be a valid Stellar address'
      }
    })
  }

  if (!signatureProof || typeof signatureProof !== 'string') {
    return res.status(400).json({
      error: {
        code: 'INVALID_SIGNATURE',
        message: 'signatureProof must be provided'
      }
    })
  }

  next()
}
```

### Testing

```typescript
// tests/credentials.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import request from 'supertest'
import app from '@/app'

describe('Credential Endpoints', () => {
  let token: string

  beforeEach(async () => {
    // Mock authentication
    token = 'mock-jwt-token'
  })

  it('verifies a credential successfully', async () => {
    const response = await request(app)
      .get('/api/v1/credentials/123/verify')
      .set('Authorization', `Bearer ${token}`)

    expect(response.status).toBe(200)
    expect(response.body.credential.id).toBe('123')
    expect(response.body.verification).toBeDefined()
  })

  it('returns 404 for non-existent credential', async () => {
    const response = await request(app)
      .get('/api/v1/credentials/nonexistent/verify')
      .set('Authorization', `Bearer ${token}`)

    expect(response.status).toBe(404)
    expect(response.body.error.code).toBe('CREDENTIAL_NOT_FOUND')
  })

  it('creates attestation with valid signature', async () => {
    const response = await request(app)
      .post('/api/v1/credentials/123/attest')
      .set('Authorization', `Bearer ${token}`)
      .send({
        attestorAddress: 'GB...',
        signatureProof: 'signature...'
      })

    expect(response.status).toBe(201)
    expect(response.body.attestation.status).toBe('pending')
  })

  it('batch verifies credentials', async () => {
    const response = await request(app)
      .post('/api/v1/credentials/batch/verify')
      .set('Authorization', `Bearer ${token}`)
      .send({
        credentialIds: ['123', '456', '789']
      })

    expect(response.status).toBe(200)
    expect(response.body.summary.total).toBe(3)
  })
})
```

### Key Features

1. **Error Handling**: Comprehensive error responses with codes and messages
2. **Caching**: Redis caching for frequently accessed credentials
3. **Logging**: Structured logging for debugging and monitoring
4. **Validation**: Input validation middleware
5. **Authorization**: Role-based access control
6. **Batch Operations**: Efficient batch processing
7. **Testing**: Unit and integration tests
8. **Documentation**: Clear comments and examples

This implementation demonstrates production-grade API development practices suitable for a blockchain-based credential system.
