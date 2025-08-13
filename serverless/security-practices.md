# Serverless Security Practices

## Authentication and Authorization

### JWT-Based Authentication
```typescript
// src/shared/services/jwt.service.ts
import jwt from 'jsonwebtoken';
import { GetSecretValueCommand, SecretsManagerClient } from '@aws-sdk/client-secrets-manager';

interface JwtPayload {
  sub: string;
  email: string;
  roles: string[];
  iat: number;
  exp: number;
}

export class JwtService {
  private secretsClient: SecretsManagerClient;
  private jwtSecret: string | null = null;
  
  constructor() {
    this.secretsClient = new SecretsManagerClient({
      region: process.env.AWS_REGION
    });
  }
  
  private async getJwtSecret(): Promise<string> {
    if (this.jwtSecret) {
      return this.jwtSecret;
    }
    
    try {
      const result = await this.secretsClient.send(new GetSecretValueCommand({
        SecretId: process.env.JWT_SECRET_ARN
      }));
      
      this.jwtSecret = result.SecretString!;
      return this.jwtSecret;
    } catch (error) {
      console.error('Error retrieving JWT secret:', error);
      throw new Error('Failed to retrieve JWT secret');
    }
  }
  
  async sign(payload: Omit<JwtPayload, 'iat' | 'exp'>): Promise<string> {
    const secret = await this.getJwtSecret();
    
    return jwt.sign(payload, secret, {
      expiresIn: '24h',
      issuer: process.env.JWT_ISSUER || 'my-serverless-app'
    });
  }
  
  async verify(token: string): Promise<JwtPayload> {
    const secret = await this.getJwtSecret();
    
    try {
      const payload = jwt.verify(token, secret) as JwtPayload;
      return payload;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('Token expired');
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error('Invalid token');
      }
      throw error;
    }
  }
  
  async refresh(token: string): Promise<string> {
    const payload = await this.verify(token);
    
    // Create new token with same payload but fresh expiration
    const newPayload = {
      sub: payload.sub,
      email: payload.email,
      roles: payload.roles
    };
    
    return this.sign(newPayload);
  }
}
```

### Lambda Authorizer
```typescript
// src/functions/auth/authorizer.ts
import { APIGatewayTokenAuthorizerHandler, APIGatewayAuthorizerResult } from 'aws-lambda';
import { JwtService } from '@/shared/services/jwt.service';

const jwtService = new JwtService();

export const handler: APIGatewayTokenAuthorizerHandler = async (event) => {
  try {
    const token = event.authorizationToken.replace('Bearer ', '');
    const payload = await jwtService.verify(token);
    
    const policy = generatePolicy(payload.sub, 'Allow', event.methodArn, {
      userId: payload.sub,
      email: payload.email,
      roles: payload.roles.join(',')
    });
    
    return policy;
  } catch (error) {
    console.error('Authorization failed:', error);
    
    // Return deny policy
    return generatePolicy('user', 'Deny', event.methodArn);
  }
};

function generatePolicy(
  principalId: string,
  effect: 'Allow' | 'Deny',
  resource: string,
  context?: Record<string, string>
): APIGatewayAuthorizerResult {
  return {
    principalId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [
        {
          Action: 'execute-api:Invoke',
          Effect: effect,
          Resource: resource
        }
      ]
    },
    context
  };
}
```

### Role-Based Access Control
```typescript
// src/shared/middleware/rbac.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { MiddlewareObj } from '@middy/core';
import { createResponse } from '@/shared/utils/response';

interface AuthorizedEvent extends APIGatewayProxyEvent {
  user?: {
    id: string;
    email: string;
    roles: string[];
  };
}

export const rbacMiddleware = (
  requiredRoles: string[]
): MiddlewareObj<AuthorizedEvent, APIGatewayProxyResult> => {
  return {
    before: async (request) => {
      const { event } = request;
      
      // Extract user from authorizer context
      if (event.requestContext.authorizer) {
        const context = event.requestContext.authorizer;
        event.user = {
          id: context.userId,
          email: context.email,
          roles: context.roles ? context.roles.split(',') : []
        };
      }
      
      if (!event.user) {
        return createResponse(401, { message: 'Unauthorized' });
      }
      
      // Check if user has required roles
      const hasRequiredRole = requiredRoles.some(role => 
        event.user!.roles.includes(role)
      );
      
      if (!hasRequiredRole) {
        return createResponse(403, { 
          message: 'Insufficient permissions',
          requiredRoles,
          userRoles: event.user.roles
        });
      }
    }
  };
};

// Usage in function
import { middyfy } from '@/shared/middleware/middy';

const deleteUserHandler: APIGatewayProxyHandler = async (event) => {
  // Function implementation
};

export const handler = middyfy(deleteUserHandler)
  .use(rbacMiddleware(['admin', 'user-manager']));
```

