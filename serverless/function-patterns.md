# Serverless Function Patterns

## Handler Patterns

### Basic HTTP Handler
```typescript
// src/functions/users/create.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { middyfy } from '@/shared/middleware/middy';
import { UserService } from '@/shared/services/user.service';
import { CreateUserRequest, CreateUserResponse } from '@/shared/types/user.types';
import { createResponse } from '@/shared/utils/response';
import { logger, enhancedLogger } from '@/shared/utils/logger';
import { tracer, metrics } from '@/shared/middleware/powertools';
import { validateRequest } from '@/shared/utils/validation';
import { createUserSchema } from '@/shared/schemas/user.schemas';

const createUserHandler: APIGatewayProxyHandler = async (event) => {
  // Create subsegment for user creation
  const subsegment = tracer.getSegment()?.addNewSubsegment('create-user');
  
  try {
    // Validate request body
    const body = validateRequest<CreateUserRequest>(event.body, createUserSchema);
    
    // Extract user context from authorizer
    const userId = event.requestContext.authorizer?.principalId;
    
    // Add tracing annotations
    tracer.putAnnotation('operation', 'createUser');
    tracer.putAnnotation('userEmail', body.email);
    
    logger.info('Creating user', { 
      email: body.email,
      requestedBy: userId
    });
    
    const userService = new UserService();
    const user = await tracer.captureAsyncFunc('userService.create', () => 
      userService.create(body)
    );
    
    const response: CreateUserResponse = {
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        createdAt: user.createdAt
      }
    };
    
    // Record business metrics
    metrics.addMetric('UserCreated', 'Count', 1);
    enhancedLogger.logBusinessEvent('user_created', {
      userId: user.id,
      email: user.email
    });
    
    return createResponse(201, response);
  } catch (error) {
    // Add error to trace
    tracer.addErrorAsMetadata(error as Error);
    
    enhancedLogger.logError('Error creating user', error as Error, {
      operation: 'createUser'
    });
    
    // Record error metrics
    metrics.addMetric('UserCreationError', 'Count', 1);
    
    if (error.name === 'ValidationError') {
      return createResponse(400, { message: error.message });
    }
    
    if (error.name === 'ConflictError') {
      return createResponse(409, { message: 'User already exists' });
    }
    
    return createResponse(500, { message: 'Internal server error' });
  } finally {
    subsegment?.close();
  }
};

export const handler = middyfy(createUserHandler)
  .use(powertoolsMiddleware());
```

### Event-Driven Handler
```typescript
// src/functions/events/user-created.ts
import { DynamoDBStreamHandler, DynamoDBRecord } from 'aws-lambda';
import { unmarshall } from '@aws-sdk/util-dynamodb';
import { EmailService } from '@/shared/services/email.service';
import { logger } from '@/shared/utils/logger';

interface UserRecord {
  id: string;
  email: string;
  name: string;
  createdAt: string;
}

const userCreatedHandler: DynamoDBStreamHandler = async (event) => {
  const emailService = new EmailService();
  
  for (const record of event.Records) {
    try {
      if (record.eventName === 'INSERT' && record.dynamodb?.NewImage) {
        const user = unmarshall(record.dynamodb.NewImage) as UserRecord;
        
        logger.info('Processing user created event', { 
          userId: user.id,
          email: user.email 
        });
        
        // Send welcome email
        await emailService.sendWelcomeEmail({
          to: user.email,
          name: user.name,
          userId: user.id
        });
        
        // Add to mailing list
        await emailService.addToMailingList(user.email, user.name);
        
        logger.info('User created event processed successfully', { 
          userId: user.id 
        });
      }
    } catch (error) {
      logger.error('Error processing user created event', {
        error: error.message,
        record: record.dynamodb?.Keys,
        eventName: record.eventName
      });
      
      // Don't throw - let other records process
      // Consider implementing DLQ for failed records
    }
  }
};

export const handler = userCreatedHandler;
```