## Secrets Management

### AWS Secrets Manager Integration
```typescript
// src/shared/services/secrets.service.ts
import { 
  SecretsManagerClient, 
  GetSecretValueCommand,
  CreateSecretCommand,
  UpdateSecretCommand 
} from '@aws-sdk/client-secrets-manager';

export class SecretsService {
  private client: SecretsManagerClient;
  private cache = new Map<string, { value: string; timestamp: number }>();
  private cacheTTL = 5 * 60 * 1000; // 5 minutes
  
  constructor() {
    this.client = new SecretsManagerClient({
      region: process.env.AWS_REGION
    });
  }
  
  async getSecret(secretId: string): Promise<string> {
    // Check cache first
    const cached = this.cache.get(secretId);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }
    
    try {
      const result = await this.client.send(new GetSecretValueCommand({
        SecretId: secretId
      }));
      
      const secretValue = result.SecretString!;
      
      // Cache the secret
      this.cache.set(secretId, {
        value: secretValue,
        timestamp: Date.now()
      });
      
      return secretValue;
    } catch (error) {
      console.error(`Error retrieving secret ${secretId}:`, error);
      throw new Error(`Failed to retrieve secret: ${secretId}`);
    }
  }
  
  async getSecretJson<T>(secretId: string): Promise<T> {
    const secretValue = await this.getSecret(secretId);
    
    try {
      return JSON.parse(secretValue) as T;
    } catch (error) {
      throw new Error(`Secret ${secretId} is not valid JSON`);
    }
  }
  
  async createSecret(name: string, value: string, description?: string): Promise<void> {
    try {
      await this.client.send(new CreateSecretCommand({
        Name: name,
        SecretString: value,
        Description: description
      }));
    } catch (error) {
      console.error(`Error creating secret ${name}:`, error);
      throw error;
    }
  }
  
  async updateSecret(secretId: string, value: string): Promise<void> {
    try {
      await this.client.send(new UpdateSecretCommand({
        SecretId: secretId,
        SecretString: value
      }));
      
      // Invalidate cache
      this.cache.delete(secretId);
    } catch (error) {
      console.error(`Error updating secret ${secretId}:`, error);
      throw error;
    }
  }
}

export const secretsService = new SecretsService();
```

### Database Credentials Management
```typescript
// src/shared/services/database-credentials.service.ts
import { secretsService } from './secrets.service';

interface DatabaseCredentials {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

export class DatabaseCredentialsService {
  private credentials: DatabaseCredentials | null = null;
  private lastFetch = 0;
  private refreshInterval = 15 * 60 * 1000; // 15 minutes
  
  async getCredentials(): Promise<DatabaseCredentials> {
    const now = Date.now();
    
    // Refresh credentials periodically
    if (!this.credentials || now - this.lastFetch > this.refreshInterval) {
      this.credentials = await secretsService.getSecretJson<DatabaseCredentials>(
        process.env.DB_CREDENTIALS_SECRET_ARN!
      );
      this.lastFetch = now;
    }
    
    return this.credentials;
  }
  
  async getConnectionString(): Promise<string> {
    const creds = await this.getCredentials();
    return `postgresql://${creds.username}:${creds.password}@${creds.host}:${creds.port}/${creds.database}`;
  }
}

export const dbCredentialsService = new DatabaseCredentialsService();
```

## Input Validation and Sanitization

### Request Validation
```typescript
// src/shared/schemas/user.schemas.ts
import Joi from 'joi';

export const createUserSchema = Joi.object({
  email: Joi.string()
    .email()
    .required()
    .max(255)
    .messages({
      'string.email': 'Please provide a valid email address',
      'any.required': 'Email is required'
    }),
  
  name: Joi.string()
    .required()
    .min(2)
    .max(100)
    .pattern(/^[a-zA-Z\s]+$/)
    .messages({
      'string.pattern.base': 'Name can only contain letters and spaces',
      'string.min': 'Name must be at least 2 characters long'
    }),
  
  password: Joi.string()
    .required()
    .min(8)
    .max(128)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/)
    .messages({
      'string.pattern.base': 'Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character'
    }),
  
  age: Joi.number()
    .integer()
    .min(13)
    .max(120)
    .optional(),
  
  preferences: Joi.object({
    newsletter: Joi.boolean().default(false),
    notifications: Joi.boolean().default(true)
  }).optional()
});

export const updateUserSchema = Joi.object({
  name: Joi.string()
    .min(2)
    .max(100)
    .pattern(/^[a-zA-Z\s]+$/)
    .optional(),
  
  age: Joi.number()
    .integer()
    .min(13)
    .max(120)
    .optional(),
  
  preferences: Joi.object({
    newsletter: Joi.boolean(),
    notifications: Joi.boolean()
  }).optional()
}).min(1); // At least one field must be provided
```

### SQL Injection Prevention
```typescript
// src/shared/services/secure-database.service.ts
import { Pool, PoolClient } from 'pg';
import { dbCredentialsService } from './database-credentials.service';

export class SecureDatabaseService {
  private pool: Pool | null = null;
  
  private async getPool(): Promise<Pool> {
    if (!this.pool) {
      const connectionString = await dbCredentialsService.getConnectionString();
      
      this.pool = new Pool({
        connectionString,
        ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
        max: 20,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000
      });
    }
    
    return this.pool;
  }
  
  // Safe parameterized query
  async query<T = any>(text: string, params: any[] = []): Promise<T[]> {
    const pool = await this.getPool();
    const client = await pool.connect();
    
    try {
      const result = await client.query(text, params);
      return result.rows;
    } finally {
      client.release();
    }
  }
  
  // Safe user lookup with parameterized query
  async getUserByEmail(email: string): Promise<User | null> {
    const query = 'SELECT id, email, name, created_at FROM users WHERE email = $1';
    const results = await this.query<User>(query, [email]);
    
    return results.length > 0 ? results[0] : null;
  }
  
  // Safe user creation with parameterized query
  async createUser(userData: CreateUserRequest): Promise<User> {
    const query = `
      INSERT INTO users (id, email, name, password_hash, created_at, updated_at)
      VALUES ($1, $2, $3, $4, $5, $6)
      RETURNING id, email, name, created_at
    `;
    
    const id = generateUUID();
    const now = new Date().toISOString();
    const passwordHash = await hashPassword(userData.password);
    
    const results = await this.query<User>(query, [
      id,
      userData.email,
      userData.name,
      passwordHash,
      now,
      now
    ]);
    
    return results[0];
  }
  