### Scheduled Handler
```typescript
// src/functions/scheduled/cleanup.ts
import { ScheduledHandler } from 'aws-lambda';
import { UserService } from '@/shared/services/user.service';
import { logger } from '@/shared/utils/logger';

const cleanupHandler: ScheduledHandler = async (event) => {
  try {
    logger.info('Starting cleanup job', { 
      time: event.time,
      resources: event.resources 
    });
    
    const userService = new UserService();
    
    // Clean up inactive users older than 90 days
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - 90);
    
    const deletedCount = await userService.cleanupInactiveUsers(cutoffDate);
    
    logger.info('Cleanup job completed', { 
      deletedCount,
      cutoffDate: cutoffDate.toISOString() 
    });
    
    return {
      statusCode: 200,
      body: JSON.stringify({
        message: 'Cleanup completed successfully',
        deletedCount
      })
    };
  } catch (error) {
    logger.error('Error in cleanup job', { 
      error: error.message,
      stack: error.stack 
    });
    
    throw error; // Let CloudWatch capture the failure
  }
};

export const handler = cleanupHandler;
```

### SQS Handler
```typescript
// src/functions/queues/process-notifications.ts
import { SQSHandler, SQSRecord } from 'aws-lambda';
import { NotificationService } from '@/shared/services/notification.service';
import { logger } from '@/shared/utils/logger';

interface NotificationMessage {
  userId: string;
  type: 'email' | 'sms' | 'push';
  template: string;
  data: Record<string, any>;
}

const processNotificationsHandler: SQSHandler = async (event) => {
  const notificationService = new NotificationService();
  
  // Process messages in parallel with controlled concurrency
  const promises = event.Records.map(async (record: SQSRecord) => {
    try {
      const message: NotificationMessage = JSON.parse(record.body);
      
      logger.info('Processing notification', {
        messageId: record.messageId,
        userId: message.userId,
        type: message.type
      });
      
      await notificationService.send(message);
      
      logger.info('Notification processed successfully', {
        messageId: record.messageId,
        userId: message.userId
      });
    } catch (error) {
      logger.error('Error processing notification', {
        error: error.message,
        messageId: record.messageId,
        body: record.body
      });
      
      // Throw error to send message to DLQ
      throw error;
    }
  });
  
  await Promise.all(promises);
};

export const handler = processNotificationsHandler;
```

## Middleware Patterns

### Authentication Middleware
```typescript
// src/shared/middleware/auth.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { MiddlewareObj } from '@middy/core';
import { JwtService } from '@/shared/services/jwt.service';
import { createResponse } from '@/shared/utils/response';

interface AuthenticatedEvent extends APIGatewayProxyEvent {
  user?: {
    id: string;
    email: string;
    roles: string[];
  };
}

export const authMiddleware = (): MiddlewareObj<AuthenticatedEvent, APIGatewayProxyResult> => {
  return {
    before: async (request) => {
      try {
        const token = extractToken(request.event.headers);
        
        if (!token) {
          throw new Error('No token provided');
        }
        
        const jwtService = new JwtService();
        const payload = await jwtService.verify(token);
        
        // Add user to event context
        request.event.user = {
          id: payload.sub,
          email: payload.email,
          roles: payload.roles || []
        };
      } catch (error) {
        return createResponse(401, { message: 'Unauthorized' });
      }
    }
  };
};

function extractToken(headers: { [name: string]: string | undefined }): string | null {
  const authHeader = headers.Authorization || headers.authorization;
  
  if (!authHeader) {
    return null;
  }
  
  const parts = authHeader.split(' ');
  if (parts.length !== 2 || parts[0] !== 'Bearer') {
    return null;
  }
  
  return parts[1];
}
```

### Validation Middleware
```typescript
// src/shared/middleware/validation.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { MiddlewareObj } from '@middy/core';
import Joi from 'joi';
import { createResponse } from '@/shared/utils/response';

interface ValidationOptions {
  body?: Joi.Schema;
  pathParameters?: Joi.Schema;
  queryStringParameters?: Joi.Schema;
}

export const validationMiddleware = (
  schemas: ValidationOptions
): MiddlewareObj<APIGatewayProxyEvent, APIGatewayProxyResult> => {
  return {
    before: async (request) => {
      try {
        const { event } = request;
        
        // Validate body
        if (schemas.body && event.body) {
          const body = typeof event.body === 'string' ? JSON.parse(event.body) : event.body;
          const { error } = schemas.body.validate(body);
          if (error) {
            return createResponse(400, {
              message: 'Validation error',
              details: error.details.map(d => d.message)
            });
          }
        }
        
        // Validate path parameters
        if (schemas.pathParameters && event.pathParameters) {
          const { error } = schemas.pathParameters.validate(event.pathParameters);
          if (error) {
            return createResponse(400, {
              message: 'Invalid path parameters',
              details: error.details.map(d => d.message)
            });
          }
        }
        
        // Validate query parameters
        if (schemas.queryStringParameters && event.queryStringParameters) {
          const { error } = schemas.queryStringParameters.validate(event.queryStringParameters);
          if (error) {
            return createResponse(400, {
              message: 'Invalid query parameters',
              details: error.details.map(d => d.message)
            });
          }
        }
      } catch (error) {
        return createResponse(400, { message: 'Invalid request format' });
      }
    }
  };
};
```

### Logging Middleware
```typescript
// src/shared/middleware/logging.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { MiddlewareObj } from '@middy/core';
import { logger } from '@/shared/utils/logger';

export const loggingMiddleware = (): MiddlewareObj<APIGatewayProxyEvent, APIGatewayProxyResult> => {
  return {
    before: async (request) => {
      const { event } = request;
      
      logger.info('Request started', {
        requestId: event.requestContext.requestId,
        httpMethod: event.httpMethod,
        path: event.path,
        userAgent: event.headers['User-Agent'],
        sourceIp: event.requestContext.identity.sourceIp
      });
    },
    
    after: async (request) => {
      const { event, response } = request;
      
      logger.info('Request completed', {
        requestId: event.requestContext.requestId,
        statusCode: response?.statusCode,
        duration: Date.now() - request.internal.startTime
      });
    },
    
    onError: async (request) => {
      const { event, error } = request;
      
      logger.error('Request failed', {
        requestId: event.requestContext.requestId,
        error: error.message,
        stack: error.stack
      });
    }
  };
};
```

## Service Layer Patterns

### Base Service Class
```typescript
// src/shared/services/base.service.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { logger } from '@/shared/utils/logger';

export abstract class BaseService {
  protected readonly dynamoClient: DynamoDBDocumentClient;
  protected readonly tableName: string;
  
  constructor(tableName?: string) {
    const client = new DynamoDBClient({
      region: process.env.AWS_REGION || 'us-west-2'
    });
    
    this.dynamoClient = DynamoDBDocumentClient.from(client);
    this.tableName = tableName || process.env.TABLE_NAME || '';
  }
  
  protected logOperation(operation: string, params: any): void {
    logger.debug(`DynamoDB ${operation}`, {
      tableName: this.tableName,
      params: this.sanitizeParams(params)
    });
  }
  
  private sanitizeParams(params: any): any {
    // Remove sensitive data from logs
    const sanitized = { ...params };
    delete sanitized.password;
    delete sanitized.token;
    return sanitized;
  }
}
```