  // Transaction support
  async withTransaction<T>(callback: (client: PoolClient) => Promise<T>): Promise<T> {
    const pool = await this.getPool();
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```

## Encryption and Data Protection

### Data Encryption at Rest
```typescript
// src/shared/services/encryption.service.ts
import { 
  KMSClient, 
  EncryptCommand, 
  DecryptCommand,
  GenerateDataKeyCommand 
} from '@aws-sdk/client-kms';
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

export class EncryptionService {
  private kmsClient: KMSClient;
  private keyId: string;
  
  constructor() {
    this.kmsClient = new KMSClient({ region: process.env.AWS_REGION });
    this.keyId = process.env.KMS_KEY_ID!;
  }
  
  // Encrypt small data using KMS directly
  async encryptWithKMS(plaintext: string): Promise<string> {
    try {
      const result = await this.kmsClient.send(new EncryptCommand({
        KeyId: this.keyId,
        Plaintext: Buffer.from(plaintext, 'utf8')
      }));
      
      return Buffer.from(result.CiphertextBlob!).toString('base64');
    } catch (error) {
      console.error('KMS encryption error:', error);
      throw new Error('Failed to encrypt data');
    }
  }
  
  // Decrypt data encrypted with KMS
  async decryptWithKMS(ciphertext: string): Promise<string> {
    try {
      const result = await this.kmsClient.send(new DecryptCommand({
        CiphertextBlob: Buffer.from(ciphertext, 'base64')
      }));
      
      return Buffer.from(result.Plaintext!).toString('utf8');
    } catch (error) {
      console.error('KMS decryption error:', error);
      throw new Error('Failed to decrypt data');
    }
  }
  
  // Envelope encryption for large data
  async encryptLargeData(plaintext: string): Promise<{
    encryptedData: string;
    encryptedKey: string;
  }> {
    try {
      // Generate data key
      const dataKeyResult = await this.kmsClient.send(new GenerateDataKeyCommand({
        KeyId: this.keyId,
        KeySpec: 'AES_256'
      }));
      
      const plaintextKey = Buffer.from(dataKeyResult.Plaintext!);
      const encryptedKey = Buffer.from(dataKeyResult.CiphertextBlob!).toString('base64');
      
      // Encrypt data with the data key
      const iv = randomBytes(16);
      const cipher = createCipheriv('aes-256-cbc', plaintextKey, iv);
      
      let encrypted = cipher.update(plaintext, 'utf8', 'base64');
      encrypted += cipher.final('base64');
      
      // Combine IV and encrypted data
      const encryptedData = Buffer.concat([iv, Buffer.from(encrypted, 'base64')]).toString('base64');
      
      return {
        encryptedData,
        encryptedKey
      };
    } catch (error) {
      console.error('Envelope encryption error:', error);
      throw new Error('Failed to encrypt large data');
    }
  }
  
  // Decrypt envelope encrypted data
  async decryptLargeData(encryptedData: string, encryptedKey: string): Promise<string> {
    try {
      // Decrypt the data key
      const keyResult = await this.kmsClient.send(new DecryptCommand({
        CiphertextBlob: Buffer.from(encryptedKey, 'base64')
      }));
      
      const plaintextKey = Buffer.from(keyResult.Plaintext!);
      
      // Extract IV and encrypted data
      const combined = Buffer.from(encryptedData, 'base64');
      const iv = combined.slice(0, 16);
      const encrypted = combined.slice(16);
      
      // Decrypt data
      const decipher = createDecipheriv('aes-256-cbc', plaintextKey, iv);
      let decrypted = decipher.update(encrypted, undefined, 'utf8');
      decrypted += decipher.final('utf8');
      
      return decrypted;
    } catch (error) {
      console.error('Envelope decryption error:', error);
      throw new Error('Failed to decrypt large data');
    }
  }
}

export const encryptionService = new EncryptionService();
```

### Password Hashing
```typescript
// src/shared/utils/crypto.ts
import bcrypt from 'bcryptjs';
import { randomBytes, pbkdf2 } from 'crypto';
import { promisify } from 'util';

const pbkdf2Async = promisify(pbkdf2);

export async function hashPassword(password: string): Promise<string> {
  const saltRounds = 12;
  return bcrypt.hash(password, saltRounds);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

// Alternative using PBKDF2 for additional security
export async function hashPasswordPBKDF2(password: string): Promise<string> {
  const salt = randomBytes(32);
  const iterations = 100000;
  const keyLength = 64;
  const digest = 'sha512';
  
  const derivedKey = await pbkdf2Async(password, salt, iterations, keyLength, digest);
  
  // Combine salt and derived key
  const combined = Buffer.concat([salt, derivedKey]);
  return combined.toString('base64');
}

export async function verifyPasswordPBKDF2(password: string, hash: string): Promise<boolean> {
  const combined = Buffer.from(hash, 'base64');
  const salt = combined.slice(0, 32);
  const storedKey = combined.slice(32);
  
  const iterations = 100000;
  const keyLength = 64;
  const digest = 'sha512';
  
  const derivedKey = await pbkdf2Async(password, salt, iterations, keyLength, digest);
  
  return storedKey.equals(derivedKey);
}

// Generate secure random tokens
export function generateSecureToken(length = 32): string {
  return randomBytes(length).toString('hex');
}

// Generate API keys
export function generateApiKey(): string {
  const prefix = 'sk_';
  const randomPart = randomBytes(24).toString('base64').replace(/[+/=]/g, '');
  return prefix + randomPart;
}
```

## Network Security

### VPC Configuration
```yaml
# serverless.yml - VPC configuration
provider:
  name: aws
  runtime: nodejs20.x
  
  # VPC configuration for secure networking
  vpc:
    securityGroupIds:
      - ${self:custom.securityGroups.lambda}
    subnetIds:
      - ${self:custom.subnets.private.a}
      - ${self:custom.subnets.private.b}

custom:
  # Security groups
  securityGroups:
    lambda: sg-lambda-functions
    rds: sg-rds-database
    redis: sg-redis-cache
  
  # Subnets
  subnets:
    private:
      a: subnet-private-1a
      b: subnet-private-1b
    public:
      a: subnet-public-1a
      b: subnet-public-1b

resources:
  Resources:
    # Lambda security group
    LambdaSecurityGroup:
      Type: AWS::EC2::SecurityGroup
      Properties:
        GroupDescription: Security group for Lambda functions
        VpcId: ${self:custom.vpcId}
        SecurityGroupEgress:
          # Allow HTTPS outbound
          - IpProtocol: tcp
            FromPort: 443
            ToPort: 443
            CidrIp: 0.0.0.0/0
          # Allow database access
          - IpProtocol: tcp
            FromPort: 5432
            ToPort: 5432
            DestinationSecurityGroupId: !Ref DatabaseSecurityGroup
          # Allow Redis access
          - IpProtocol: tcp
            FromPort: 6379
            ToPort: 6379
            DestinationSecurityGroupId: !Ref RedisSecurityGroup
    
    # Database security group
    DatabaseSecurityGroup:
      Type: AWS::EC2::SecurityGroup
      Properties:
        GroupDescription: Security group for RDS database
        VpcId: ${self:custom.vpcId}
        SecurityGroupIngress:
          - IpProtocol: tcp
            FromPort: 5432
            ToPort: 5432
            SourceSecurityGroupId: !Ref LambdaSecurityGroup
```

### API Gateway Security
```yaml
# serverless.yml - API Gateway security
provider:
  apiGateway:
    # Request validation
    requestValidators:
      ValidateRequestBody:
        validateRequestBody: true
        validateRequestParameters: false
      ValidateRequestParameters:
        validateRequestBody: false
        validateRequestParameters: true
      ValidateAll:
        validateRequestBody: true
        validateRequestParameters: true
    
    # API keys for rate limiting
    apiKeys:
      - name: ${self:service}-${self:provider.stage}-api-key
        description: API key for ${self:service}
    
    # Usage plans
    usagePlan:
      quota:
        limit: 10000
        period: MONTH
      throttle:
        rateLimit: 100
        burstLimit: 200

functions:
  api:
    handler: src/handler.api
    events:
      - http:
          path: /{proxy+}
          method: ANY
          # Enable API key requirement
          private: true
          # Request validation
          request:
            parameters:
              headers:
                Authorization: true
          # CORS configuration
          cors:
            origin: ${self:custom.corsOrigins.${self:provider.stage}}
            headers:
              - Content-Type
              - X-Amz-Date
              - Authorization
              - X-Api-Key
              - X-Amz-Security-Token
            allowCredentials: true

custom:
  corsOrigins:
    dev: 'http://localhost:3000'
    staging: 'https://staging.example.com'
    prod: 'https://app.example.com'
```

## Compliance and Auditing

### Audit Logging
```typescript
// src/shared/services/audit.service.ts
import { CloudWatchLogsClient, PutLogEventsCommand } from '@aws-sdk/client-cloudwatch-logs';

interface AuditEvent {
  userId?: string;
  action: string;
  resource: string;
  resourceId?: string;
  ipAddress?: string;
  userAgent?: string;
  timestamp: string;
  success: boolean;
  details?: Record<string, any>;
}

export class AuditService {
  private cloudWatchLogs: CloudWatchLogsClient;
  private logGroupName: string;
  private logStreamName: string;
  
  constructor() {
    this.cloudWatchLogs = new CloudWatchLogsClient({
      region: process.env.AWS_REGION
    });
    this.logGroupName = `/aws/lambda/${process.env.AWS_LAMBDA_FUNCTION_NAME}/audit`;
    this.logStreamName = new Date().toISOString().split('T')[0]; // Daily log streams
  }
  
  async logEvent(event: Omit<AuditEvent, 'timestamp'>): Promise<void> {
    const auditEvent: AuditEvent = {
      ...event,
      timestamp: new Date().toISOString()
    };
    
    try {
      await this.cloudWatchLogs.send(new PutLogEventsCommand({
        logGroupName: this.logGroupName,
        logStreamName: this.logStreamName,
        logEvents: [
          {
            timestamp: Date.now(),
            message: JSON.stringify(auditEvent)
          }
        ]
      }));
    } catch (error) {
      console.error('Failed to log audit event:', error);
      // Don't throw - audit logging shouldn't break the main flow
    }
  }
  
  async logUserAction(
    userId: string,
    action: string,
    resource: string,
    resourceId?: string,
    success = true,
    details?: Record<string, any>
  ): Promise<void> {
    await this.logEvent({
      userId,
      action,
      resource,
      resourceId,
      success,
      details
    });
  }
  
  async logSecurityEvent(
    action: string,
    ipAddress: string,
    userAgent: string,
    success: boolean,
    details?: Record<string, any>
  ): Promise<void> {
    await this.logEvent({
      action,
      resource: 'security',
      ipAddress,
      userAgent,
      success,
      details
    });
  }
}

export const auditService = new AuditService();
```

### GDPR Compliance
```typescript
// src/shared/services/gdpr.service.ts
import { encryptionService } from './encryption.service';
import { auditService } from './audit.service';

export class GDPRService {
  
  // Right to be forgotten - data deletion
  async deleteUserData(userId: string, requestedBy: string): Promise<void> {
    try {
      // Log the deletion request
      await auditService.logUserAction(
        requestedBy,
        'DELETE_USER_DATA',
        'user',
        userId,
        true,
        { reason: 'GDPR_RIGHT_TO_BE_FORGOTTEN' }
      );
      
      // Delete from all systems
      await this.deleteFromDatabase(userId);
      await this.deleteFromStorage(userId);
      await this.deleteFromCache(userId);
      await this.deleteFromLogs(userId);
      
      console.log(`User data deleted for user ${userId}`);
    } catch (error) {
      await auditService.logUserAction(
        requestedBy,
        'DELETE_USER_DATA',
        'user',
        userId,
        false,
        { error: error.message }
      );
      throw error;
    }
  }
  
  // Data export for portability
  async exportUserData(userId: string, requestedBy: string): Promise<any> {
    try {
      await auditService.logUserAction(
        requestedBy,
        'EXPORT_USER_DATA',
        'user',
        userId
      );
      
      const userData = await this.collectUserData(userId);
      
      // Encrypt exported data
      const encryptedData = await encryptionService.encryptLargeData(
        JSON.stringify(userData)
      );
      
      return {
        userId,
        exportDate: new Date().toISOString(),
        data: encryptedData
      };
    } catch (error) {
      await auditService.logUserAction(
        requestedBy,
        'EXPORT_USER_DATA',
        'user',
        userId,
        false,
        { error: error.message }
      );
      throw error;
    }
  }
  
  // Consent management
  async updateConsent(
    userId: string,
    consentType: string,
    granted: boolean,
    requestedBy: string
  ): Promise<void> {
    await auditService.logUserAction(
      requestedBy,
      'UPDATE_CONSENT',
      'consent',
      userId,
      true,
      { consentType, granted }
    );
    
    // Update consent in database
    // Implementation depends on your data model
  }
  
  private async deleteFromDatabase(userId: string): Promise<void> {
    // Delete user data from database
    // Implementation depends on your database structure
  }
  
  private async deleteFromStorage(userId: string): Promise<void> {
    // Delete user files from S3
    // Implementation depends on your storage structure
  }
  
  private async deleteFromCache(userId: string): Promise<void> {
    // Delete user data from cache
    // Implementation depends on your caching strategy
  }
  
  private async deleteFromLogs(userId: string): Promise<void> {
    // Anonymize user data in logs
    // Note: Complete log deletion might not be feasible
  }
  
  private async collectUserData(userId: string): Promise<any> {
    // Collect all user data from various sources
    // Return structured data for export
    return {};
  }
}

export const gdprService = new GDPRService();
```

## Best Practices

### Security Checklist
- [ ] Use HTTPS/TLS for all communications
- [ ] Implement proper authentication and authorization
- [ ] Validate and sanitize all inputs
- [ ] Use parameterized queries to prevent SQL injection
- [ ] Store secrets in AWS Secrets Manager
- [ ] Encrypt sensitive data at rest and in transit
- [ ] Implement proper error handling without information leakage
- [ ] Use least privilege IAM policies
- [ ] Enable audit logging for security events
- [ ] Implement rate limiting and DDoS protection
- [ ] Regular security assessments and penetration testing
- [ ] Keep dependencies updated and scan for vulnerabilities

### Development Guidelines
- **Never hardcode secrets** in source code
- **Use environment variables** for configuration
- **Implement defense in depth** with multiple security layers
- **Follow OWASP guidelines** for web application security
- **Regular security training** for development team
- **Automated security scanning** in CI/CD pipeline
- **Incident response plan** for security breaches
- **Regular backup and recovery testing**

### Compliance Considerations
- **Data classification** and handling procedures
- **Privacy by design** principles
- **Regular compliance audits**
- **Data retention policies**
- **Cross-border data transfer compliance**
- **Vendor security assessments**
- **Business continuity planning**