### User Service Implementation
```typescript
// src/shared/services/user.service.ts
import { GetCommand, PutCommand, UpdateCommand, DeleteCommand, QueryCommand } from '@aws-sdk/lib-dynamodb';
import { BaseService } from './base.service';
import { User, CreateUserRequest, UpdateUserRequest } from '@/shared/types/user.types';
import { ConflictError, NotFoundError } from '@/shared/utils/errors';
import { hashPassword } from '@/shared/utils/crypto';
import { v4 as uuidv4 } from 'uuid';

export class UserService extends BaseService {
  constructor() {
    super(process.env.USERS_TABLE_NAME);
  }
  
  async create(userData: CreateUserRequest): Promise<User> {
    // Check if user already exists
    const existingUser = await this.getByEmail(userData.email);
    if (existingUser) {
      throw new ConflictError('User with this email already exists');
    }
    
    const user: User = {
      id: uuidv4(),
      email: userData.email,
      name: userData.name,
      passwordHash: await hashPassword(userData.password),
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      isActive: true
    };
    
    const params = {
      TableName: this.tableName,
      Item: user,
      ConditionExpression: 'attribute_not_exists(id)'
    };
    
    this.logOperation('PutItem', params);
    
    await this.dynamoClient.send(new PutCommand(params));
    
    // Remove password hash from response
    const { passwordHash, ...userResponse } = user;
    return userResponse as User;
  }
  
  async getById(id: string): Promise<User | null> {
    const params = {
      TableName: this.tableName,
      Key: { id }
    };
    
    this.logOperation('GetItem', params);
    
    const result = await this.dynamoClient.send(new GetCommand(params));
    
    if (!result.Item) {
      return null;
    }
    
    const { passwordHash, ...user } = result.Item;
    return user as User;
  }
  
  async getByEmail(email: string): Promise<User | null> {
    const params = {
      TableName: this.tableName,
      IndexName: 'EmailIndex',
      KeyConditionExpression: 'email = :email',
      ExpressionAttributeValues: {
        ':email': email
      }
    };
    
    this.logOperation('Query', params);
    
    const result = await this.dynamoClient.send(new QueryCommand(params));
    
    if (!result.Items || result.Items.length === 0) {
      return null;
    }
    
    const { passwordHash, ...user } = result.Items[0];
    return user as User;
  }
  
  async update(id: string, updates: UpdateUserRequest): Promise<User> {
    const updateExpression: string[] = [];
    const expressionAttributeNames: Record<string, string> = {};
    const expressionAttributeValues: Record<string, any> = {};
    
    Object.entries(updates).forEach(([key, value]) => {
      if (value !== undefined) {
        updateExpression.push(`#${key} = :${key}`);
        expressionAttributeNames[`#${key}`] = key;
        expressionAttributeValues[`:${key}`] = value;
      }
    });
    
    // Always update the updatedAt timestamp
    updateExpression.push('#updatedAt = :updatedAt');
    expressionAttributeNames['#updatedAt'] = 'updatedAt';
    expressionAttributeValues[':updatedAt'] = new Date().toISOString();
    
    const params = {
      TableName: this.tableName,
      Key: { id },
      UpdateExpression: `SET ${updateExpression.join(', ')}`,
      ExpressionAttributeNames: expressionAttributeNames,
      ExpressionAttributeValues: expressionAttributeValues,
      ConditionExpression: 'attribute_exists(id)',
      ReturnValues: 'ALL_NEW' as const
    };
    
    this.logOperation('UpdateItem', params);
    
    try {
      const result = await this.dynamoClient.send(new UpdateCommand(params));
      const { passwordHash, ...user } = result.Attributes!;
      return user as User;
    } catch (error) {
      if (error.name === 'ConditionalCheckFailedException') {
        throw new NotFoundError('User not found');
      }
      throw error;
    }
  }
  
  async delete(id: string): Promise<void> {
    const params = {
      TableName: this.tableName,
      Key: { id },
      ConditionExpression: 'attribute_exists(id)'
    };
    
    this.logOperation('DeleteItem', params);
    
    try {
      await this.dynamoClient.send(new DeleteCommand(params));
    } catch (error) {
      if (error.name === 'ConditionalCheckFailedException') {
        throw new NotFoundError('User not found');
      }
      throw error;
    }
  }
  
  async cleanupInactiveUsers(cutoffDate: Date): Promise<number> {
    // Implementation for cleanup job
    // This would typically involve scanning and batch deleting
    // For brevity, returning a mock count
    return 0;
  }
}
```

## Error Handling Patterns

### Custom Error Classes
```typescript
// src/shared/utils/errors.ts
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  
  constructor(message: string, statusCode: number, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    
    Error.captureStackTrace(this, this.constructor);
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400);
  }
}

export class NotFoundError extends AppError {
  constructor(message: string) {
    super(message, 404);
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string) {
    super(message, 401);
  }
}
```

## Best Practices

### Function Design
- **Single Purpose**: Each function should have one clear responsibility
- **Stateless**: Functions should not maintain state between invocations
- **Idempotent**: Functions should produce the same result when called multiple times
- **Fast Startup**: Minimize cold start time with efficient imports and initialization
- **Error Handling**: Implement comprehensive error handling and logging

### Performance Optimization
- **Connection Reuse**: Initialize clients outside the handler function
- **Bundle Optimization**: Use webpack to minimize bundle size
- **Memory Sizing**: Right-size memory allocation based on function requirements
- **Provisioned Concurrency**: Use for latency-critical functions

### Security Considerations
- **Input Validation**: Always validate and sanitize inputs
- **Least Privilege**: Grant minimal required permissions
- **Secrets Management**: Use AWS Secrets Manager or Parameter Store
- **Environment Variables**: Never store secrets in environment variables
- **CORS Configuration**: Configure CORS appropriately for your use case